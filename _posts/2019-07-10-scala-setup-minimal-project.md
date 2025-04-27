---
layout: post
title:  "Scala setup a Minimal Project"
date:   2019-07-10 07:00:00 -0300
categories: scala
comments: true
---

## Why to have a quick way to run minimal projects

Sometimes you just want to do a proof of concept or try something quick, that's why it is a good idea to have a quick way to test things.

Sometimes the REPL, irb for Ruby or scala for Scala, is just not enough.

This post explain how to create a minimal project for Scala, with ScalaTest Library already setup.

## Prequisites

* Java 8 JDK

```bash
javac -version
```

* sbt

```bash
sbt sbtVersion
```

## Create the project

1. On the command line, create or go to the directory where your projects reside

2. Execute

```bash
sbt new scala/scalatest-example.g8
```

3. Choose an arbitrary name for the project

## Check everything is going well

4. Go to the project folder

```
cd ${projectName}
```

5. Clean, Compile, Test and Run the project

```bash
sbt clean compile test run
```

## References

* [GETTING STARTED WITH SCALA AND SBT ON THE COMMAND LINE](https://docs.scala-lang.org/getting-started-sbt-track/getting-started-with-scala-and-sbt-on-the-command-line.html)

* [TESTING SCALA WITH SBT AND SCALATEST ON THE COMMAND LINE](https://docs.scala-lang.org/getting-started-sbt-track/testing-scala-with-sbt-on-the-command-line.html)
