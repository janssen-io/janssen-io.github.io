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
a name. Of course, everyone must agree and use this name. The
reason this is a weaker type of connascence is that with modern
IDE tools this is easy to change. Those tools analyze the code and
catch all the references.

### Using static analysis to find occurrences
Now that we have a rough idea of what connascence is, how can we
detect it on our code? I eagerly started writing an analyzer for
C#, but soon encountered that I had many false assumptions. Let's
take a closer look at the various trickeries of detecting
connascence of meaning.

Initially, I was hoping to detect CoM by looking at individual
cases. However, I soon found out that, as we are speaking of
coupling, this was not sufficient.

#### Example 1: Extracting a constant
For example, take the representation of roles as integers:

```csharp
public BanUser(Guid userIdToBan, Guid adminId)
{
    User admin = GetUser(userId);
    if (admin.Role != 0)
        throw new InsufficientPermissionsException();

    // ...
}
```

Both the part of the code responsibility for assigning roles
as this method must agree that role `0` represents an
adminstrator. Now if we lift the literal to a constant, it seems
like it is no longer CoM, but now is CoN. However, we have only
moved the CoM to another place. If all parts that convert from or
to this integer reference the constant, we are golden. But if
there exists at least one part that does not or cannot reference
it, we have CoM once again.

Now the next problem is: how do we know that two (or more)
literals, variables or constants that share the same value
actually also share the same concept?

#### Example 2: The type is too forgiving
Another example is shown below, where the type of the variable
does not sufficiently express what the meaning of it is.

We have a method that will send a reminder to some e-mail address
if more than 7 days have passed:

```
public SendReminder(int timeAgo, string e-mailaddress)
{
    if (timeAgo > 7) { /* send e-mail */ }
}
```

The integer literal `7` has a meaning to it, but whether it's
seconds, weeks or years you can't tell from the code. Renaming the
variable to `daysAgo` might make it clearer locally, but anyone
working with milliseconds might still accidentally send the millis
instead of converting to days first.

So, this is fairly easily detectable. Just search for a condition
with an (integer) literal in the binary expression. But what if
the literal is stored in a constant? Both the client of this
method as the method itself could reference it, but it will not
make the meaning clearer. So it does not move the CoM nor does it
make the connascence weaker by reducing it to CoN.

The difference with Example #1, however, is that the client of this
method will not reference the new constant. The constant is very
specific to this use case. So perhaps we can recognize this
instance, by noticing that the call site passes in a seemingly
unrelated value that will be compared to some "arbitrary" other
value.

But how do we notice that the type caries a specific
meaning? We humans might be able to detect that `timeAgo` is
ambiguous, but can a computer detect it? Let me know in the
comments if you have any thoughts about this!

## Conclusion
Creating an analyzer to detect Connascence of Meaning is not as
easy as I initially thought. Simply detecting conditions where
some variable is compared to a literal is not sufficient.

Firstly, the literal can be moved elsewhere, so we need to keep an
eye on all places where this specific instance of CoM occurs. But
how do we know that two variables/constants assigned `0` mean the
same thing?

Secondly, the type can be too liberal and thus accepting values
that might mean something else than what we intended when writing
the method. Detecting the meaning of a certain type and whether it
is too liberal or not is a difficult task for a computer.

## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
