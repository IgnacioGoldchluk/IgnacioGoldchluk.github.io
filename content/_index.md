# Hello!
```ex
Person.new(%{name: "Ignacio"})
|> Person.graduate("Electronics Engineer")
|> Engineer.switch_career(to: "Software Engineer")
|> Engineer.add_languages(["C", "Python", "JavaScript"], experience: :professional)
|> Engineer.add_languages(["Elixir", "Rust"], experience: :hobby)
|> Engineer.add_databases(["PostgreSQL", "MariaDB", "MongoDB"])
|> Engineer.add_interests(["Distributed Systems", "Cloud", "DevOps"])
|> Engineer.introduce()
```
```ex
iex> """Hi! My name is Ignacio. I am a software engineer with over 4 years of professional experience.
I work primarily using Django and React, but I also like learning different technologies and different ways of solving problems.

I studied Electronics Engineering but I fell in love with Software Engineering while designing circuits and writing simulations for my final project.
Programming to me is about modeling a problem in such a precise and elegant way that the solution becomes trivial.
I hope to convey to you the same passion I have for programming and software engineering through this blog! 
"""
```
