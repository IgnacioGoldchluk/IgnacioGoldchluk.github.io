+++
title = "Pytest Tips and Tricks 1: Executing specific fixtures"
date = "2024-01-24T20:44:14-03:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

## Introduction
Every project should contain tests, lots of tests, thousands of tests. Often it contains an entire codebase, parallel to the actual production codebase, of the same size or bigger, just for testing, full of factories and helper functions. Sometimes tests begin to fail because assumptions you made in fixtures are factories are no longer valid, and you must also update the helper code.

Running the entire test suite might take a couple of minutes. You could reduce the time by skipping certain folders or files, but it requires knowledge of the entire test suite. It would be better if, when you are refactoring a pytest fixture, you could tell pytest to only run tests that depend on that specific fixture and save some time.

## Implementation
pytest is a powerful and highly customizable framework, and rarely utilized to its full potential. You can take advantage of pytest [hooks](https://docs.pytest.org/en/7.1.x/reference/reference.html#hooks), [objects](https://docs.pytest.org/en/7.1.x/reference/reference.html#objects) and an undocumented behavior to execute only the tests that depend on a specific fixture. Here is what you have to do:
1. Add an option to only run tests that depend on a given fixture. 
2. Keep only the tests that contain the fixture you want.

### 1. Adding a custom option
In order to add the custom option, the code relies on the [`pytest_addoption`](https://docs.pytest.org/en/7.1.x/reference/reference.html#pytest.hookspec.pytest_addoption) hook. The hook takes a `pytest.Parser` object and modifies it in-place. You must add the option via the `addoption` method, which has the same API as the `add_argument` method in the stdlib `argparse` module.

Add the following code to your `conftest.py`:

```py
def pytest_addoption(parser: pytest.Parser):
    parser.addoption("--only-for-fixture", dest="only-for-fixture", required=False)
```

### 2. Selecting tests
To modify the list of tests that to execute, the code relies on the [`pytest_collection_modifyitems`](https://docs.pytest.org/en/7.1.x/reference/reference.html#pytest.hookspec.pytest_collection_modifyitems) hook. The hook takes 3 arguments:
* `session: pytest.Session`, which you can ignore for this case.
* `config: pytest.Config`, which you need to fetch the command line option you just created.
* `items: list[pytest.Item]`, the list of items, which in this case you can think of them as tests, that you filter to only keep the tests of interest.

Add the following code to your `conftest.py`
```py
def pytest_collection_modifyitems(session: pytest.Session, config: pytest.Config, items: list[pytest.Item]):
    fixture = config.getoption("only-for-fixture")
    if fixture:
        selected = [item for item in items if fixture in item.fixturenames]
        items[:] = selected
```
You obtain the option value via `config.getoption(name)` method, where `name` is the `dest` parameter from `addoption`. In case there is a value, it creates a new list with the tests that contain the specified fixture in its fixtures, and then reassigns the new value to `items`.

### Cautions
1. `Item.fixturenames` attribute **isn't documented**. You can access the list of fixtures in pytest 7, but future versions may remove this feature.
2. You might want to write `items = selected`, or to iterate over each item and pop them from `items`. The first case doesn't work because the original `items` argument remains unchanged. The second case doesn't work because the iterable's size changes while iterating.

## Example
Given the following fixtures
```py
@pytest.fixture()
def random_number_between_0_and_100():
    return random.randint(0, 100)


@pytest.fixture()
def random_number_between_100_and_200(random_number_between_0_and_100):
    return 100 + random_number_between_0_and_100
```

and the following tests in `tests/test_examples.py`: a test that doesn't depend on `random_number_between_0_and_100`, a test that depends on it explicitly, and a test that depends on the fixture implicitly
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

Running `pytest` executes every test:
```
collected 3 items

tests/test_examples.py::test_does_not_use_fixture PASSED                                               [ 33%]
tests/test_examples.py::test_uses_fixture_explicitly PASSED                                            [ 66%]
tests/test_examples.py::test_uses_fixture_implicitly PASSED                                            [100%]

============================================= 3 passed in 0.01s ==============================================
```

Running `pytest --only-for-fixture random_number_between_0_and_100` executes only the tests that depend on the fixture directly and indirectly:
```
collected 3 items

tests/test_examples.py::test_uses_fixture_explicitly PASSED                                            [ 50%]
tests/test_examples.py::test_uses_fixture_implicitly PASSED                                            [100%]

============================================= 2 passed in 0.01s ==============================================
```

And running `pytest --only-for-fixture invalid_fixture` doesn't execute any tests:
```
collected 3 items

=========================================== no tests ran in 0.00s ============================================
```

## Conclusion
You were able to customize pytest to save time by adding the option to only execute tests that depend on a specific fixture. Quite useful when modifying or refactoring fixtures.

Keep in mind that, while useful, this was still a workaround. Ideally your test suite should take little time to run, and you should make extensive use of pytest [markers](https://docs.pytest.org/en/7.1.x/example/markers.html), which already come with the `-m` option for selecting/ignoring tests.