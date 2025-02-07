# Master thesis notes

This document is a live document tracking the progress and development of my master thesis project.
It will be updated as the project evolves.

## Goals

**Problem Statement**: Scafi is a programming model for aggregate computing that enables writing distributed programs in a high-level, declarative manner, based on _field calculus_. 
The Scafi library is implemented in Scala and designed for the JVM platform.
Alternative implementations have been developed in different languages, including C++, Kotlin, and Rust to address performance needs or support different platforms, such as mobile or edge devices. 
However, these implementations have been created independently from scratch, with no code reuse, and no compatibility in mind.

The **goal** of the thesis is to re-engineer scafi architecture to achieve multi-platform and multi-language independence.

More in details, the thesis aims to:

- define an extensible architecture for scafi allowing to target heterogeneous devices, both JVM-based and native, and to support web and non-web environments;
- nimbly support the development of Scafi programs in various languages, such as (but not limited to) C++, Python, Kotlin, and Rust.

In the end the contribution of this thesis should enable the aggregate programming community to effortlessly use Scafi in diverse computational environments and with different programming languages.

## Requirements

-- Requirements are still incomplete and need to be refined --

### Functional requirements

#### User requirements

1. The library client should be able to write Scafi programs in different supported languages through a consistent API. Supported languages must include:
    - _TBD_
2. The library client should be able to target different platforms, including:
    - JVM-based platforms;
    - native platforms;
    - web platforms.

#### System requirements

1. The core aggregate computing abstractions and API implementation should be shared across all supported languages and platforms.
2. The system should provide the ability to target different platforms through a consistent user-side API for each supported language.
3. The infrastructure should provide a mechanism to automatically generate user-side API bindings for each supported language so that any change can be automatically propagated without manual intervention.
4. Programs written in different languages should exhibit equivalent behavior, regardless of the platform.
5. For JVM the library should ensure compatibility with the existing Scafi library.

### Non-functional requirements

1. Performance: _TBD_
2. Extensibility: the architecture should allow for easy addition of new language bindings and platform-specific optimizations / logic.
    - _TBD what is meant by "easy"_
3. Reliability: the architecture should ensure robustness in program execution across platforms with clear error reporting.
4. The user-side API should be idiomatic to the target language, providing a seamless experience for the library client.

## Coarse grained architecture

The architecture follows the Basilisk architectural pattern [^1] which aims to establish clear boundaries to decouple software products from both the underlying platforms and user-end programming languages.

![architecture](./diagrams/architecture.svg)

Highlights:

- the abstraction layer leverages scala multi-platform compiler capabilities, targeting JVM, Javascript and Native platforms.
  - the _product-to-platform_ interface is a contract defining the platform-specific functionalities, which are appropriately implemented in each supported backend according to the target platform.
  - an adapter of the exposed API will be implemented to be accessible from all the platforms.
    - for example, in native platform, leveraging scala native, an adapter will be implemented to make the API accessible from native languages through C ABI.
- the transpilation infrastructure allow to generate a consistent user-side facade over the scafi API, tailored to the specific target language of choice (similarly to the approach described in [^1]).
  - important: the _user-to-product_ interface contains only the bindings to access the software product, implemented using a _foreign function interface_ (FFI) library for the specific targeted language.
    - for example, when targeting the native platform, the transpilation infrastructure must be able to generate bindings via the C ABI using ad-hoc FFI libraries (e.g. [`swig`](https://www.swig.org/)).

Challenges to address:

- shared code must use _cross-platform_ libraries and features;
  - currently scafi `spala` module leverages Akka actors, which is not _cross-platform_;
- for native and web platforms crafting a facade over the scafi API that can be called from another language using a FFI is not trivial;
- some specific language adapter may be required to better fit the target language idioms and conventions.

## Work plan

The initial work plan is to start with a proof of concept to validate the architecture.

1. Implementation of a simple DSL inspired to the _Sapere_ incarnation using scala multi-platform;
2. creation of a native interoperable layer over the DSL that make it possible to call it using FFI libraries in other programming languages;
3. creation of a minimal transpilation infrastructure inspired to Hydra [^1], initially for a small set of languages;
4. add distribution;
5. creation of a JS interoperable layer

[^1]: [When the dragons defeat the knight: Basilisk an architectural pattern for platform and language independent development](https://www.sciencedirect.com/science/article/pii/S016412122400133X?via%3Dihub)

## Proof of concept

The selected proof of concept for testing the feasibility of the architecture is the case of Distributed Asynchronous Petri Nets.

The code is available [here](https://github.com/tassiluca/dap).

### 1 + 2) Multi-platform and native interoperable layer

Key points:

- for targeting native platform, [Scala Native](https://scala-native.org/en/stable/) have been used
  - it provides scala bindings for C programming constructs, standard library and a core subset of POSIX libraries, making it interoperable with C code
  - leveraging sbt [cross-project plugin](https://github.com/portable-scala/sbt-crossproject) it is possible to compile for native platform and generate shared / static libraries
  - ! scala native do not generate header files for the C code (differently from Kotlin Native)
  - A library that can help generating bindings for C libraries is [Scala Native Bindgen](https://sn-bindgen.indoorvivants.com) even if in our case we don't have an external C library to bind to but we want to expose a C API for our scala code

- the data structures and methods that are part of the public API have been re-written using C programming constructs (e.g. `struct` and prototypes) so that from C it is possible to properly interact with them $\Rightarrow$ this incarnates the _native interoperability protocol_ [^2]
  - scala shared data structures are converted in their C counterparts back and forth using appropriate conversion methods
  - the C prototypes are implemented in scala native and linked to the C code during the compilation phase (through `@exported` methods in scala native)
  - _transpilation infrastructure_ will be in charge of generating this
- one important aspect is how to make the C API generic:
  - for generic types opaque pointers are used.
  - the scala code only knows the generic type is a pointer to an opaque structure, hence it can only pass it around without knowing its internals, while the C code knows the actual implementation of the structure and can properly interact with it
  - _side effect_: all the operations that need to work on the internals of the generic types must be implemented in C and make them available to the scala code (see `@extern` in scala native)
    - this is necessary, for example, to implement the distribution layer for (un)marshalling the data structures

- Python examples uses [`cffi`](https://cffi.readthedocs.io/en) to interact with the C API
  - `cffi` support [pyhton code embedding](https://cffi.readthedocs.io/en/stable/embedding.html)
  - `swig` support different languages

Open questions:

1. `cffi` and other FFI libraries require the opaque data types to be defined in the header file (see the [tasks](https://github.com/tassiluca/dap/blob/0b278c4359735f7076d8f9058318b60decdb629a/dap-native-examples/tasks.py#L55) used to build the C API for Python). The transpilation infrastructure should be able to generate the header files enriched with the C concrete types

2. The examples implemented so far uses opaque pointers. The behavior is programmed using simple rewrite rules where the state is generic in `T` and all the rules are defined in terms of `T`. This is a limitation that must be addressed now?
   1. maybe it is necessary to replace opaque pointers with `void*` (and use a type tag to distinguish the actual type of the pointer)

3. For the moment the Python wrapper code have been written manually. Moving towards point 3. of the work plan, is it possible to automate this in the transpilation infrastructure?
    - things to be taken in consideration:
      1. it is very hard to debug
      2. beware garbage collection free memory
      3. the generated code can be idioamtic?
      4. how to document the generated code?

![Architecture flow](./diagrams/architecture-flow.svg)

[^2]: https://faultlore.com/blah/c-isnt-a-language/
