# Refactoring Meshtastic Firmware from Inheritance to Composition

- Start Date: 2023-11-13
- RFC PR: [Meshtastic/rfcs#0000](https://github.com/Meshtastic/rfcs/pull/0000)
- Affected Components: Firmware

## Summary

The Meshtastic firmware currently heavily utilizes an inheritance pattern of OOP. This RFC proposes to migrate the firmware from this inheritance pattern to a composition pattern. Perceived benefits of this approach can be found in the [Motivation](#motivation) section.

Examples of inheritance being used within the Meshtastic firmware are listed below:

- <https://github.com/meshtastic/firmware/blob/8b16367597cad303df923690cd9916d7b2dc646f/src/mesh/api/ethServerAPI.h#L10>
- <https://github.com/meshtastic/firmware/blob/8b16367597cad303df923690cd9916d7b2dc646f/src/graphics/Screen.h#L109>
- <https://github.com/meshtastic/firmware/blob/8b16367597cad303df923690cd9916d7b2dc646f/src/mesh/RadioLibRF95.h#L9>

One point to note is that C++ doesn't have an explicit `interface` keyword, meaning contracts are declared as shown below:

- [contract] <https://github.com/meshtastic/firmware/blob/8b16367597cad303df923690cd9916d7b2dc646f/src/Status.h#L15>
- [implementation] <https://github.com/meshtastic/firmware/blob/8b16367597cad303df923690cd9916d7b2dc646f/src/GPSStatus.h#L13>

In this context, inheritance is where a superclass contains methods from a subclass, which is not the case in the two lines directly above. An example of this can be found [here](https://stackoverflow.com/questions/49890491/a-request-for-simple-c-composition-vs-inheritance-examples).

## Motivation

The primary motivation of this RFC is to enable a migration of the Meshtastic firmware from C++ to Rust. This would come with the additional benefits listed below. The resulting incremental refactor would also give an opportunity to write unit and integration testing natively into the firmware codebase.

At a high level, this migration would have the following benefits to the project:

1. **Allowing for an incremental migration of the firmware repo to Rust**
1. Allows for testing of firmware components in isolation
1. Removes tight code coupling, allowing for simpler refactoring
1. Improves code comprehensibility due to removal of "hidden" methods defined in base classes

### Background

The [Rust programming language](https://en.wikipedia.org/wiki/Rust_(programming_language)) is a general-purpose language that features total, non-GC memory safety. Rust writes like a high-level language, but can be used anywhere from embedded systems to web applications. Rust enforces memory safety through a compiler feature known as the borrow checker," which guarantees that all references will always point to valid data. Additionally, Rust replaces `NULL` with the [Option\<T\>](https://doc.rust-lang.org/std/option/) type, which removes the possibility of null pointer-style exceptions.

Rust is a [well-loved language](https://survey.stackoverflow.co/2023/#section-admired-and-desired-programming-scripting-and-markup-languages), and is rapidly growing in popularity in the [embedded programming space](https://github.com/rust-embedded/awesome-embedded-rust). Rust allows for complete memory safety even on embedded hardware, which saves developer time and effort while also producing more effective programs.

### Why Does Rust Require Composition?

While the Rust language is [considered object-oriented](https://doc.rust-lang.org/book/ch17-00-oop.html), it **does not** support inheritance. Object-oriented Rust programs are based on interface-style contracts known as [Traits](https://doc.rust-lang.org/book/ch10-02-traits.html), which objects ([structs](https://doc.rust-lang.org/book/ch05-01-defining-structs.html)) implement. I've attached a short video that I find explains this paradigm well [here](https://www.youtube.com/watch?v=z-0-bbc80JM&ab_channel=NoBoilerplate).

Because of the fact that Rust only supports composition, it is not possible to incrementally migrate any portions of the firmware codebase that rely on inheritance to Rust. Currently, as an example, this prevents incremental migration of the internal software module system.

## Ecosystem Impact

This change is not intended to create any public API changes within the Meshtastic ecosystem, and any change will either be documented separately or will be considered a regression.

## Protocol Buffer Changes

This change does not require any protocol buffer changes.

## Technical Details

[Provide a more in-depth technical explanation of the proposed changes, focusing on the high-level architecture and how different components of the ecosystem will interact with these changes. This section should explain your proposed solution in enough detail that someone familiar with the Meshtastic ecosystem can understand the design and implementation of the feature.]

The technical requirements of this RFC are very broad in their scope but not easily described in their entirety. While the general refactoring process will be fairly similar to what's described below, it will require intentional decicions as to how much to expose to higher levels of abstraction. The following process can be applied to a component to determine whether it needs to be refactored.

1. Identify whether the component inherets from another class (i.e., whether it is declared in its header file in the form `class ClassName : OtherClassName { ... }`). If the class is not declared in this form, it does not need to be refactored.
1. Identify whether `OtherClassName` has any default implementations (either non-`virtual` or `virtual` but with implementation). If no methods have default implementations, then this class is simulating a Java-style interface and doesn't need to be refactored.
1. This class needs to be refactored from an inheritance-based pattern to a composition-based pattern.

If a class needs to be refactored, apply the following considerations:

- Limit methods that are exposed to higher-level abstractions. Currently, many sub-classes expose a significant number of methods and member variables due to deeply-nested inheritance. Start by assuming that no child methods and members are needed by higher-level abstractions, then add pass-through methods as necessary.
- Check whether multiple child method or members being exposed to higher-level abstractions can be combined into a single method or member variable. This will both simplify the high-level API and simplify the refactoring process.

### Example

In the example below, we can see a _very_ basic example of how to refactor the `ethServerAPI` class, which inherets from the `ServerAPI` template class.

```cpp
// Current implementation

class ethServerAPI : public ServerAPI<EthernetClient>
{
  public:
    explicit ethServerAPI(EthernetClient &_client);
};
```

```cpp
// Proposed implementation

class ethServerAPI
{
  public:
    explicit ethServerAPI(EthernetClient &_client) : serverAPI(_client) {}

    // Add methods that call the corresponding methods on serverAPI
    // Note that this would only be necessary when this functionality
    // is needed at higher levels of abstraction. Otherwise, this functionality
    // could be contained in methods that expose only what's necessary.
    void close() {
        serverAPI.close();
    }

    // Repeat for other methods as necessary

  private:
    ServerAPI<EthernetClient> serverAPI;
};
```

### Compatibility Considerations

This change will not impact backwards compatibility within the system, and as such will not require a version bump.

### Security Considerations

This change has the potential to introduce regressions into the firmware, which have the potential to affect project security.

### Performance Considerations

Composition does not inherently have performance impacts in C++.

## Drawbacks

Potential downsides of this migration are discussed below:

- One major drawback with composition-based OOP is the fact that methods can't be directly shared between components. This has the potential to lead to an increase in boilerplate or repetitive code. This will be a larger problem in C++ where interfaces can't implement default methods, but will not be as much of an issue after a potential Rust migration. This is because Rust traits support default implementations of methods.
- Refactoring large portions of the existing codebase introduces a large risk of functionality regression. This is a particular problem since the current firmware codebase does not have any test coverage. While regression does present a relatively large risk in the short term, in the long term a migration towards more testable software patterns will allow and give the opportunity for robust code testing.

## Rationale and Alternatives

- Why is this approach the best option?
- What other solutions were considered, and why were they not chosen?
- What would be the impact of not implementing this change?

## Prior Art

[Discuss any similar implementations in other projects or technologies and what can be learned from them. Include both successful and unsuccessful examples.]

None to speak of.

## Unresolved Questions

- How much of a shift in developer coding patterns would this take?
- Is the community interested in incrementally migrating to Rust?
