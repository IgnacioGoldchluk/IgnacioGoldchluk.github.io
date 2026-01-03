+++
title = "We can do better than dicts for structured data!"
date = "2023-12-27T21:54:51-03:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = [ ]
+++

Despite its extensive standard library, Python does not have **record** or **struct** like data structures. **dict** became the de facto data type for both structured and structured data. This lead to the use (and abuse) of "ad-hoc" structures that can only be understood by digging through the codebase. It is not uncommon to find legacy code like this
```py
def parse(data):
    ...
```
or its more helpful type-annotated version
```py
def parse(data: dict):
    ...
```

What does data contain? what are its allowed keys? which values can the keys have? If you are lucky and the person responsible for that monstrosity still works on the project you can ask them, if not then get ready for some software archeology. In any case, we can, and we **should** do better. Do it for the next poor soul who has to modify that code.

Let's explore some options to improve this code. As an (unrealistic and didactic) example, let's pretend `parse(data)` takes a dict representing a writer with their `name`, `year_of_birth` and the `books` they have published. A valid input could be
```py
{
    "name": "Paulo Coelho",
    "year_of_birth": 1947,
    "books": [
        {"title": "The Alchemist", "year_of_publication": 1988}
    ],
}
```
(I am being generous with `year_of` key. I could have used a more realistic and infamous `date` key, which could be a timestamp, a `datetime` object, a `str`, etc.)

~~(I am also being generous by considering Paulo Coelho to be a writer)~~

## Dataclasses
Introduced in Python 3.7 standard library, [**Dataclasses**](https://docs.python.org/3/library/dataclasses.html) generate some dunder methods automatically for us based on type annotations. Additionally, they come with handy methods and properties that a regular class does not have, such as `frozen`, `asdict()`, etc.

Using dataclasses, our example could be written as follows
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

Much better! We don't have to navigate through the code to find out what `data` contains. However, we now have to modify the code to construct `Author` from `data`. Also, remember that type hints are just **hints**, and not enforced by the interpreter at runtime. The following code will run successfully
```py
book = Book(title="The Alchemist", year_of_publication="More than 30 years ago")
```
We have to rely on a type checker, such as [mypy](https://mypy-lang.org/), to catch the error
```
error: Argument "year_of_publication" to "Book" has incompatible type "str"; expected "int"  [arg-type]
```

We have to be careful when defining mutable default types. Let's say that an `Author` should be created with an empty list of `books` if a value is not provided. The correct way of doing it would be
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
We can solve this by implementing the nested parsing inside the [`__post_init__`](https://docs.python.org/3/library/dataclasses.html#dataclasses.__post_init__) magic method of the parent class.
```py
@dataclass
class Author:
    name: str
    year_of_birth: int
    books: list[Book]

    def __post_init__(self):
        self.books = [Book(**b) for b in self.books]
```
and we get our expected result
```py
Author(name='Paulo Coelho', year_of_birth=1947, books=[Book(title='The Alchemist', year_of_publication=1988)])
```

## TypedDict
Introduced in Python 3.8, [**TypedDict**](https://docs.python.org/3/library/typing.html#typing.TypedDict) are (as the name suggests) type annotations for dicts. **TypedDict** are still dicts under the hood. Similar to Dataclasses, we have to use a type checker to catch errors. They are more straightforward to implement than Dataclasses since we don't have to modify the existing code at all, only define the TypedDict and update the function header. The example would be as follows
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
Note that unlike Dataclasses, we cannot define default values. There isn't much to say about TypedDict, so let's move on.

## Pydantic
The only 3rd party option from this list. [Pydantic](https://docs.pydantic.dev/latest/) is a data validation module for Python. While it might look similar to Dataclasses for simple cases, Pydantic classes can get complex and the library is a beast of its own. I will only show a few features of the library, but I highly encourage you to go through the documentation, especially the examples, to get a grasp of how much it can do. For our example, let's start with the same specification we had in our dataclass section.
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
If we now attempt to create an instance with invalid data, we get a runtime error
```py
book = Book(title="The Alchemist", year_of_publication="More than 30 years ago")
```
```
pydantic_core._pydantic_core.ValidationError: 1 validation error for Book
year_of_publication
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='More than 30 years ago', input_type=str]
```
Pydantic is actually performing **data validation** at runtime! In fact, we can write our own custom validators. For example, we can validate that the `Author` name is not empty, and that the `Book` instances were published after the `Author` was born
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
However, if we change year_of_publication to 1945, it fails as expected
```
pydantic_core._pydantic_core.ValidationError: 1 validation error for Author
  Assertion failed, Author cannot contain books published before their birth [type=assertion_error, input_value={'name': 'Paulo Coelho', ...of_publication': 1945}]}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.5/v/assertion_error
```

One final caveat: by default, Pydantic attempts to cast the input to the expected type. The following code succeeds because the string "1988" can be casted to an `int`.
```py
Book(**{"title": "The Alchemist", "year_of_publication": "1988"})
```

Pydantic provides better safety, functionality and runtime guarantees compared to plain Dataclasses, and it is incredibly extensible and customizable, which also means one can get overwhelmed with so much customization options, models, etc.

As of writing this blog, Pydantic has recently gone through a major version update (1. -> 2.) with multiple breaking changes. It is likely that you will have to rely mostly on the official docs for learning and development, since most of the tutorials, blog articles, etc. are written for the 1.X versions.

# Conclusions
We explored some of the alternatives to plain dicts for structured data with their pros and cons. To summarize, my suggestions would be:
1. **Pydantic** when possible. Learn it regardless.
2. **Dataclasses** when you are allowed to refactor the code but you cannot use Pydantic.
3. **TypedDict** as last resort when you cannot modify existing code logic.

# Extra

## NamedTuple
If your structured data is represented as tuples instead of dicts you can use [NamedTuple](https://docs.python.org/3/library/collections.html#collections.namedtuple). It adds type annotations and you can also access the elements by name.
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

## NewType
Another cool pattern for validating data is to define a new type for the same data as an existing type but, unlike type aliases, they are treated as different types by type checkers. This behavior is useful when certain functions should only accept values returned from other functions that perform some kind of data validation. We can achieve this behavior with [NewType](https://docs.python.org/3/library/typing.html).
For example, if we had a `store` function for our data, we should only accept validated author dicts.
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

You can verify that the underlying data has not changed and that the returned value is still a dict. However, if we run mypy we get the following error
```
error: Argument 1 to "store" has incompatible type "dict[str, object]"; expected "ValidatedAuthor"  [arg-type]
```
While this pattern allows you to perform data validation using built-in data types, it is completely useless in Python because of mutability. The following code validates the data, then modifies it so that it becomes invalid, and then passes it to our function. The type checker does not report any errors. Oh well...
```py
validated_data = validate(data)
validated_data["books"][0]["year"] = validated_data["year_of_birth"] - 1
store(validated_data)
```