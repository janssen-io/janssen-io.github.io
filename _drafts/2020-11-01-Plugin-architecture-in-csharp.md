---
layout: post
title: "Plugin Architecture in C#"
excerpt_separator: <!--more-->
---

# Plugin Architecture in C#
What is a plugin? Generally a plugin is a piece of functionality that is not shipped with the core product, but can be enabled/disabled separately.

Often this is done by defining some sort of interface, or contract, that is implemented by the plugin. The core product does not care where the implementation comes from as long as it adheres to this contract.

In a statically typed language, one could achieve this by publishing the interface in a package that plugin developers will use.
This way, the compiler will ensure that the plugin will adhere to the contract. In dynamically typed languages, developers will have to make sure to adhere to this contract themselves.

<!--more-->

## Plugins in the .NET world
The most common pattern to support plugins in a .NET application is to create a dynamically linked library (DLL) and place it inside a designated plugin directory. The core product then dynamically loads plugins from this directory. This is often done on startup, requiring a reboot if plugins are added or removed, but that is not a requirement.

An example of a plugin in C# would look like this:
```csharp
namespace CoreProduct.Plugins {
    public interface IPlugin {
        string Execute(string input);
    }
}
```
```csharp
using CoreProduct.Plugins;

namespace CoreProduct {
    public class Product {
        private readonly IPlugin stringMapper;

        public Product(IPlugin stringMapper) {
            this.stringMapper = stringMapper;
        }

        public void SomeAction() {
            this.stringMapper.Execute("Input from core");
        }
    }
}
```

```csharp
using CoreProduct.Plugins;

namespace ThirdPartyDev.CoreProductPlugin {
    public class InputMapper : IPlugin {
        public string Execute(string input) {
            return $"ThirdParty InputMapper received: {input}";
        }
    }
}
```

## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
