+++
title = "Identifying and fixing N+1 queries problem in Django"
date = "2024-03-24T22:42:54-03:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = [ ]
+++

## Introduction
The Django Object Relational Mapping is an extremely powerful, yet dangerous tool of the Django framework, primarily because of its implicit query execution. Django loads records automatically from the DB whenever it needs to access a non-loaded field, which leads to potential performance issues and an unexpected number of queries executed.

For example, imagine you have a `Person` model that contains a reference to another model `Country` as `country_of_residence`. If you execute the following code, Django performs 2 database queries
```
person = Person.objects.get(pk=id)
person.country_of_residence
```
The code executes one query to fetch the `Person` object, and an extra query when it accesses `country_of_residence`. This behavior is transparent to the developer, you can't tell simply by reading the code how many DB queries it executes without knowing the query that loaded the model initially.

## N+1 queries problem
While there is no formal definition for the N+1 queries problem, the most common definition you can often find online is *an operation that executes N+1 additional queries when accessing N records*.
This definition is still incomplete and misleading because it conveys a sense of linear growth. It could be better defined as *an operation where the resulting number of SQL queries is Ω(N), where N is the number of records accessed*.

The following example "accidentally" introduces an N+1 queries problem. You are going to learn how to identify it, fix it and prevent it from happening again in the future.

## Example
You are creating the new [Steam](https://en.wikipedia.org/wiki/Steam_(service)) and your task is to implement the `players` endpoint to retrieve the list of players, which video games they've played and for how long. The JSON response must have the following format
```
[
  {
    "name": "John Doe",
    "playtimes": [
      {
        "game": {"title": "Bubble Bobble"},
        "hours_played": 123
      }
    ]
  }
]
```

### Setup
The example contains a `Player` model, `Game` model, and the junction table `PlayTime` each with their corresponding serializers.

You could have added a `ManyToMany` with a `through` for `Player <-> Game`, but it isn't needed for this example.

If something isn't clear from the files, you can refer to the [Django Rest Framework](https://www.django-rest-framework.org/) documentation.

models file
```py
# games/models.py
from django.db import models


class Game(models.Model):
    title = models.CharField(max_length=250, null=False)


class Player(models.Model):
    name = models.CharField(max_length=250, null=False, unique=True)


class PlayTime(models.Model):
    player = models.ForeignKey(
      Player, related_name="playtimes", on_delete=models.CASCADE
    )
    game = models.ForeignKey(Game, related_name="playtimes", on_delete=models.CASCADE)

    hours_played = models.PositiveIntegerField()
```

serializers file.

Note that the serializers go in the direction of `Player -> PlayTime -> Game`. If you had to implement `Game -> PlayTime -> Player` to list the players with most hours ~~wasted~~ spent for each game, then you would have to implement different serializers.
```py
# games/serializers.py
from rest_framework import serializers

from .models import Game, Player, PlayTime


class GameSerializer(serializers.ModelSerializer):
    class Meta:
        model = Game
        fields = ["title"]


class PlayTimeSerializer(serializers.ModelSerializer):
    game = GameSerializer(read_only=True)

    class Meta:
        model = PlayTime
        fields = ["hours_played", "game"]


class PlayerSerializer(serializers.ModelSerializer):
    playtimes = PlayTimeSerializer(many=True, read_only=True)

    class Meta:
        model = Player
        fields = ["name", "playtimes"]
```

views file
```py
# games/views.py
from rest_framework import generics

from .models import Player
from .serializers import PlayerSerializer


class PlayerView(generics.ListAPIView):
    queryset = Player.objects.all()
    serializer_class = PlayerSerializer
```

The app `urls.py` file, and the root project `urls.py` file.
```py
# games/urls.py
from django.urls import path
from .views import PlayerView

urlpatterns = [
    path("players", PlayerView.as_view(), name="players"),
]

# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("games/", include("games.urls")),
]
```

### Testing
The following test creates 2 players with 3 different games played each, then verifies that both players are present in the response and finally verifies the games they've played. For simplicity, it skips verifying `hours_played` or any ordering in either the `Player` list or the games played list for each player.

#### Functionality
```py
# test/games/test_views.py
@pytest.mark.django_db
class TestPlayerView:
    def test_player_view_lists_players(
        self, api_client, player_factory, play_time_factory
    ):
        [p1, p2] = player_factory.create_batch(2)
        p1_played = [play_time_factory(player=p1).game.title for _ in range(3)]
        p2_played = [play_time_factory(player=p2).game.title for _ in range(3)]

        response = api_client.get("/games/players", format="json").json()
        player_data = next(r for r in response if r["name"] == p1.name)

        playtimes_titles = [p["game"]["title"] for p in player_data["playtimes"]]
        assert all(played in playtimes_titles for played in p1_played)

        player2_data = next(r for r in response if r["name"] == p2.name)
        playtimes_titles = [p["game"]["title"] for p in player2_data["playtimes"]]
        assert all(played in playtimes_titles for played in p2_played)
```

The test is using `pytest` with `pytest-django` and `pytest-factoryboy`. Registering factories as fixtures with `pytest-factoryboy` helps identifying complex test setups. If you need more than 3 factories in a single test then you should probably split the test into multiple tests, or create a new fixture that performs all the setup.

If you run `pytest` the test should pass.

#### Number of queries
You can add a new test to verify how many DB queries the new endpoint is performing. The test creates 20 players with 20 games each. The `django_assert_max_num_queries` fixture from `pytest-django` asserts on the number of queries made in a specific context. You can start with a guess of 5 max queries when writing the test, and change the number later once you know the final value.

```py
# test/games/test_views.py
@pytest.mark.django_db
class TestPlayerView:
    ...

    def test_player_list_num_queries(
        self,
        api_client,
        player_factory,
        play_time_factory,
        django_assert_max_num_queries,
    ):
        players = player_factory.create_batch(20)
        for player in players:
            play_time_factory.create_batch(20, player=player)

        with django_assert_max_num_queries(5):
            api_client.get("/games/players", format="json").json()
```

Running the test results in the following
```
FAILED test/games/test_views.py::TestPlayerView::test_player_list_num_queries - Failed: Expected to perform 5 queries or less but 421 were done (add -v option to show queries)
```
That's 2 orders of magnitude more queries than the expected value. How's this possible?

If you follow pytest advice and run again as `pytest -v` you get the following output
```
SELECT "games_player"."id", "games_player"."name" FROM "games_player"
E
E               SELECT "games_playtime"."id", "games_playtime"."player_id", "games_playtime"."game_id", "games_playtime"."hours_played" FROM "games_playtime" WHERE "games_playtime"."player_id" = 1
E
E               SELECT "games_game"."id", "games_game"."title" FROM "games_game" WHERE "games_game"."id" = 1 LIMIT 21
E
E               SELECT "games_game"."id", "games_game"."title" FROM "games_game" WHERE "games_game"."id" = 2 LIMIT 21
[...]
```
The code is performing:
- 1 query to get the `Player` records. 
- 20 queries to fetch the related `PlayTime` records, 1 per `Player`.
- 400 queries to fetch the `Game` records, 20 per `PlayTime`.

Looks like the code has an N+1 query problem.

### Solution

#### Preloading related objects in a separate query
Starting with the easiest part, you can fetch all the `PlayTime` records in a single query with the `prefetch_related` method. `prefetch_related` works by performing a single query for all the related objects separately and then joins the result in Python.

Adding a `prefetch_related("playtimes")` to the `queryset` attribute of the view
```py
# games/views.py
class PlayerView(generics.ListAPIView):
    queryset = Player.objects.prefetch_related("playtimes").all() # NEW
    serializer_class = PlayerSerializer
```
You only need 1 query to fetch all the `PlayTime` records instead of N. If you run the test again you should see 402 queries.
```
E               Failed: Expected to perform 5 queries or less but 402 were done
E
E               Queries:
E               ========
E
E               SELECT "games_player"."id", "games_player"."name" FROM "games_player"
E               
E               # Here comes the prefetch query
E               SELECT "games_playtime"."id", "games_playtime"."player_id", "games_playtime"."game_id", "games_playtime"."hours_played" FROM "games_playtime" WHERE "games_playtime"."player_id" IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20)
E
E               SELECT "games_game"."id", "games_game"."title" FROM "games_game" WHERE "games_game"."id" = 1 LIMIT 21
E
E               SELECT "games_game"."id", "games_game"."title" FROM "games_game" WHERE "games_game"."id" = 2 LIMIT 21
[...]
```
While you could do the same for the `Game` records by adding `prefetch_related("playtimes__game")` and verifying that it performs only 3 queries, it would be inefficient. This would be the resulting query
```
SELECT "games_game"."id", "games_game"."title" FROM "games_game" WHERE "games_game"."id" IN (1, 2, ..., 399, 400)
```
It's a single query, but it can still be a potentially slow query. You can improve this with some effort.

#### Prefetch object and `select_related`
Django comes with both `prefetch_related` and `select_related`. As per the documentation: *select_related is limited to single-valued relationships - foreign key and one-to-one.* You couldn't use `select_related` before in `Player->PlayTime` because it's a one-to-many.

Unlike `Player->PlayTime` which was a one-to-many, `Game` is a foreign key of `PlayTime`. That means you should be able to add a `select_related("game")` somewhere and fetch both `PlayTime` and `Game` in a single query.

If you read Django's documentation, you can find the `Prefetch` object. It has the same behavior as `prefetch_related`, but it gives you more control over the prefetch action. You can pass a `queryset` parameter to `Prefetch`, which is where `select_related("game")` should go. The `views` file then looks like
```py
# games/views.py
from django.db.models import Prefetch
from rest_framework import generics

from .models import Player, PlayTime
from .serializers import PlayerSerializer


class PlayerView(generics.ListAPIView):
    queryset = Player.objects.prefetch_related(
        Prefetch("playtimes", queryset=PlayTime.objects.select_related("game"))
    )
    serializer_class = PlayerSerializer
```
If you run the test again you see only 2 queries, which is the lowest possible for this endpoint.

### Preventing N+1 and best practices

#### General guidelines
*N+1 problem* is a very deceiving name. First of all "N+1" assumes linear growth, and as you saw in the example, it's possible to get quadratic growth or higher. Second, it's called "problem" instead of bug, you should always treat the potential existence of N+1 as a bug.

Here are some guidelines on how to test for the N+1 problem in Django codebases
#### Test your endpoints

1. List-type endpoints. Add a test with a significant number of results that only checks the number of queries. Verify the correct behavior of the endpoint in another test. This is what you did in the previous example.
2. Detail-type endpoints. Create a record for the worst case scenario, where all of the relationships have values. Verify the correct behavior of the endpoint in another test.
3. If you think "this code is too simple, there is no way it contains an N+1 query problem" then prove it. Another developer might add an extra field to an endpoint/serializer/model, accidentally introducing an N+1 problem. The only way to prevent it's by writing tests.

#### Avoid properties that access relationships
While you can't detect that `person.country_of_residence` performs an extra query, you can suspect it from its type, since it can't be a DB column. Using nested properties in Django models obscures the queries even further, especially when they access related objects.

Imagine you add the following method to `PlayTime` model
```py
@property
def game_title(self) -> str:
    return self.game.title
```
This code is misleading and dangerous. Another developer working with the `PlayTime` object might assume that `game_title` is a field of `PlayTime` in the DB since it's a string, therefore accessing `play_time.game_title` without preloading `Game`. You should be careful with properties that rely on related objects, they're a common source of N+1 queries problem in Django's Object Relational Mapping.

### Custom querysets
The code so far is correct, it works and it's efficient, but you can improve it in terms of design. The program is exposing too many DB details in the `views` file, and the query isn't reusable. If you have to write a new view that also fetches `Player->PlayTime->Game` you need to copy & paste the `queryset` attribute everywhere.

Django lets you define custom `QuerySet` which you can use to abstract and reuse common patterns for accessing/filtering data. You can create a  `managers.py` with custom querysets and use them in the models
```py
# games/managers.py
from django.db import models


class PlayTimesQuerySet(models.QuerySet):
    def with_game(self):
        return self.select_related("game")


class PlayerQuerySet(models.QuerySet):
    def with_playtimes(self):
        from .models import PlayTime # cyclic import

        return self.prefetch_related(
            models.Prefetch("playtimes", queryset=PlayTime.objects.with_game())
        )


# games/models.py
from django.db import models
from .managers import PlayerQuerySet, PlayTimesQuerySet # NEW


class Player(models.Model):
    name = models.CharField(max_length=250, null=False, unique=True)
    objects = PlayerQuerySet.as_manager() # NEW


class PlayTime(models.Model):
    player = models.ForeignKey(
        Player, related_name="playtimes", on_delete=models.CASCADE
    )
    game = models.ForeignKey(Game, related_name="playtimes", on_delete=models.CASCADE)

    hours_played = models.PositiveIntegerField()

    objects = PlayTimesQuerySet.as_manager() # NEW
```

Updating the views file
```py
# games/views.py
from rest_framework import generics

from .models import Player
from .serializers import PlayerSerializer


class PlayerView(generics.ListAPIView):
    queryset = Player.objects.with_playtimes()
    serializer_class = PlayerSerializer
```

You can find the example code [here](https://github.com/IgnacioGoldchluk/n_plus_one)