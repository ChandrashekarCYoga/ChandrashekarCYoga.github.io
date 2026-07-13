---
layout: post
title: "C++20 Concepts in Embedded Systems: Better Contracts, No Measured Runtime Cost"
date: 2026-07-10
categories: [modern-cpp, embedded-systems]
tags: [c++20, concepts, embedded-cpp, templates, compile-time]
permalink: /articles/cpp20-concepts-embedded-contracts/
---

Templates are powerful in embedded software because they let us write reusable
code without introducing virtual dispatch or mandatory runtime state.

But an unconstrained template can hide its real assumptions.

A function may appear to accept any device type while quietly requiring that the
type provides:

- `reset()`
- `is_ready() const`
- `transfer(...)`
- a specific transfer result type

Without an explicit contract, those requirements become visible only when
compilation fails.

C++20 Concepts let us name those requirements and make them part of the
interface.

To test the practical effect, I built a small SPI-connected device experiment
and compared:

- compiler diagnostics for invalid device types
- optimized assembly for constrained and unconstrained templates
- executable section sizes and on-disk binary sizes

The measured result was narrow but useful:

> In this experiment, Concepts improved compile-time contract visibility without
> changing the optimized instruction sequence or measured binary size.

That does not mean Concepts prove hardware correctness, timing behavior, DMA
lifetime, or thread safety. They validate the shape of a software interface—not
the physical system behind it.

## The hidden contract in an unconstrained template

Consider a generic initialization function:

```cpp
template<typename Device>
SpiTransferStatus initialize_unconstrained(Device& device)
{
    device.reset();

    if (!device.is_ready())
    {
        return SpiTransferStatus::io_failure;
    }

    constexpr std::array<std::uint8_t, 2> command{0x01, 0x00};
    std::array<std::uint8_t, 2> response{};

    return device.transfer(
        std::span<const std::uint8_t>{command},
        std::span<std::uint8_t>{response});
}
```

At a high level, this function is generic. In practice, it is not unconstrained
at all.

The implementation assumes that `Device` provides:

- `reset()` returning `void`
- `is_ready()` callable on a const object and returning `bool`
- `transfer(...)` accepting transmit and receive spans
- `transfer(...)` returning `SpiTransferStatus`

Those requirements exist whether we write them down or not.

With an unconstrained template, the contract lives inside the implementation.
The compiler discovers it only when a particular expression fails during
instantiation.

That works, but it makes the interface less explicit for both the caller and the
maintainer.

## Making the contract explicit with Concepts

Instead of leaving those requirements hidden inside the function body, we can
name them:

```cpp
template<typename T>
concept Resettable = requires(T& device)
{
    { device.reset() } -> std::same_as<void>;
};

template<typename T>
concept ReadyCheckable = requires(const T& device)
{
    { device.is_ready() } -> std::same_as<bool>;
};

template<typename T>
concept SpiTransferDevice =
    requires(
        T& device,
        std::span<const std::uint8_t> tx,
        std::span<std::uint8_t> rx)
{
    { device.transfer(tx, rx) } -> std::same_as<SpiTransferStatus>;
};

template<typename T>
concept BleSpiDevice =
    Resettable<T> &&
    ReadyCheckable<T> &&
    SpiTransferDevice<T>;
```

The initialization function can now express its dependency directly:

```cpp
template<BleSpiDevice Device>
SpiTransferStatus initialize_ble_device(Device& device)
{
    device.reset();

    if (!device.is_ready())
    {
        return SpiTransferStatus::io_failure;
    }

    constexpr std::array<std::uint8_t, 2> command{0x01, 0x00};
    std::array<std::uint8_t, 2> response{};

    return device.transfer(
        std::span<const std::uint8_t>{command},
        std::span<std::uint8_t>{response});
}
```

The runtime logic is unchanged. The difference is that the capability boundary
is now visible at the function declaration.

That improves readability in two directions:

- callers can see what kind of device is accepted
- maintainers can change the implementation without losing the declared contract

One trade-off to consider is that overly broad Concepts can become vague, while
overly detailed Concepts can couple the interface too closely to one
implementation style. I would lean toward small capability Concepts that can be
composed into a higher-level device contract.

## What the compiler reports when the contract is violated

I tested three invalid device types with GCC 10.2.1 using:

```bash
-std=c++20
-Wall
-Wextra
-Wpedantic
-fsyntax-only
-fconcepts-diagnostics-depth=3
```

### Missing `transfer()`

The compiler reported:

```text
the required expression ‘device.transfer(tx, rx)’ is invalid
```

and then:

```text
‘struct MissingTransferDevice’ has no member named ‘transfer’
```

This is the clearest case. The failed capability is easy to identify, and the
diagnostic points back to the named `SpiTransferDevice` requirement.

### Wrong `transfer()` return type

For a device returning `bool` instead of `SpiTransferStatus`, GCC reported:

```text
‘device.transfer(tx, rx)’ does not satisfy return-type-requirement
```

and eventually showed:

```text
is_same_v<bool, SpiTransferStatus> evaluated to ‘false’
```

This diagnostic was more verbose because GCC expanded the nested
`std::same_as` requirement.

### Non-const `is_ready()`

For a device that provided only:

```cpp
bool is_ready();
```

the compiler reported:

```text
the required expression ‘device.is_ready()’ is invalid
```

and then:

```text
passing ‘const NonConstReadyDevice’ as ‘this’ argument discards qualifiers
```

That result is useful because it shows that the Concept is checking more than a
method name. It also captures const-correctness as part of the interface.

The evidence supports a narrow conclusion:

> Concepts do not guarantee short diagnostics, but they often make the violated
> capability more explicit and move the failure closer to the declared
> interface.

## Did the Concept change the generated code?

To compare runtime impact, I compiled the constrained and unconstrained versions
with GCC 10.2.1 using:

```bash
-std=c++20
-O2
-Wall
-Wextra
-Wpedantic
```

The generated assembly differed only in the source-file directive:

```diff
-.file   "unconstrained.cpp"
+.file   "constrained.cpp"
```

No instruction-level differences were observed.

That supports this narrow result:

> For this source code, compiler version, target, and optimization level, the
> Concept changed compile-time validation but did not change the optimized
> runtime instruction sequence.

## Binary-size comparison

I also compared the Release binaries with `size` and `stat`.

| Variant | Text | Data | BSS | Total | File size |
|---|---:|---:|---:|---:|---:|
| Unconstrained | 1366 | 528 | 8 | 1902 | 16472 bytes |
| Constrained | 1366 | 528 | 8 | 1902 | 16472 bytes |

The constrained and unconstrained variants produced identical values for:

- executable code and read-only data
- initialized writable data
- zero-initialized data
- total measured section size
- on-disk executable size

No binary-size increase was observed in this experiment.

This result should not be generalized beyond the measured configuration.
Concepts do not inherently require runtime storage, but code size can still
change indirectly through overload selection, template instantiation, inlining,
or different generated code paths.

## What Concepts do not prove

The Concept verifies that the required expressions are well-formed and return
the expected types.

It does not prove that the embedded system is correct.

In this example, the compiler cannot verify:

- the configured SPI mode
- the selected clock frequency
- chip-select timing
- peripheral boot time
- transfer deadline compliance
- DMA buffer lifetime
- ISR safety
- thread safety
- power-state sequencing
- electrical connectivity
- protocol correctness

A type can satisfy `BleSpiDevice` and still communicate with the hardware
incorrectly.

This distinction matters in embedded architecture:

> A Concept validates a compile-time software contract. It does not validate the
> temporal, electrical, or physical behavior of the system.

Those concerns still require other engineering mechanisms, such as:

- datasheet-based configuration checks
- timing analysis
- integration tests
- hardware-in-the-loop tests
- watchdog and timeout handling
- concurrency design reviews
- runtime fault detection

## When I would use Concepts in embedded code

I would lean toward Concepts when a template depends on a meaningful set of
capabilities rather than on one concrete type.

They are especially useful for:

- driver and transport abstractions
- test doubles and hardware mocks
- policy-based components
- compile-time selected peripherals
- adapters between HAL, middleware, and application layers

A good Concept should describe what a component must be able to do, not how it
must be implemented.

For this example, `BleSpiDevice` communicates the required capability boundary
without introducing virtual dispatch, runtime state, or measured code-size
growth.

## Experiment source

The complete source code, raw GCC diagnostics, generated assembly, and binary
size measurements are available in the companion repository:

[modern-embedded-cpp-experiments — concepts-spi-device](https://github.com/ChandrashekarCYoga/modern-embedded-cpp-experiments/tree/main/concepts-spi-device)

## Final takeaway

The most valuable part of Concepts in this experiment was not shorter syntax.

It was making an implicit dependency explicit.

The compiler could then explain failures in terms of the declared capability,
while the optimized runtime code and measured binary size remained unchanged.

Concepts strengthen the software boundary. They do not replace timing analysis,
hardware validation, integration testing, or fault handling—but they can make
the interface between those concerns much clearer.

Where have Concepts improved an interface in your embedded code, and where did
they make the design more complicated?