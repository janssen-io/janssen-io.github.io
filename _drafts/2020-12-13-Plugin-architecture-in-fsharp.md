---
layout: post
title: "Plugin Architecture in F#"
excerpt_separator: <!--more-->
---

# Plugin Architecture in F#
Last month I wrote about plugins in C#.

<!--more-->

## F# Advent Calendar
First things first. This post is part of the [F# Advent Calendar 2020](https://sergeytihon.com/2020/10/22/f-advent-calendar-in-english-2020/). Thank you, Sergey, for organizing this advent calendar year after year! Every year there is a ton of great content posted and I am proud to be part of it this time.

## Options
- Define an interface and go the C# route.
- Define function signatures and find functions with matching signatures
- Document a naming convection and find functions with the right names
- Combine the last two

## Interoperability with C#
- Define interface -> works
- Function signatures -> won't work, type is FSharpFunc

## Way to go for TQL
- Interoperability with C# matters if others want to extend it without F# knowledge
- I like that the compiler helps out
- Looks a bit out of place (no interface elsewhere in the app)

## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
