+++
title = "You can do better than dicts for structured data"
date = "2023-12-27T21:54:51-03:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = [ ]
+++

Despite its extensive standard library, Python doesn't have **record** or **struct** like data structures. **dict** became the de-facto data type for both structured and structured data. This lead to the use and abuse of "ad-hoc" structures that you can only understand by digging through the codebase. You can often find legacy code like this
```py
def parse(data):
    ...
```
or its more helpful type-annotated version
```py
def parse(data: dict):
    ...
```

What does data contain? what are its allowed keys? which values can the keys have? If you are lucky and the person responsible for that monstrosity still works on the project you can ask them, if not then get ready for some software archeology. In any case, you can, and you **should** do better. Do it for the next poor soul who has to modify that code.

There are better options to improve this code. As an example, you can pretend `parse(data)` takes a dict representing a writer with their `name`, `year_of_birth` and the `books` they have published. A valid input could be
```py
{
    "name": "Paulo Coelho",
    "year_of_birth": 1947,
    "books": [
        {"title": "The Alchemist", "year_of_publication": 1988}
    ],
}
```
You could have used a more realistic `date` key for `year_of_birth`, which could be a timestamp, a `datetime` object, a `str`, etc.

## Dataclasses
Introduced in Python 3.7 standard library, [**Dataclasses**](https://docs.python.org/3/library/dataclasses.html) generate a couple of dunder methods automatically based on type annotations. Additionally, they come with handy methods and properties that a regular class doesn't have, such as `frozen`, `asdict()`, etc.

Using dataclasses, you can write the example as
```py
from dataclasses import dataclass

@dataclass
class Book:
    title: str
    year_of_publication: int


@dataclass
class Author:
    name: str
    year_of_birth: int
    books: list[Book]

def parse(data: Author):
    ...
```

You don't have to navigate through the code to find out what `data` contains. However, you now have to modify the code to construct `Author` from `data`. Also, remember that type hints are just **hints**, and not enforced by the interpreter at runtime. The following code runs successfully
```py
book = Book(title="The Alchemist", year_of_publication="More than 30 years ago")
```
You have to rely on a type checker, such as [mypy](https://mypy-lang.org/), to catch the error
```
error: Argument "year_of_publication" to "Book" has incompatible type "str"; expected "int"  [arg-type]
```

You have to be careful when defining mutable default types. Imagine that an `Author` should have an empty list of `books` as default value. The correct way of doing it would be
```py
from dataclasses import dataclass

@dataclass
class Author:
    name: str
    year_of_birth: int
    books: list[Book] = field(default_factory=list) # or field(default_factory=lambda: [])
``` 

The following code would create a single shared list of books for every `Author` instance
```py
books: list[Book] = []
```

Note also that dataclasses don't parse nested fields by default. The following code
```py
data = {
    "name": "Paulo Coelho",
    "year_of_birth": 1947,
    "books": [{"title": "The Alchemist", "year_of_publication": 1988}],
}

Author(**data)
```

produces
```py
Author(name='Paulo Coelho', year_of_birth=1947, books=[{'title': 'The Alchemist', 'year_of_publication': 1988}])
```
You can solve this by implementing the nested parsing inside the [`__post_init__`](https://docs.python.org/3/library/dataclasses.html#dataclasses.__post_init__) magic method of the parent class.
```py
@dataclass
class Author:
    name: str
    year_of_birth: int
    books: list[Book]

    def __post_init__(self):
        self.books = [Book(**b) for b in self.books]
```
and you get the expected result
```py
Author(name='Paulo Coelho', year_of_birth=1947, books=[Book(title='The Alchemist', year_of_publication=1988)])
```

## TypedDict
Introduced in Python 3.8, [**TypedDict**](https://docs.python.org/3/library/typing.html#typing.TypedDict) are type annotations for dicts. **TypedDict** are still dicts under the hood. Similar to Dataclasses, you have to use a type checker to catch errors. **TypedDict** are more straightforward to implement than Dataclasses since you don't have to modify the existing code at all, only define the TypedDict and update the function header. The example would be as follows
```py
from typing import TypedDict

class Book(TypedDict):
    title: str
    year_of_publication: int


class Author(TypedDict):
    name: str
    year_of_birth: int
    books: list[Book]


def parse(data: Author):
    ...

```
Note that unlike Dataclasses, you can't define default values. There isn't much to talk about regarding TypedDict.

## Pydantic
The only third party option from this list. [Pydantic](https://docs.pydantic.dev/latest/) is a data validation module for Python. While it might look similar to Dataclasses for simple cases, Pydantic classes can get complex and the library has its learning curve. This post shows only a few features of the library, but you should to go through the documentation, especially the examples, to get a grasp of how much it can do. For the current example, you can start with the same specification from the dataclass section.
```py
from pydantic import BaseModel, Field


class Book(BaseModel):
    title: str
    year_of_publication: int


class Author(BaseModel):
    name: str
    year_of_birth: int
    books: list[Book] = Field(default_factory=list)


def parse(data: Book):
    ...
```
If you now attempt to create an instance with invalid data, you get a runtime error
```py
book = Book(title="The Alchemist", year_of_publication="More than 30 years ago")
```
```
pydantic_core._pydantic_core.ValidationError: 1 validation error for Book
year_of_publication
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='More than 30 years ago', input_type=str]
```
Pydantic is actually performing **data validation** at runtime. In fact, you can write your own custom validators. For example, you can validate that the `Author` name isn't empty, and that the `Book` publish year is after the `Author`'s birth date.
```py
from pydantic import BaseModel, Field, model_validator


class Book(BaseModel):
    title: str
    year_of_publication: int


class Author(BaseModel):
    name: str = Field(min_length=1)
    year_of_birth: int
    books: list[Book] = Field(default_factory=list)

    @model_validator(mode="after")
    def validate_consistent_year(self):
        assert all(
            book.year_of_publication >= self.year_of_birth for book in self.books
        ), "Author cannot contain books published before their birth"
```
The following code runs successfully
```py
author = Author(name="Paulo Coelho", year_of_birth=1947, books=[{"title": "The Alchemist", "year_of_publication": 1988}])
```
However, if you change `year_of_publication` to 1945, it fails as expected
```
pydantic_core._pydantic_core.ValidationError: 1 validation error for Author
  Assertion failed, Author cannot contain books published before their birth [type=assertion_error, input_value={'name': 'Paulo Coelho', ...of_publication': 1945}]}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.5/v/assertion_error
```

One final caveat: by default, Pydantic attempts to cast the input to the expected type. The following code succeeds because Pydantic casts the string "1988" to the integer `1988`.
```py
Book(**{"title": "The Alchemist", "year_of_publication": "1988"})
```

Pydantic provides better safety, features and runtime guarantees compared to plain Dataclasses, and it's easy to extend and customizable, which also means you can get overwhelmed.

As of writing this blog, Pydantic has recently gone through a major version update from version 1 to 2, with multiple breaking changes. You have to rely mostly on the official docs for learning the library, since most of the existing tutorials, blog articles, etc. are for the 1.X versions.

# Conclusions
This post explored some of the alternatives to plain untyped dicts for structured data, each with their pros and cons. The suggestions of when you should use each one are:
1. **Pydantic** whenever possible.
2. **Dataclasses** when you can refactor the code but you can't use Pydantic.
3. **TypedDict** as last resort when you can't modify existing code logic.

# Extra

## Named tuples
If you are fine representing structured data as tuples instead of dicts then you can use [NamedTuple](https://docs.python.org/3/library/collections.html#collections.namedtuple). It adds type annotations and you can also access the elements by name.
```py
from typing import NamedTuple


class Book(NamedTuple):
    title: str
    year_of_publication: int


book = Book(title="The Alchemist", year_of_publication=1988)
(title, year) = book
assert book[0] == title
assert book.title == title
```

## Distinct types
Another cool pattern for validating data is to define a new type for the same data as an existing type but, unlike type aliases, the type checker treats them as different incompatible types. This behavior is useful when certain functions should only accept values returned from other functions that perform some kind of data validation. You can achieve this behavior with [NewType](https://docs.python.org/3/library/typing.html#newtype).
For example, if you have a `store` function for the data, it should only accept validated author dicts.
```py
from typing import NewType

ValidatedAuthor = NewType("ValidatedAuthor", dict)

def validate(data: dict) -> ValidatedAuthor:
    year_of_birth = data["year_of_birth"]
    books_year_of_publication = (b["year_of_publication"] for b in data.get("books", []))

    if not all(book_year >= year_of_birth for book_year in books_year_of_publication):
        raise ValueError("Author cannot contain books published before their birth")

    return ValidatedAuthor(data)


def store(data: ValidatedAuthor):
    ...

data = {
    "name": "Paulo Coelho",
    "year_of_birth": 1947,
    "books": [{"title": "The Alchemist", "year_of_publication": 1988}],
}

store(validate(data))
store(data)
```

You can verify that the underlying data hasn't changed and that the returned value is still a dict. However, if you run mypy again you get the following error
```
error: Argument 1 to "store" has incompatible type "dict[str, object]"; expected "ValidatedAuthor"  [arg-type]
```
While this pattern allows you to perform data validation using built-in data types, it's completely useless in Python because of mutability. The following code validates the data, then modifies it so that it becomes invalid, and then passes it to the function. The type checker doesn't report any errors.
```py
validated_data = validate(data)
validated_data["books"][0]["year"] = validated_data["year_of_birth"] - 1
store(validated_data)
```