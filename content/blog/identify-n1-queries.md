+++
title = "Identifying and fixing N+1 queries problem in Django"
date = "2024-03-24T22:42:54-03:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = [ ]
+++

The Django ORM is an extremely powerful and yet dangerous ORM because of its implicit query execution. The ORM loads related objects from the DB whenever its fields are accessed, providing for both a great developer experience and a footgun, or in this case, a minefield of performance issues.

## N+1 Query Problem

There is not a formal definition of the "N+1 query problem". Some sites define it as "an operation where the ORM must execute N+1 queries for N records", but I believe it could be better defined as: *An ORM operation where the resulting number of SQL queries is at least O(N), where N is the number of records (rows) being accessed*.

## Example
We are working on the new [Steam](https://en.wikipedia.org/wiki/Steam_(service)) and we have the task to implement the `players` endpoint to retrieve the list of players, which videogames they've played and for how many hours each. The JSON response must have the following format
```
[
  {
    "name": "John Doe",
    "playtimes": [
      {
        "game": {"title": "Pac-Man"}
        "hours_played": 123, 
      }
    ]
  }
]
```

### Setup
We are experienced Django developers, we can implement everything we need in a few lines without much explanation.

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

serializers file
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

Adding the view to a new `urls.py` file and including it in the main project `urls.py`.
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
The test creates 2 players with 3 (different) games played each, verifies that both players are present in the response and the games they've played. For brevity, we are not verifying `hours_played` or any ordering in either the `Player` list or the games played list for each player.

#### Functionality
```py
# test/games/test_views.py
@pytest.mark.django_db
class TestPlayerView:
    def test_player_view_lists_players(
        self, api_client, player_factory, play_time_factory
    ):
        p1 = player_factory()
        p1_played = [play_time_factory(player=p1).game.title for _ in range(3)]

        p2 = player_factory()
        p2_played = [play_time_factory(player=p2).game.title for _ in range(3)]

        response = api_client.get("/games/players", format="json").json()
        player_data = next(r for r in response if r["name"] == p1.name)

        playtimes_titles = [p["game"]["title"] for p in player_data["playtimes"]]
        assert all(played in playtimes_titles for played in p1_played)

        player2_data = next(r for r in response if r["name"] == p2.name)
        playtimes_titles = [p["game"]["title"] for p in player2_data["playtimes"]]
        assert all(played in playtimes_titles for played in p2_played)
```

We are using `pytest` with `pytest-django` and `pytest-factoryboy`. I find `pytest` much better `unittest` in every aspect. Registering factories as fixtures with `pytest-factoryboy` helps in identifying complex test setups. If we need more than 3 factories for a test then we should break the test apart into multiple tests, or create a new fixture that performs all the setup.

We run `pytest` and the test passes. Awesome, we can praise ourselves (and Django) for being so productive. Or maybe not.

#### Number of queries
Let's add one more test to verify how many DB queries the new endpoint is performing. This time, let's create 20 players with 20 games each *(1)*, since it seems like a reasonable number for a future paginated response. Thanks to `django_assert_max_num_queries` fixture from `pytest-django` we can easily verify how many queries we've made. We'll take a guess for now and set a max of 5 queries *(1)*, sounds like a reasonable number right?

*(1) I am an enigneer, I am legally allowed (and encouraged) to use random numbers*
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
        for _ in range(20):
            player = player_factory()
            for _ in range(20):
                play_time_factory(player=player)

        with django_assert_max_num_queries(5):
            api_client.get("/games/players", format="json").json()
```

We run the test and we get
```
FAILED test/games/test_views.py::TestPlayerView::test_player_list_num_queries - Failed: Expected to perform 5 queries or less but 421 were done (add -v option to show queries)
```
That's 2 orders of magnitude higher than what we expected! But our code is so simple. What happened? How is this possible? If we follow pytest advice and run again as `pytest -v` we get see the following
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

We got an N+1 problem, or in this case an N^2 problem.

### Solution

#### prefetch_related for one to many
Let's start from the easiest part. We can fetch all the `PlayTime` records in a single query with the `prefetch_related` method. `prefetch_related` works by performing a single query for all the related objects separately and then joins the result in Python.

Let's add a `prefetch_related("playtimes")` to the `queryset` atribute of our view
```py
# games/views.py
class PlayerView(generics.ListAPIView):
    queryset = Player.objects.prefetch_related("playtimes").all() # NEW
    serializer_class = PlayerSerializer
```
Instead of 20 queries to fetch the `PlayTime` records we only need one now. If we run the test again we should see 402 queries.
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
While we could do the same for the `Game` records, that is, add a `prefetch_related("playtimes__game")`, verify that we would perform only 3 queries and pat ourselves on the back, it would not be efficient. This would be the resulting query
```
SELECT "games_game"."id", "games_game"."title" FROM "games_game" WHERE "games_game"."id" IN (1, 2, ..., 399, 400)
```
Yeah, it's a single query, but we can do much better

#### Prefetch and select_related
Django has both `prefetch_related` and `select_related`. As per the documentation: *select_related is limited to single-valued relationships - foreign key and one-to-one.* We cannot use `select_related` in `Player->PlayTime` because it is a one-to-many.

We cannot do `select_related("playtimes")`, but `Game` is a foreign key of `PlayTime`. We should be able to somehow add a `select_related("game")` somewhere and fetch both `PlayTime` and `Game`.

If we read through Django's documentation, we'll come across the `Prefetch` object. It has the same behavior as `prefetch_related`, but it gives us more control over the prefetch action. We can pass a `queryset` parameter to `Prefetch`, that's where the `select_related("game")` should go, let's update the views file
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
If we run the test again we'll see only 2 queries, which is the lowest possible for this endpoint. Our job is done.

### Preventing N+1 and best practices

#### General guidelines
"N+1 problem" is a very deceiving name. First of all "N+1" assumes linear growth, and as we saw in the example, we can get quadratic growth or higher. Second, it is called "problem" instead of "bug". I believe it should be treated like any other bug, one should assume it's present until proven otherwise.

Here are some guidelines on how to test for the N+1 problem in Django codebases:
1. Test your list-type endpoints.
    * The test should have the maximum number of possible results. If your pagination is 10 results, then create 10 records.
    * Verify the correct behavior of the endpoint in another test with fewer results, as it was done in the example.
2. Test your detail-type endpoints if they include one-to-many relationship.
    * Create a record for the worst case scenario, where all of the (nullable) relationships have values.
    * Same as the list-type endpoint, verify the correct behavior in another test, and write a single test to assert the number of queries.
3. Test, test, test. If you think "this code is too simple, there is no way I added an N+1" then prove it. Another developer might add an extra field to an endpoint, accidentally causing an N+1, and that developer could be you. You can prevent it by writing tests.

#### Custom querysets
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

You can find the final version of the code [here](https://github.com/IgnacioGoldchluk/n_plus_one)