# Hello!
```ex
myself = 
  Engineer.new(%{name: "Ignacio", degree: "Electronic Engineering"})
  |> Engineer.add_languages(["C", "Python", "JavaScript", "Elixir"], level: :professional)
  |> Engineer.add_languages(["Rust", "Gleam"], level: :hobby)
  |> Engineer.add_databases(["PostgreSQL", "MariaDB", "MongoDB", "NeptuneDB"])
  |> Engineer.add_cloud(["AWS", "GCP"])
  |> Engineer.add_interests(["Distributed Systems", "Cloud", "DevOps"])

Person.introduce(myself)
```

```ex
iex> """
Hi, my name is Ignacio. I am a Software Engineer with over 6 years of professional experience.
Over the course of my career, I have worked primarily with Python, Django and React, and currently I am working full time with Elixir as my primary language.

In this blog you can find my opinions and some small guides for solutions that I have implemented at work or for side projects that I could not find on the internet.
"""
```