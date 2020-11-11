---
title: "Connascence"
layout: post
excerpt_separator: <!--more-->
category: "Series Introduction"
tags:
    - Connascence
    - Roslyn
    - Static Code Analysis
    - Code Quality
---

When we speak about object oriented programming, we often speak of
cohesion and coupling. But what does this actually mean? Something
that is cohesive consists out of elements that seem to belong to
each other. And when we think of things that are coupled, then
they must be difficult to pull apart, right? So in a way, cohesion
is coupling between elements you want or need stay together.

However, I have hardly encountered any quantitative measures for
cohesion and coupling. How do you measure the degree of coupling?
From the above description, we can see that not only the existence
of dependencies is important, but also the distance between them.
Are they in the same class, namespace, assembly?

<!--more-->

In 1992 Meilir Page-Jones researched this. He coined the term
_connascence_. They defined it as a metric with three dimensions:

* Strength: the ease with which it can be refactored
* Locality: the distance between the coupled entities
* Degree: the number of times a specific instance occurs

The different types of connascence that they defined are (from
weak to strong):

1. Name
2. Type
3. Meaning
4. Position
5. Algorithm
6. Execution
7. Timing
8. Value
9. Identity

On this blog, I will probably not dig into all of these, but you
can read more about them on [this beautiful website](https://connascence.io).
For now, I'd like to dive deeper into the feasibility of detecting
them using static code analysis tools.

## Connascence of Meaning
As Page-Jones makes a distinction between static and dynamics types
of connascence, I was curious if there were existing analyzers for
the static types. It sounds like it should be possible, but I still
had a hard time finding any. So what does a developer do then?
Right, they attempt to make their own! So let's pick a type and get
to work, shall we?

I decided to go with [Connascence of Meaning](https://connascence.io/meaning.html).
It's not the weakest type, so it's probably something worth
refactoring. Also, in my experience it also occurs quite
frequently. It will be useful to have a tool to detect it.

### Description
Connascence of meaning is when multiple components must agree on
the meaning of particular values. Perhaps the most common example
of this is a comparing function:

```csharp
public int Compare(int a, int b) {
    if (a < b) return -1;
    if (a > b) return 1;
    return 0;
}
```

Every client of this function must know the meaning of the integer
that is returned. If this is part of an interface with multiple
implementations, all implementations must agree on the meaning of
the returned values, too!

One way to refactor this to a connascence with lower meaning is to
introduce some kind of constant or enum:

```csharp
enum Compared {
    LesserThan  = -1,
    GreaterThan =  1,
    Equal       =  1
}

public Compared Compare(int a, int b) {
    if (a < b) return Compared.LesserThan;
    if (a > b) return Compared.GreaterThan;
    return Compared.Equal;
}
```

This immediately reveals the meaning of the numbers by giving them
a name. Of course, everyone must agree and use this name, but with
modern IDE tools this is easy to change.

