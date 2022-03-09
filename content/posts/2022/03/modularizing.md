---
title: "Modularizing"
date: 2022-03-09T09:02:50+01:00
draft: false
toc: false
images:
categories:
- software development
  tags:
- kotlin
---

In the past few years I had the chance to work with modularization Android Apps in small, medium and large codebases. There are many articles on modularization - this one is mine.

Heads-up: If it is not clear yet, this article is my personal experience and it is heavily opinionated. Take it with a grain of salt.

Modularization can be very powerful and can bring many advantages when attempting to work at scale.

- Maintainability: a developer working in a specific set of modules is not required to know about the entire codebase. If you have a big quantity of teams, they can work separately without stepping on each other toes.
- Faster Build Times: changing a single module does not require to rebuild the whole codebase. For large projects, modules can also be built in parallel to optimise clean builds.
- Reusability: modules can be recomposed and combined to address new business needs on the go.

On the other hand, modularization is (as expected) a trade-off game. If you don't play your cards well, you can easily put yourself in a corner.

- Complexity: before trivial tasks, as for example navigation, need to be well designed to avoid centralized points of failure in your build (ie, a single module that is constantly changing and invalidates all other modules during build time).
- Dependency Relations: the team needs to actively design to avoid cyclic dependencies in the module structure.
- Build Logic: normally, modularization comes with tons of custom scripts and tune-in and out that must be done by an engineer or team.

Now that you have an idea of some pros and cons, let's go over modularization strategies that you can apply in your codebase to get the most of it.

## Do not premature modularize

The problem about hype in tech is that people tend to overuse it. I have seen very small teams with hundreds or thousands of modules. They would spend more time talking about hypothetical scenarios that would result in a over engineered modularization. Modularization for the sake of modularization does not help anyone.

Instead, you can focus in packaging your project in a way that modularize it later would be simple. Actually, I find fascinating how developers will focus on modularization without ever learning the basic of package design - for those who want to learn better about it, [Principles of Package Design: Creating Reusable Software Components](https://www.amazon.de/-/en/Matthias-Noback/dp/1484241185) is a must.

Heads-up: To be make it clear, some teams do have reason to modularize since day one - but if you can not justify the decision in a pragmatic way, try to avoid premature modularization.

## Module Levering

Not all modules have the same importance and that will ensure you the right amount of attention on each part of your system. Some of your modules should be designed to be stable and others are fine to be left as unstable.

- Stable Modules: they are foundational modules. They do not change often, and normally a large quantity of modules depend upon it. If your stable module are changing often, that is a smell and you should dedicate some time to refactor and change its design.
- Unstable Modules: they are changed often, mostly due to business needs. None to few other modules will depend on it. Normally, unstable modules are features or pieces of features that are shared within your application.

## Cohesion

One important principle in any good software is [cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science)) - and that is extra important when we are in the context of modularization. Some rules you can follow to guide you when understanding how cohesive a module is:

- Classes that changes at different rates belong in separate modules.
- Classes that change at the same rate belong in the same module.
- Classes not reused together belong in separate modules.
- Classes reused together belong in the same module.

## API Modules

### Separate Abstractions from Implementations

# Credits

Special credits for [Secure By Design](https://www.manning.com/books/secure-by-design) (Chapter 2: Shallow Modeling) and [A Philosophy of Software Design](https://www.amazon.de/-/en/John-Ousterhout/dp/1732102201) (Chapter 4: Modules should be deep) where I learnt about Deep and Shallow modeling, both exceptional books.

If you like my posts, follow me on Twitter: [@marcellogalhard](https://twitter.com/marcellogalhard)

# References:

- [Android at Scale @Square](https://www.droidcon.com/2019/11/15/android-at-scale-square/) by [Ralf Wondratschek](https://twitter.com/vrallev).
- [Preparing for Growth @Spotify](https://www.youtube.com/watch?v=sZuI6z8qSmc) by [Bruno Rocha](https://twitter.com/rockbruno_).
- [Build a Modular Android App Architecture](https://www.youtube.com/watch?v=PZBg5DIzNww) by [Yigit Boyar](https://twitter.com/yigitboyar).
