+++
title = "The Future of AI Assisted Development"
date = "2026-01-07T09:40:28+02:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = []
+++

*This is an opinion post*

*Note: the acronym AI in the context of this post refers to Large Language Models and Coding Agents.*

Entering 2026, the usage of AI for software development is still a controversial topic. While many developers and non-developers claim AI has boosted their productivity, scientific studies show that the adoption of AI hasn't had any noticeable impact yet[[1]](https://mlq.ai/media/quarterly_decks/v0.1_State_of_AI_in_Business_2025_Report.pdf), or worse, it has decreased productivity while developers still believe they were more productive[[2]](https://arxiv.org/abs/2507.09089).

There have been already articles aiming to predict where software development is going, for example, comparing this new software era to the industrial revolution[[3]](https://chrisloy.dev/post/2025/12/30/the-rise-of-industrial-software). There is also an infinite number of apocalyptic articles claiming that, by the year `n+5`, where `n` is the year you are currently reading the article in, AI is going to write all code. While there is some truth and some human hallucinations everywhere, the reality seems to point to a less exciting future.

The goal of this post isn't to validate, analyse, or discredit any existing study, but rather to guess what the possible future of AI-assisted software development can be. It's also not going to cover AI usage for creative tasks since that's a separate topic. As an aside, none of the posts of this blog use AI, not even for spell checking, and rather all the spelling, grammar, and styling rely on [Vale](https://vale.sh/).

## Where AI-assisted development succeeds
If you tried using coding agents, AI-integrated IDE, etc. for your job, you might have noticed that AI performs better the more constrained it's. Codebases containing exhaustive test suites and strict linter rules limit the possible set of programs the AI can write, making it more likely to produce a valid result. From experience, AI is the most useful in:
- Migration projects. For example from an old version of a language/framework to a newer one, or migrating code from one programming language to another one.
- Protocol implementations. Since protocols are fully defined, AI can write valid code for the parsing, encoding, and decoding tasks.
- Projects with exhaustive test suite. A good example of this is [bun](https://github.com/oven-sh/bun), which currently makes heavy usage of Claude for development. The project aims to be 100% compatible with Nodejs, thus having a comprehensive test suite. Additionally, the developers invested time into writing a [custom crash reporter](https://github.com/oven-sh/bun.report), which makes AI job easier.

## Comparison of conclusions and future predictions

### High level language
Almost every article reaches the conclusion that the era of AI-assisted development introduces a new *high level language*, and points to the prompts, `AGENTS.md`, and whatever the AI uses as input to write code. This statement is usually accompanied by mentioning the transition from binary -> assembly -> low level languages -> high level languages. The problem is that prompts aren't deterministic, and you can sometimes find cases where the AI ignores some guidelines from `AGENTS.md`.

The *high level language* that people are looking for already exists: **tests**. A comprehensive test suite including unit tests, functional tests, acceptance tests and performance tests, paired with strict coding rules should be enough to guarantee that the AI produces a valid output of expected quality. Tests and coding rules, which are specific and deterministic, are going to win over markdown files and prompts that aren't only ambiguous, but also the AI sometimes ignores.

If you want the AI to not write functions with more than X input arguments, and no longer than Y lines, you can write it in a file and pass it when the agent session starts, hoping that the AI follows it. You can also add it to your linter rules, let the AI code, raise the error, and let it refactor the solution. 

### The future professional
[Computer aided development](https://en.wikipedia.org/wiki/Computer-aided_design) has existed for decades and people still study mechanical engineering, civil engineering, electrical engineering, architecture, etc. Software engineering is going to take the same path: engineers are going to learn how to write code but they're going to do so sparingly in a professional setting. Instead, they're going to define the architecture and technologies, write tests, specifications, code/linter rules, let the AI do an initial version and manually modify it for better performance, and maintainability. Software engineers require the same amount of knowledge as before, or even more.

If you don't have the knowledge to choose the architecture and technologies for your system, and instead delegate it to the AI, how can you validate it? You could tell it to experiment with a few options, run benchmarks and performance tests, and then choose the best one, but that might be an expensive experiment. Imagine if an electrical engineer didn't know about different capacitor types, and built the same circuit 3 times, they're going to arrive to the *optimal* design but the research is going to cost them a lot.

If you don't know how to code, and the AI is incapable of producing a valid output for your specifications, then how are you going to fix it? A similar case happens when doing PCB design using any EDA software: the *auto-routing* feature can do an initial layout for you that saves a ton of time, but almost always requires some manual tweaking due to physical limitations of the manufacturer, potential interference, etc.

Assuming the price of AI is going to increase and match CAD and ECAD tools, then students don't have to worry about not learning how to code. As examples, [AutoCAD](https://www.autodesk.com/products/autocad/overview) and [Altium Designer](https://www.altium.com/develop/pricing) prices are out of range for university students, so they usually learn how to work with worse alternatives. By the time you can afford a pro license of any CAD, including AI in the future, you already have enough hands-on experience.

### The future of software quality
Most articles come to the conclusion that software quality is going to decrease because of the accessibility and lower costs of creating new software. While the quality of the average product might decrease, professional software quality is going to maintain or increase its quality. After some major accidents and a deserved bad reputation for software in general, software companies are going to enforce strict standards for their products. CI/CD pipelines are going to include at a minimum: e2e testing, minimum code coverage, vulnerability scanning and static code analysis.

There is also one random prediction/wish. On top of the existing Software Bill of Materials and the `audit` feature that many package managers already offer, in the future public packages and libraries can come with stricter requirements and information. For example, you could configure your project to only allow libraries that have > X% code coverage through testing, and that also pass a standard vulnerability scan. This doesn't mean that hobby libraries are going to disappear, but when including third-party code in a professional project, you are responsible for each dependency and can decide whether to include or not a package that doesn't meet certain standards.

## Summary
- Software engineers are going to write more tests and become more proficient at implementing different testing strategies such as E2E testing, property-based testing, performance testing.
- More usage of static code analysis tools and linters, and with stricter configurations and maybe custom rules. The industry switches away from storing prompt files.
- Average software product quality decreases, professional software product quality increases.
- Enforced stricter standards for software services and dependencies.

## Why it might not happen
- Writing a good test suite is many times as hard as writing the code itself.
- This new way of writing software makes the initial development phase slower.
- The software engineering discipline isn't mature enough yet and doesn't have established standards.
