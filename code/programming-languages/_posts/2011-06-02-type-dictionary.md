--- 
layout: post
title: Type Dictionary
---
Imagine a programming environment where the type implementations need not be bound to a name or may be bound to multiple names. Type implementations just consist of a collections of methods and fields. A type is only associated with a name when it is registered in a directory service. The type need not be registered or it could be registered under multiple names depending on the whims of the programmer. So far this could be a description of any number of type systems for dynamic languages such as Ruby.

A type that refers to another type would use a particular resolution protocol to determine the type of the referred entity. Consider a type that declares that it is derived from the type “MyParentClass”; the simplest resolution protocol may state that it looks in the directory for that name in the current context. Another resolution protocol may state that it looks for the name within the context of the declaring class and then it each parent context until the name is found. So if the declaring class name is “MyClass” and it is in the context of “/mygroup/myapp” (i.e. it is bound to “/mygroup/myapp/MyClass”) then it may first look for the class in “/mygroup/myapp/MyParentClass”, then “/mygroup/MyParentClass” and finally “/MyParentClass” stopping if it finds the type. This resolution protocol is not dissimilar to the default resolution protocol used in the JVM where each context is a classloader and the name is the classname. The difference is that in the JVM each context can override the default behaviour and implement a custom resolution protocol and the contexts are just objects that need not be registered anywhere.

Using the directory as the central location to bind names to types has some interesting implications depending on the characteristics of the directory and the types. Can types be rebound? Or can names only be bound once? Are types that are candidates for garbage collection automagically unbound? Can an unregistered type be used to instantiate instances of objects or must it all be routed through the directory? Are contexts symbolically named or to they effectively use a PID. Are context PIDs accessible outside the context and are they forgeable? What are the rules for types in one context referring to types in another context? Is it disallowed or can only parent contexts be referred to? Do references from context A to context B restrict the ability of B to be garbage collected or does the collection of B force A to fault?

We could also go one step further and consider types to be just another “context”. Each instance of the type effectively has a local directory context that is examined when invoking a method, accessing a field or sending a message. So binding a type into the directory is effectively just binding a different kind of context into the directory.

So the directory consists of:

-   Names bound to a context that contain a collection of classes (a.k.a. packages or modules).
-   Names bound to a context that contain a collection of fields and operations (i.e. The Class).
-   Names bound to methods and names bound to fields. Private, protected and public access specifiers are mechanisms that change the resolution policy for fields.

This seems to indicate that functions and potentially fields should be first class objects that can be passed around. Creating a type is effectively binding functions or fields together and potentially also adding in binding parameters. A binding parameter determines whether a inherited type can rebind the name or access the bound value.

An interesting thought exercise. Maybe I will see how it plays out next time I feel the urge to write another interpreter.
