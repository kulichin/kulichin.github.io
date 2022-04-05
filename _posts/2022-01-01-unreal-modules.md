---
title: Unreal Engine Notes — Modules
date: 2022-03-31 11:58:47 +07:00
---

**Links: [Unreal Engine - Modules](https://www.youtube.com/watch?v=DqqQ_wiWYOw)**

## Contents
- [Modules in general](#modules-in-general)
- [Private/Public Dependency](#privarepublic-dependency)
- [Precompiled Headers](#precompiled-headers)
	- [When to use what PCH's](#when-to-use-what-pchs)
	- [Private PCH](#private-pch)
	- [Shared PCH](#shared-pch)

# Modules in general
Think of modules as dll's.

It exposes any public properties or functions of our class: `[YourModuleName]_API`

# Privare/Public Dependency
Private dependencies are preferred as they reduce compile times. Forward declare when you can.

Missing module dependencies will produce compiler or linker errors.

# Precompiled Headers
PCH define one header file that includes your most common header files and gets compiled before other files. It doesn't compile again, unless any of it's included headers are changed.

## When to use what PCH's?
You have three options for precompiled headers:
- Own private PCH
	- Good for modules with very big codebases
		- Often the case with the primary game module on bigger games
	- You have to decide what to put in there and how to balance it
- Use a shared engine PCH
	- Good for all smaller modules
- Don't use a PCH
	- Not really practical

## Private PCH
This a custom type of PCH that your create for your module.

### How to create?
- Define it in your `.Build.cs` file:
```c#
PrivatePCHHeaderFile = "FooBarPrivatePCH.h";
```
- Never include it yourself in your `.h` or '.cpp' files
	- UBT will automatically inject it for all compiled files in your module
- PCH's should be considered an optimization layer:
	- Don't treat it as an easy "include all", still include only what you use
	- Your code should compile even if PCH's are turned off
	
## Shared PCH
Instead of defining your own PCH you can use a shared PCH
- A shared PCH is when a module defines a PCH for other depending modules to use
- Exists in some foundational often-used UE4 modules
	- UnrealEd, Engine, Slate, CoreUObject and Core to be specific
- Only engine modules can create a shared PCH

A shared PCH will only get compiled once, even if multiple modules use it.

UE4 will choose the "highest priority" shared-PCH to use for your.
- Sorted by how many other modules with a shared PCH it depends on.