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
The Django ORM is an extremely powerful, yet dangerous tool of the Django framework, primarily because of its implicit query execution. The ORM loads records automatically from the DB whenever a non-loaded field is being accessed, providing for both a great developer experience and a footgun, or in this case, a minefield of performance issues.

For example, let's say we have a `Person` model that contains a reference to another model `Country` as `country_of_residence`. If we execute the following code, Django will perform 2 database queries
```
person = Person.objects.get(pk=id)
person.country_of_residence
```
One (explicit) query is executed to fetch the `Person` object, and one (implicit) query when we access `country_of_residence`. This behavior is transparent to the developer, we cannot tell just by reading the code when a DB query is being executed.

## N+1 queries Problem

There is no formal definition for the "N+1 queries problem". The most common definition I found is "an operation where N+1 queries are executed when accessing N records". I believe this definition is incomplete and misleading because it conveys a sense of linear growth. It could be better defined as: *An operation where the resulting number of SQL queries is Ω(N), where N is the number of records (rows) being accessed*.

Additionally, one can also usually find the following code as an example of the N+1 problem
```py
people = Person.objects.all()

for person in people:
    print(f"{person.name} lives in {person.country_of_residence}")
```
Which is the equivalent of me explaining how not to introduce division by zero error with the following example
```py
def divide_by_zero(number: int):
    return number / 0
```
Exemplifying the N+1 queries problem by showing that kind of trivial example makes it even more dangerous. It leads us to believe that since we did not write any loop where we access nested attributes, there is no way we could have introduced the N+1 queries problem in our codebase. The fact is, we can introduce the N+1 problem without writing a single loop, and unfortunately Django is extremely susceptible to this problem due to the aforementioned implicit query execution.

In the following example we are going to accidentally introduce an N+1 queries problem, identify it, fix it, and learn how to prevent it in the future. 

## Example
We are working on the new [Steam](https://en.wikipedia.org/wiki/Steam_(service)) and we have the task to implement the `players` endpoint to retrieve the list of players, which video games they've played and for how long. The JSON response must have the following format
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
Let's go over the code without much explanation, if you don't understand something please refer to the (awesome) Django and DRF docs.

We have a `Player` model, `Game` model, and the junction table `PlayTime` each with their corresponding serializers.

We could have added a `ManyToMany` with a `through` for `Player <-> Game`, but it is not needed for this example.

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

Note that the serializers are implemented in the direction of `Player -> PlayTime -> Game`. If we had to implement `Game -> PlayTime -> Player` to list the players with most hours ~~wasted~~ spent for each game we'd have to implement different serializers.
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
The following test creates 2 players with 3 (different) games played each, verifies that both players are present in the response and also verifies the games they've played. For brevity, we are not verifying `hours_played` or any ordering in either the `Player` list or the games played list for each player.

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

The test is using `pytest` with `pytest-django` and `pytest-factoryboy`. I find `pytest` much better `unittest` in every aspect. Also, registering factories as fixtures with `pytest-factoryboy` helps identifying complex test setups. If we need more than 3 factories in a single test then we should split the test into multiple tests, or create a new fixture that performs all the setup.

We run `pytest` and the test passes, great. We can praise ourselves (and Django) for being so productive. Or maybe not.

#### Number of queries
Let's add one more test to verify how many DB queries the new endpoint is performing. Let's create 20 players with 20 games each *(1)*, since it seems like a reasonable number for a future paginated response. Thanks to the `django_assert_max_num_queries` fixture from `pytest-django` we can easily verify how many queries we've made. We'll take a guess for now and set a max of 5 queries *(1)*, sounds like a reasonable number right?

*(1) I am an engineer, I am legally allowed (and sometimes encouraged) to define random numbers heuristically*
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

We run the test and we get
```
FAILED test/games/test_views.py::TestPlayerView::test_player_list_num_queries - Failed: Expected to perform 5 queries or less but 421 were done (add -v option to show queries)
```
That's 2 orders of magnitude higher than our expected value! But our code is so simple and we did not write any loops. What happened? How is this possible?

If we follow pytest advice and run again as `pytest -v` we get see the following
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
We are performing:
- 1 query to get the `Player` records. 
- 20 queries (1 per `Player`) to fetch the related `PlayTime` records.
- 400 queries (20 per `PlayTime`) to fetch the `Game` records.

Looks like we have an N+1 problem, well actually an N^2 problem. Now you see why I said the definition of "N extra queries" is misleading.

### Solution

#### prefetch_related for one to many
Let's start with the easiest part. We can fetch all the `PlayTime` records in a single query with the `prefetch_related` method. `prefetch_related` works by performing a single query for all the related objects separately and then joins the result in Python.

Let's add a `prefetch_related("playtimes")` to the `queryset` attribute of our view
```py
# games/views.py
class PlayerView(generics.ListAPIView):
    queryset = Player.objects.prefetch_related("playtimes").all() # NEW
    serializer_class = PlayerSerializer
```
We only need 1 query now to fetch all the `PlayTime` records instead of N. If we run the test again we should see 402 queries.
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
While we could do the same for the `Game` records, i.e. add `prefetch_related("playtimes__game")`, verify that we perform only 3 queries and pat ourselves on the back, it would not be efficient. This would be the resulting query
```
SELECT "games_game"."id", "games_game"."title" FROM "games_game" WHERE "games_game"."id" IN (1, 2, ..., 399, 400)
```
Yes, it's a single query, but it can be a potentially slow query. We can do better.

#### Prefetch and select_related
Django comes with both `prefetch_related` and `select_related`. As per the documentation: *select_related is limited to single-valued relationships - foreign key and one-to-one.* We could not use `select_related` before in `Player->PlayTime` because it is a one-to-many.

Unlike `Player->PlayTime` which was a one-to-many, `Game` is a foreign key of `PlayTime`. That means we should be able to add a `select_related("game")` somewhere and fetch both `PlayTime` and `Game` in a single query.

If we read through Django's documentation, we'll come across the `Prefetch` object. It has the same behavior as `prefetch_related`, but it gives us more control over the prefetch action. We can pass a `queryset` parameter to `Prefetch`, that is where the `select_related("game")` should go, let's update the views file
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
If we run the test again we'll see only 2 queries, which is the lowest possible for this endpoint.

Our job is done :)

### Preventing N+1 and best practices

#### General guidelines
"N+1 problem" is a very deceiving name. First of all "N+1" assumes linear growth, and as we saw in the example, we can get quadratic growth or higher. Second, it is called "problem" instead of "bug". I believe it should be treated like any other bug, one should assume it's present until proven otherwise.

Here are some guidelines on how to test for the N+1 problem in Django codebases
#### Test your endpoints
1. List-type endpoints. Add a test with a significant number of results that only checks the number of queries. Verify the correct behavior of the endpoint in another test. This is what we did in the example above.
2. Detail-type endpoints. Create a record for the worst case scenario, where all of the (nullable) relationships have values. Verify the correct behavior of the endpoint in another test.
3. If you think "this code is too simple, there is no way I added an N+1 query problem" then prove it. Another developer might add an extra field to an endpoint/serializer/model, accidentally introducing an N+1 problem. The only way to prevent it is by writing tests.

#### Avoid properties that access relationships
While we could not tell in `person.country_of_residence` that we were performing an extra query, we could at least suspect it from its type, since it could not be represented as a DB column. Using nested properties in Django models obscures the queries even further, especially when they access related objects.

Imagine we added the following to `PlayTime`
```py
@property
def game_title(self) -> str:
    return self.game.title
```
This would be misleading and dangerous. A developer working with the `PlayTime` object might assume that `game_title` is a field of `PlayTime` since it is a string, therefore accessing `play_time.game_title` couldn't possibly trigger a DB access. However, if we replaced the code with `play_time.game.title` it becomes obvious that we must either preload `game` or perform an additional query.

### Custom querysets
The code we wrote is fine, it works and it is efficient, but we can improve it in terms of design. We are exposing too many DB details in the views file, and our query is not reusable. If we had to write a new view that also fetches `Player->PlayTime->Game` we would have to copy & paste the `queryset` attribute everywhere.

Django allows us to define custom `QuerySet` which we can use to abstract and reuse common patterns for accessing/filtering data. Let's add a `managers.py` to our app with custom querysets and update our models
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
And finally let's update our views file

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