---
layout: post
title: "Plugin Architecture in F#"
excerpt_separator: <!--more-->
---

# Plugin Architecture in F#
Last month I [wrote about plugins in C#]({% post_url 2020-10-25-Plugin-architecture-in-csharp %}). Using a predefined interface, 3rd party developer could develop plugins. The plugin is then loaded from a DLL in a designated directory and the implementation of the interface is instantiated.

In F# we could use this same approach, as F# also allows us to work with objects and C#-style interfaces. But what other possibilities does F# offer and what their pros and cons?

<!--more-->

## F# Advent Calendar
First things first. This post is part of the [F# Advent Calendar 2020](https://sergeytihon.com/2020/10/22/f-advent-calendar-in-english-2020/). Thank you, Sergey, for organizing this advent calendar year after year! Every year there is a ton of great content posted and I am proud to be part of it this time.

## Plugin Architecture Options
With F# we can take one of four routes. The first one being the same as [C#]({% post_url 2020-10-25-Plugin-architecture-in-csharp %}), so I won't cover that in too much detail in this post.

The second is to define a function signature that plugins have to implement. We can then load the module and find the function with the same signature to execute the plugin.

Alternatively, we could create a naming convention for our plugin and _pray_ the function signature will match.

Finally we can combine the last two, define a function signature that the plugin can reference and expect developers to use a specific name for the function that is to be called upon executing the plugin.

### Predefined interface
In F# we can also define an interface and share it with our plugin developers:
```fsharp
type IConverter =
    abstract member Read : FilePath -> seq<Row>
    abstract member Map : Row -> Entry
    abstract member DateFormat : string
```

Then find the implementation in a runtime-loaded DLL and instantiate it:
```fsharp
let load pluginName pluginDirectory = // string -> string -> IConverter option
    let context = new AssemblyLoadContext("PluginLoader", true)
    (Path.GetFullPath pluginDirectory, pluginName)
    |> Path.Combine
    |> context.LoadFromAssemblyPath
    |> fun assembly -> assembly.GetTypes ()
    |> Array.tryFind (fun t -> (typeof<IConverter>).IsAssignableFrom t)
    |> Option.map (fun t -> Activator.CreateInstance t :?> IConverter)
```

This has the benefits that the plugin DLL can be created by any .NET language that supports implementing such an interface. So we support both plugins written on F# and C#. However, for TransactionQL, I'm not very fond of introducing this more 'object-oriented' approach. So let's take a look at the alternatives!

### Predefined function signature
Similar to the predefined interface approach, we first define a function signature that needs to be implemented in the plugin.

```fsharp
type Reader = FilePath -> seq<Row>
type Mapper = Row -> Entry
type DateFormat = () -> string
```

Then we need to load the assembly and find the implementation.
```fsharp
let load pluginName pluginDirectory =
    let context = new AssemblyLoadContext("PluginLoader", true)
    (Path.GetFullPath pluginDirectory, pluginName)
    |> Path.Combine
    |> context.LoadFromAssemblyPath
    |> fun assembly -> assembly.GetTypes ()
    |> Array.tryFind FSharpType.IsModule          // Assuming the module consists out of one module
    |> Option.map (fun modl -> modl.GetMethods()) // Get all functions inside that module
    |> Option.bind (Array.tryFind                  // Find the function that matches the signature
        (fun method -> typeof<Reader>.IsAssignableFrom (method.GetType ())))
```

Then, to cover all of our functionality, we also need to load the Mapper and the DateFormat functions in a similar fashion.
All in all, this is a bit more involved and we already need to make more assumptions about the design of our plugins.

Firstly, interoperability with C# is harder, because `typeof<Reader>` shows our signature is of a type `FSharpFunc`. This makes it more difficult to provide a plugin writtein in C#. One solution for this could be to figure out what the parameters and return type are and search for a method that matches that. But that would make our solution a lot more complicated.

Secondly, we are assuming that it only consists out of a single module. It is not unimaginable, however, that more complex plugins could consist out of multiple modules. In that case we would either have to setup a naming convention or scan all modules for the appropriate function signature.

And lastly, what do we do if we find multiple implementations? Especially if our signature isn't very specific (e.g. `string -> string`), it is quite likely there can be multiple functions matching the signature. This could be simplified by only looking at publicly exposed functions. But that might still clash in case of multiple modules. So how about using a name convention?

### Name conventions
As you can probably guess, using this method we first define the name of the function we will execute.

```fsharp
let readerName = "reader"
```

Then we need to load the assembly and find the implementation based on this name.
```fsharp
let load pluginName pluginDirectory =
    let context = new AssemblyLoadContext("PluginLoader", true)
    (Path.GetFullPath pluginDirectory, pluginName)
    |> Path.Combine
    |> context.LoadFromAssemblyPath
    |> fun assembly -> assembly.GetTypes ()
    |> Array.tryFind FSharpType.IsModule          // Assuming the module consists out of one module
    |> Option.map (fun modl -> modl.GetMethods()) // Get all functions inside that module
    |> Option.bind (Array.tryFind                  // Find the function that matches the name
        (fun method -> method.Name = readerName)))
```

This code looks even more similar to the predefined function signature method, than that code did to the predefined interface method.
The problem this method solves, however, is the one of multiple functions with the same signature inside one module.

There could still be multiple modules, possibly each with a function named `reader`, but as long as we have one module, we know there can be at most one function with our function name.

The next problem we have is that there is no compiler checking if the function signature is actually correct. So we might run into exceptions during runtime. Let's see if combining these two approaches can alleviate this problem.

### Predefined signature + Name convention
To combine the last two approaches, we define both the name as the signature of the function that needs to be implemented. The name is more of a convention, we cannot enforce it during plugin development. The signature can be part of a package that we can reference during plugin development to let the compiler help (read: force) us to adhere to it.

```fsharp
let readerName = "reader"
type Reader = string -> seq<Row>
```

Then we combine the logic of finding the implementation of the signature:
```fsharp
let hasName name methodInfo =
    methodInfo.Name = name

let implementsSignature signature methodInfo =
    signature.IsAssignableFrom (methodInfo.GetType ())

let load pluginName pluginDirectory =
    let context = new AssemblyLoadContext("PluginLoader", true)
    (Path.GetFullPath pluginDirectory, pluginName)
    |> Path.Combine
    |> context.LoadFromAssemblyPath
    |> fun assembly -> assembly.GetTypes ()
    |> Array.tryFind FSharpType.IsModule          // Assuming the module consists out of one module
    |> Option.map (fun modl -> modl.GetMethods()) // Get all functions inside that module
    |> Option.bind                                // Find method matching the name and signature
         (Array.tryFind (fun method ->
             hasName readerName method
             && implementsSignature typeof<Mapper> method))
```

Combining the two plugin loading methods offers the advantage of having a single implementation of the function per module and knowing the signature will be correct. However, we still cannot enforce the right name during development and doesn't specifying both a name and signature remind you of something? An interface already defines the name and signature of its methods.

## Publishing your application
While working on the plugin architecture for TransactionQL, I was met with a curious error:

> Could not load file or assembly 'netstandard, Version=2.1.0.0, Culture=neutral, PublicKeyToken=cc7b13ffcd2ddd51'. The system cannot find the file specified.

Apparently you shouldn't use the `-p:PublishTrimmed=true` flag. Publishing it as either `--self-contained=true` or `false` works, though! So the only way to save some space is by making it _not_ self-contained, which means the platform where it will run will need to have dotnet installed.

## Conclusion
So I would argue that, for this purpose, it's best to stick to using an interface. While it brings a little OO into what could be a 100% functional application, it does bring a lot of advantages with it:

|     Approach     | C# interop | Enforce signature | Enforce name |           Easily loadable           |
|:-----------------|:----------:|:-----------------:|:------------:|:------------------------------------|
| Interface        |      ✔     |         ✔         |       ✔      |        ✔                            |
| Signature        |            |         ✔         |              | Might have multiple implementations |
| Name convention  |            |                   |       ✔      | Might have the wrong signature      |
| Signature + Name |            |         ✔         |       ✔      | Might not use the right name        |

In my TransactionQL project I also opted for the interface approach. You can check out the code [here](https://github.com/janssen-io/transactionql-fsharp) (plugins are loaded in the [Console project](https://github.com/janssen-io/TransactionQL-fsharp/blob/master/TransactionQL.Console/PluginLoader.fs)).

## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
