---
layout: post
title: "Plugin Architecture in C#"
excerpt_separator: <!--more-->
category: Software Architecture
tags:
    - C#
    - Plugins
    - Architecture
---

What is a plugin? Generally a plugin is a piece of functionality that is not shipped with the core product, but can be enabled/disabled separately.

Often this is done by defining some sort of interface, or contract, that is implemented by the plugin. The core product does not care where the implementation comes from as long as it adheres to this contract.

In a statically typed language, one could achieve this by publishing the interface in a package that plugin developers will use.
This way, the compiler will ensure that the plugin will adhere to the contract. In dynamically typed languages, developers will have to make sure to adhere to this contract themselves.

<!--more-->

## Plugins in the .NET world
The most common pattern to support plugins in a .NET application is to create a dynamically linked library (DLL) and place it inside a designated plugin directory. The core product then dynamically loads plugins from this directory. This is often done on startup, requiring a reboot if plugins are added or removed, but that is not a requirement.

An example of a plugin in C# would look like this:
```csharp
namespace CoreProduct.Plugins
{
    public interface IPlugin
    {
        string Execute(string input);
    }
}
```
```csharp
using CoreProduct.Plugins;

namespace CoreProduct
{
    public class Product
    {
        private readonly IPlugin stringMapper;

        public Product(IPlugin stringMapper)
        {
            this.stringMapper = stringMapper;
        }

        public void SomeAction()
        {
            this.stringMapper.Execute("Input from core");
        }
    }
}
```

```csharp
using CoreProduct.Plugins;

namespace ThirdPartyDev.CoreProductPlugin
{
    public class InputMapper : IPlugin
    {
        public string Execute(string input)
        {
            return $"ThirdParty InputMapper received: {input}";
        }
    }
}
```

## Loading plugins
The missing link in the above application is the part that actually loads the plugin from the DLL. This could look something like this:

1. Find the assembly containing the plugin
2. Find the concrete implementation of our plugin interface
3. Create an instance of this implementation
4. Use this instance in our core product

```csharp
using System;
using System.Reflection;
using CoreProduct;
using CoreProduct.Plugins;

namespace CoreProduct.Console
{
    class Program
    {
        static void Main(string[] args)
        {
            string pluginDirectory = Path.GetFullPath("./plugins");
            string pluginLocation = Path.Combine(pluginDirectory, args[0]);

            Assembly pluginAssembly = Assembly.LoadFile(pluginLocation);
            Type pluginInterface = typeof(IPlugin);
            Type plugin = pluginAssembly.GetTypes()
                .Where(type => pluginInterface.IsAssignableFrom(type));
            IPlugin instance = Activator.CreateInstance(plugin);

            Product core = new Product(instance);
            core.SomeAction();
        }
    }
}
```

Running this with the plugin assembly inside a local _plugins_ directory would result in the following:

```bash
$ dotnet run CoreProduct.Console ThirdPartyDev.CoreProductPlugin.dll
ThirdParty InputMapper received: Input from core
```

And that's all there is to it! One note of warning, however, all code from the plugin assembly that is called from our core product will be executed with the same rights as the rest of our application. So make sure you (or your users) trust the plugin developers!

Next month, I'll be writing about how to use the same strategy to use plugins in your F# application as part of Sergey's [F# Advent Calendar](https://sergeytihon.com/2020/10/22/f-advent-calendar-in-english-2020/).

## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
