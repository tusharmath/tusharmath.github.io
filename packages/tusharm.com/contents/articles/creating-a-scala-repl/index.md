---
title: Creating a Scala REPL
hide: true
date: 2018-08-21
template: article.pug
---
The best way for me to learn a new language is to create a simple file for that programming language and keep running it in watch mode. Here are some ways you can achieve this —

### Option 1, Simple watch

```bash
watch scala Program.scala
```

#### Pros

- Simple setup.

#### Cons

- Doesn't work with 3rd party dependencies/libraries.
- Very slow.
- Doesn't work with multiple files.

### Option 2, Using [SBT]

SBT provides a standard way to watch and run scala code. Though this is designed for larger projects, it can be used on projects with one file.

**Directory Structure**

```
|-src
  |-main
    |-scala
      |-Main.scala
build.sbt
```

Run the following command on the console.

```bash
sbt ~run
```

**Pros**

1. Standard technique to run any scala projects.
2. Supports adding a library
   This way the `Main.scala` is only executed when the file actually changes. The disadvantage of this approach is that you can not pass a random file path.

Also checkout **[repl.it]** if you don't want to setup all this stuff locally. Unfortunately at the time of writing this blog, scala wasn't supported.

There is a bit of additional bootstrapping that is required with SBT. The advantage is that you can easily add dependencies into the project thru `build.sbt` if you want to play with them.

[repl.it]: https://repl.it/
[sbt]: https://www.scala-sbt.org/

### Option 3, Using [Ammonite]

[ammonite]: http://ammonite.io

This is possibly the best alternative out there!
