# Hello
```ex
me = 
  Engineer.new(%{name: "Ignacio", degree: "Electronic Engineering"})
  |> Engineer.experience("Marvell", from: "Dec 2019", to: "Sept 2022")
  |> Engineer.experience("Growth Acceleration Partners", from: "Sept 2022", to: "Oct 2023")
  |> Engineer.experience("Octobot", from: "Oct 2023", to: "May 2024")
  |> Engineer.experience("Yolo Group", from: "June 2024", to: "Nov 2025")
  |> Engineer.languages(["C", "Python", "Elixir", "JavaScript"], level: :professional)
  |> Engineer.languages(["Rust", "Haskell", "Gleam"], level: :hobby)
  |> Engineer.databases(["PostgreSQL", "MariaDB", "MongoDB", "NeptuneDB"])
  |> Engineer.clouds(["AWS", "GCP", "OpenShift", "Hetzner"])
  |> Engineer.add_interests(["Distributed Systems", "Platform Engineering"])

Engineer.introduce(me)
```

```ex
iex> """
Hi, my name is Ignacio. I am a Software Engineer with over 6 years of professional experience across several industries.

I started my career working as a Software Development Engineer in Test at Marvell, where I applied my knowledge as an Electronic Engineer and my passion for software development to improve the quality of the company's firmware. For almost 3 years, I was part of a team that developed an internal testing framework in Python for testing low level protocols implemented in C. The internal framework was initially planned to support only 3 products, and it was extended to support over 20 products with minimal integration time. Additionally I took part in the development of a data analysis tool for electrical and electronic measurements.

I then switched to full stack web development with Django and React for almost 2 years on 2 different projects: I first worked at a SaaS that automated legal documentation for small businesses, primarily fixing bugs and improving the system performance. At my second Django+React job I worked on a government digital ID platform, where I was responsible for integrating with other governments using the OpenID Connect protocol and also bringing modern standards to the project, such as CI/CD pipelines, automated testing practices, etc.

At my last job I worked as a backend developer in automated fraud detection at an iGaming company, using Elixir and Phoenix as main language/framework. I was responsible for implementing self-exclusion (responsible gambling) functionality from scratch, and also improving the overall fraud detection system performance.

I am always looking for ways to improve the quality of the products I work on, which has lead me to learn multiple technologies by myself (Elixir, Rust) which, combined with my formal education in traditional engineering, has allowed me to deliver high quality maintainable solutions efficiently.

In this blog you can find my thoughts and opinions, and also some technical tips and solutions. Anything I feel worth sharing and that I could not find anywhere else.
"""
```