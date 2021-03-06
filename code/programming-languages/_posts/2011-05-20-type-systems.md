--- 
layout: post
title: Type Systems
---
Types are one of the most fundamental elements of a programming language. A type system classifies values and expressions into types. The type of an element identifies the set of allowable operations that can be performed on a type. The domain of a value is typically restricted by the type of the value. Type systems vary significantly between languages.

This page collects a number of thoughts on type systems and is by no means comprehensive. The original notes were derived from a brilliant reddit comment that I have since lost the link to. Additions and modifications have potentially changed the content significantly from the original source and any inaccuracies were most likely introduced by yours truly.

Type System Dimensions
======================

Static vs. Dynamic
------------------

Type checking aims to ensure that operations on a type are in the set of allowable operations and can be performed either during compilation or at runtime. A language where the majority of type checking occurs during compilation is said to be statically typed, while dynamically typed languages perform the majority of the type checking at runtime.

It is typical that the type of every expression can be determined at compile time in a statically typed language. A dynamic language is more likely to rely on a tag associated with a value during run time type checks.

The purpose of statically typed languages is to catch errors early on during the development cycle (i.e. during compilation). In some scenarios it may also be possible to optimize statically typed languages more effectively.

Dynamically typed languages tend to be more flexible than statically typed languages and can execute programs that would be marked as invalid by a static type checker. The downside to this is there are fewer a priori guarantees and development often need to be supported by comprehensive unit testing. (Although some would argue that this is not a downside and all programs should be supported by such practices).

Dynamically typed languages may also make use of more sophisticated type checking that takes advantage of both compile time and run time information. However this often means that the type checking has to occur with each execution of the program which often results in slower execution speed.

The static-dynamic is a spectrum and many static languages support dynamic type checks. Static languages such as downcasting, dynamic type checks and dynamic or “safe” casts (i.e. casts that fail if the type can not be safely cast to specified type).

Manifest vs. Latent
-------------------

Manifest typing requires the developer to explicitly declare the types of elements in the source code. Latent typing is the opposite, type declarations do not appear in the source code.

Historically statically typed languages were often equated to manifest typing. However type inference systems make it possible to infer the types at compilation time resulting in a latent+static type system. Some recent languages (i.e. Python 3000, Common Lisp, Dylan) make it possible to declare the types but will not check the types until runtime resulting in a manifest+dynamic type system.

Strong vs. Weak
---------------

In a strong type system an element can not be coerced into another type without an explicit cast. A weak type system attempts to coerce the elements to the required type through a variety of means.

Safe vs. Unsafe
---------------

Safe type systems disallow invalid operations. An unsafe type system make it possible for type errors to occur where an element of one type is manipulated as if it was an element of another type. This is often due to “unsafe casts” or explicit loopholes in the type system.

Unsafe languages exist to support a certain class of system level programming. These languages assume the developer knows what they are doing when they use possibly unsafe constructs and rely on the developer avoiding system corruption.

Nominal vs. Structural Subtyping
--------------------------------

Verifying that the type of an element is compatible with another type is central to most type checking systems. The simplest type systems only consider a type compatible if the types are equal or equivalent. However most type systems include a notion of subtyping where the compatibility between types is a transitive relation.

Nominal subtyping is where type relationships are explicitly declared (and named). Structural subtyping occurs when the type relationships are inferred from the structure of types. Most dynamically typed languages are also structurally subtyped.

Languages Classified
--------------------

<table>
<tr>
<th>
C

</th>
<td>
static, weak, manifest, nominal subtyping

</td>
</tr>
<tr>
<th>
C++

</th>
<td>
static, weak, manifest, nominal subtyping with structural subtyping available via templates

</td>
</tr>
<tr>
<th>
Common Lisp

</th>
<td>
dynamic, strong, latent or manifest, nominal and structural subtyping

</td>
</tr>
<tr>
<th>
Erlang

</th>
<td>
dynamic, strong, latent, structural subtyping

</td>
</tr>
<tr>
<th>
Java

</th>
<td>
static, strong, manifest, nominal subtyping

</td>
</tr>
<tr>
<th>
Ocaml

</th>
<td>
static, strong, latent, structural subtyping

</td>
</tr>
<tr>
<th>
Python/Ruby

</th>
<td>
dynamic, strong, latent, structural subtyping with nominal subtyping available

</td>
</tr>
<tr>
<th>
Scheme

</th>
<td>
dynamic, strong, latent, structural subtyping with nominal subtyping available

</td>
</tr>
<tr>
<th>
Haskell

</th>
<td>
static, strong, latent and optional manifest, structural subtyping with nominal subtyping available via newtype

</td>
</tr>
</table>

