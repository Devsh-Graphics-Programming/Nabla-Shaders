# Nabla Shader Compiler Library

A tool to meta-compile a subset of C++20 (mostly omitting runtime Standard Library features) to a domain specific Abstract Syntax Tree.

Also a library to allow you to transliterate entire GLSL codebases to C++, **it does NOT attempt to implement or define a _new_ shading language!**

This AST can be compiled, by a variety of backends, into beautiful name mangled GLSL, HLSL2021 or even SPIR-V.

This project is developed to be used by the [Nabla](https://github.com/Devsh-Graphics-Programming/Nabla) Rendering Framework, however after realizing its potential we decided to make develop it in a standalone repository.

## The cool thing is that we don't replace your existing Shader Development toolchain, we merely augment it

Same way how you can link C++ to C or mix with other languages (if you only know the name mangled entry points), you can `#include` or otherwise concatenate the output of our compiler backends with regular dumb GLSL (or HLSL) code.

# PROJECT IS CURRENTLY IN CONCEPTUALIZATION STAGE, DO NOT EXPECT TO BE ABLE TO USE IT

# Features compared to GLSL

- Namespaces
- Methods
- Templates

# Features compared to HLSL2021

- Templates more complex than C++03
- Ability to express all of GLSL (inline GLSL, extension enablement)
- Shader Module Linking (work with compilation results like you'd work with static libraries in C++)

# How it works

## Declarations of GLSL builtin types and functions as a C++ Library 

We use GLM for basic math types, but have our own prototypes for various GLSL language builtins.

This way you can transliterate GLSL to valid C++, although without the ability to run certain parts of it on the CPU without providing implementations for certain abstract classes (like `sampler`).

You might notice that all types in the Library are templated on the `EvalMode` class parameter, but more on that later.

## Using Clang into a Meta-Compiler

Our CMake module provides you with a function you can use in your own build system to compile the C++ code from the step above as you would a single Shared Library, this means being able to:
- split your shader codebase arbitrarily across headers and translation units
- linking your shaders using static libraries
- manually tagging which functions should be exported

Given the list of exported functions, our magical tool does the rest.

Secret sauce, TODO explanation.

**NOTE:** Due to the need to use our custom Clang build, reasons outlined in the "Dependencies" section below, you will experience the following issues:
- you cannot declare any of the methods and functions as inline, they all need to be exported and defined in private translation units
- the C++ standard library runtime will be statically linked into every DLL you compile, so better group your shaders together
- this step does not produce a regular Shared Library CMake dependency as it needs to ignore your chosen toolchain compiler
- the shader code cannot be compiled to a Static Library due to runtime mismatch
- we prevent you from using any Standard Library data types (different compilers implement them differently) in your shader's headers/API

#### `noHostCode` = do not instantiate execution evaluation mode templates

This can save you a bit in the size of the DLL module.

#### `cAPI` = use SWIG to create C bindings for your shader code

Its basically an explicit import library with public headers.

## Shared Library Output

You essentially get two versions of your shader code (two template instantiations).

### Execution Evaluation

When you call this function overload it performs the computations on the inputs as written in your source code.

### Abstract Syntax Tree Building

When you call this function overload:
1. The arguments are not values but references to values
2. It does not perform the computation but records the computations to be performed
3. It gathers the definitions of any types it encounters
4. It gathers the declarations of any globals it might encounter
5. Returns Abstract Syntax Trees of the function as well as struct/class declarations

Thanks to some template magick this function call is almost a `consteval` (we don't actually build the AST on the fly) so your shaders load ultra fast.

## [BONUS ROUND] Linking

Because our AST building implements Consing (by abusing the C++ template type instantiation system), and we don't complain about missing symbols until we hit the backend, you can merge two AST compilation results using the following policies:

#### Multiple Definitions Error

If you have two implementations of the same function in your compilation results we return an error/empty result.

#### By dominance

The first definition encountered hides all others.

## Compilation by Backend to target GPU Language

You can run simple codegen on the ASTs to emit some relatively sane name mangled "High Level" Shading Language source.

Actually as long as you don't embed the C++ code in any class/struct method or namespace, you will get no name mangling, and the autogenerated GLSL will be identical to GLSL written by hand.

## [BONUS ROUND] Running your shader code on the CPU

TODO

# LICENSE (can I use it?)

The code in this repository is Dual Licensed to:
- DevSH Graphics Programming and any of its Clients, as under Apache 2.0
- to anyone else, as Mozilla Public License 2.0

However these licenses only cover the Library's source code, not any output generated by its execution (whether compile time via template metaprogramming, or runtime), so the shaders you write and their compilation results remain your property to do with as you please.

# Commercial Support

Being a Graphics Programming Consulting company we offer targeted development (payment for new or prioritizing features), integration into exisitng codebases and general support & training as part of our services.

# Backends

Due to current profit incentives DevSH Graphics Programming will only support and maintain a GLSL backend. All other backends are the responsibility of external contributors.

### [TODO] HLSL

### GLSL

#### Compile Options

`exportAll=None,Public,Protected` whether to skip attempting to hide (via prefixing the compilation GUID into symbol name) the functions, methods and static variables which have not been explicitly marked for export.

# Dependencies

Already present in the repo as submodules, will be turned off if your cmake already defines the variables we are looking for.

### Clang

Need to build a custom version of clang from source because we need the branch with the prototype implementation of the Reflection TS.
As soon as Reflection TS makes it into your CMake toolchain of choice's compiler, you can stop using our custom clang.

### GLM

For implementations of basic GLSL math types like `vec4`, etc.

### SWIG (optional)

To generate C-API headers and marshalling code for the output DLLs.

# Inspirations

SwiftShader's Reactor system for the idea to use overloaded operators on faux-standard-types with AST return types to build AST by writing what semantically looks like regular GLSL code.

ShaderWriter for an initial idea of the syntax for declaring new structs and being an example of the room for improvement.
