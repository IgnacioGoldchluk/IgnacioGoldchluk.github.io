+++
title = "Pytest Tips and Tricks 1: Executing specific fixtures"
date = "2024-01-24T20:44:14-03:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

## Skippable introduction
I write tests, lots of tests, thousands of tests. I often build an entire codebase, parallel to the actual production codebase, just for testing, full of factories and helper functions. I could go on explaining why, but that's not the point of this post.

The point is, code evolves, and business logic changes. Best case scenario, tests start failing because the test code is no longer valid. Worst case scenario, tests still pass, and you should question what on earth were you testing this whole time then.

Tests can also fail because assumptions made in helper code are no longer valid, and we must update the helper code. In the specific case of pytest, this helper code is often written as fixtures.

Running your entire test suite might take a couple of minutes. You could reduce the time by skipping certain folders or files, but it requires knowledge of the entire test suite. Wouldn't it be great if, when we are refactoring a pytest fixture, we could tell pytest to only run tests that depend on that specific fixture and save us a ton of time?

## Implementation
pytest is a powerful and highly customizable framework, and rarely utilized to its full potential. We can take advantage of pytest [hooks](https://docs.pytest.org/en/7.1.x/reference/reference.html#hooks), [objects](https://docs.pytest.org/en/7.1.x/reference/reference.html#objects) and (just) 1 undocumented behavior to execute only the tests that depend on a specific fixture. To do so we have to:
1. Add an option to only run tests that depend on a given fixture. 
2. Keep only the tests that contain the fixture we want.

### 1. Adding a custom option
In order to add our custom option, we rely on the [`pytest_addoption`](https://docs.pytest.org/en/7.1.x/reference/reference.html#pytest.hookspec.pytest_addoption) hook. The hook receives a `pytest.Parser` object and modifies it in-place. The option is added via the `addoption` method which has the same API as the `add_argument` method in the stdlib `argparse` module.

Add the following code to your `conftest.py`:

```py
def pytest_addoption(parser: pytest.Parser):
    parser.addoption("--only-for-fixture", dest="only-for-fixture", required=False)
```

### 2. Selecting tests
To modify the list of tests that will be executed, we rely on the [`pytest_collection_modifyitems`](https://docs.pytest.org/en/7.1.x/reference/reference.html#pytest.hookspec.pytest_collection_modifyitems) hook. It receives 3 arguments:
* `session: pytest.Session`, which we'll ignore for this case.
* `config: pytest.Config`, which we'll use to fetch the command line option we just created.
* `items: list[pytest.Item]`, the list of items (you can think of them as tests for this case) that we'll modify to only keep the tests we care about.

Add the following code to your `conftest.py`
```py
def pytest_collection_modifyitems(session: pytest.Session, config: pytest.Config, items: list[pytest.Item]):
    fixture = config.getoption("only-for-fixture")
    if fixture:
        selected = [item for item in items if fixture in item.fixturenames]
        items[:] = selected
```
The option value is obtained via `config.getoption(name)` method, where `name` is the `dest` parameter from `addoption`. In case there is a value, we create a new list with the items (tests) that contain the specified fixture in its fixtures, and then reassign the new value to items.

### Cautions
1. `Item.fixturenames` attribute **is not a documented behavior**. The list of fixtures is accessible as of writing this post (pytest 7.X) but may be removed in the future.
2. One might be tempted to write `items = selected`, or to iterate over each item and pop them from `items`. The first case does not work because the original `items` argument will remain unchanged. The second case does not work because the iterable is size is being modified while iterating.

## Example
Let's say we have the following fixtures
```py
@pytest.fixture()
def random_number_between_0_and_100():
    return random.randint(0, 100)


@pytest.fixture()
def random_number_between_100_and_200(random_number_between_0_and_100):
    return 100 + random_number_between_0_and_100
```

And the following tests in `tests/test_examples.py`: A test that does not depend on `random_number_between_0_and_100`, a test that depends on it explicitly, and a test that depends on the fixture implicitly
```py
def test_does_not_use_fixture():
    a = 1
    b = 2
    assert a + b == 3


def test_uses_fixture_explicitly(random_number_between_0_and_100):
    assert 0 <= random_number_between_0_and_100 <= 100


def test_uses_fixture_implicitly(random_number_between_100_and_200):
    assert 100 <= random_number_between_100_and_200 <= 200
```

Running `pytest`, all tests are executed:
```
collected 3 items

tests/test_examples.py::test_does_not_use_fixture PASSED                                               [ 33%]
tests/test_examples.py::test_uses_fixture_explicitly PASSED                                            [ 66%]
tests/test_examples.py::test_uses_fixture_implicitly PASSED                                            [100%]

============================================= 3 passed in 0.01s ==============================================
```

Running `pytest --only-for-fixture random_number_between_0_and_100`, only the tests that depend on the fixture (implicitly and explicitly) are executed:
```
collected 3 items

tests/test_examples.py::test_uses_fixture_explicitly PASSED                                            [ 50%]
tests/test_examples.py::test_uses_fixture_implicitly PASSED                                            [100%]

============================================= 2 passed in 0.01s ==============================================
```

And `pytest --only-for-fixture invalid_fixture`, no tests are executed:
```
collected 3 items

=========================================== no tests ran in 0.00s ============================================
```

## Conclusion
We were able to customize pytest to save time by adding the option to only execute tests that depend on a specific fixture. Quite useful when modifying or refactoring fixtures.

Keep in mind that, while useful, this was still a workaround. Ideally your test suite should take little time to run, and you should make extensive use of pytest [markers](https://docs.pytest.org/en/7.1.x/example/markers.html), which already come with the `-m` option for selecting/ignoring tests.