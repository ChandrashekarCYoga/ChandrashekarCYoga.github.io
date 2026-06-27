---
title: "Making Invalid Timer Configuration Fail at Compile Time"
description: "Using strong types and compile-time evaluation to make timer configuration harder to misuse without adding runtime work."
reading_time: "7 min read"
---

A timer can be configured incorrectly and still appear to work.

The code compiles. The firmware flashes. The interrupt fires.

It just fires at the wrong rate.

The idea is simple: give each timer value its own type, calculate the result during compilation, and let invalid configurations fail before the firmware runs.

![From raw timer parameters to compile-time validated configuration](/assets/images/timer-configuration-compact-infographic.png)

*From raw timer parameters to compile-time validated configuration.*

Consider a helper like this:

```cpp
std::uint32_t calculate_reload(
    std::uint32_t timer_clock_hz,
    std::uint32_t prescaler,
    std::uint32_t target_frequency_hz);
```

There is nothing unusual about the interface, but all three arguments use the same type. The compiler therefore accepts both of these calls:

```cpp
calculate_reload(48'000'000, 48, 1'000);
```

```cpp
calculate_reload(1'000, 48, 48'000'000);
```

Only the first call represents the intended configuration.

The integers themselves are not the problem. The problem is that values with different meanings look identical to the compiler.

## Keep the meaning in the type

A small set of domain types can make that meaning explicit:

```cpp
#include <cstdint>
#include <limits>

struct TimerClockHz {
    std::uint32_t value;
};

struct TargetFrequencyHz {
    std::uint32_t value;
};

struct Prescaler {
    std::uint16_t value;
};

struct ReloadValue {
    std::uint32_t value;
};
```

The interface can now describe what each argument represents:

```cpp
consteval ReloadValue calculate_reload(
    TimerClockHz clock,
    Prescaler prescaler,
    TargetFrequencyHz target);
```

And the call site becomes harder to misread:

```cpp
constexpr auto reload =
    calculate_reload(
        TimerClockHz{48'000'000},
        Prescaler{48},
        TargetFrequencyHz{1'000});
```

With this interface, swapping the clock and target frequency is no longer a valid function call.

The values are still stored as integers. The extra types exist to preserve intent and let the compiler check it.

## Do fixed calculations at compile time

In many embedded products, the MCU clock, prescaler, and timer frequency are fixed when the firmware is built.

That makes the reload calculation a good candidate for compile-time evaluation:

```cpp
consteval ReloadValue calculate_reload(
    TimerClockHz clock,
    Prescaler prescaler,
    TargetFrequencyHz target)
{
    if (clock.value == 0) {
        throw "Timer clock must be greater than zero";
    }

    if (prescaler.value == 0) {
        throw "Prescaler must be greater than zero";
    }

    if (target.value == 0) {
        throw "Target frequency must be greater than zero";
    }

    const auto divisor =
        static_cast<std::uint64_t>(prescaler.value) *
        static_cast<std::uint64_t>(target.value);

    if (divisor > clock.value) {
        throw "Requested timer frequency is not achievable";
    }

    const auto counts =
        static_cast<std::uint64_t>(clock.value) / divisor;

    const auto remainder =
        static_cast<std::uint64_t>(clock.value) % divisor;

    if (counts == 0) {
        throw "Calculated timer period is invalid";
    }

    if ((counts - 1U) >
        std::numeric_limits<std::uint32_t>::max()) {
        throw "Reload value does not fit the register type";
    }

    // A production implementation should decide whether a nonzero
    // remainder is acceptable for the required timing tolerance.
    (void)remainder;

    return ReloadValue{
        static_cast<std::uint32_t>(counts - 1U)
    };
}
```

For the example above:

```text
48,000,000 / (48 × 1,000) = 1,000 timer counts
```

If the peripheral register stores the terminal count, the value written to that register would be:

```text
999
```

Because the function is `consteval`, every call must be evaluated during compilation.

If an invalid input reaches one of the failing branches, the call is not a valid constant expression and compilation fails.

The target does not need to perform the calculation or the validation at runtime.

## Does the abstraction really cost nothing?

I would not call this zero overhead without checking the generated code.

A fair comparison should keep the arithmetic and valid-input assumptions equivalent, then build both versions with the same compiler and options.

A raw-integer version:

```cpp
constexpr std::uint32_t raw_reload(
    std::uint32_t clock,
    std::uint32_t prescaler,
    std::uint32_t target)
{
    const auto divisor =
        static_cast<std::uint64_t>(prescaler) *
        static_cast<std::uint64_t>(target);

    const auto counts =
        static_cast<std::uint64_t>(clock) / divisor;

    return static_cast<std::uint32_t>(counts - 1U);
}
```

And the strongly typed version:

```cpp
constexpr auto typed_reload =
    calculate_reload(
        TimerClockHz{48'000'000},
        Prescaler{48},
        TargetFrequencyHz{1'000});
```

I would keep these variables identical across both builds:

- compiler and version,
- C++ language standard,
- optimization level,
- target architecture,
- CPU options,
- and linker settings.

For example:

```bash
arm-none-eabi-g++ \
  -std=c++20 \
  -O2 \
  -mcpu=cortex-m4 \
  -mthumb \
  -ffunction-sections \
  -fdata-sections \
  timer.cpp \
  -S \
  -o timer.s
```

Then I would compare:

1. Generated assembly
2. Flash usage
3. RAM usage
4. Whether runtime division remains
5. The final constant used by the program
6. Diagnostics for invalid input

Since every input is known at compile time, I expect both valid versions to produce the same final constant.

I would still inspect the assembly and size output before making that claim in a production review.

In embedded work, “zero overhead” is something to verify, not something to assume.

## The timer formula is hardware-specific

The calculation above is only an example.

Different timer peripherals use different register models. Some expect:

```text
reload = counts - 1
```

Others use compare registers, auto-reload registers, or separate prescaler and period registers.

A production implementation also needs to consider:

- division that does not produce an exact result,
- the resulting frequency error,
- the width of the timer register,
- minimum and maximum counter values,
- and overflow in intermediate calculations.

The `remainder` in the example is important because integer division can silently truncate. A nonzero remainder does not always mean the configuration is unusable, but it does mean the actual frequency should be calculated and checked against an accepted tolerance.

A reusable timer helper should therefore define:

- the exact peripheral formula,
- the rounding policy,
- the acceptable frequency error,
- the register width,
- and the valid range for the target timer.

Strong types can prevent some interface mistakes, but they do not replace the reference manual.

## One toolchain trade-off

The `throw` statements above are used only to make invalid constant evaluation fail.

That keeps the example small, but it may not fit every embedded build. Some projects compile with exceptions disabled, and some toolchains may reject `throw` even when it appears only in a `consteval` function.

In that environment, I would use a template-based interface with `static_assert`, a constrained factory, or another project-specific compile-time failure mechanism.

The right choice depends on the toolchain and on the quality of the diagnostics it produces.

I would choose the simplest approach that gives the team useful compiler errors and leaves the runtime path unchanged.

## Takeaway

Strong types are useful when several values share the same underlying representation but have different meanings.

Compile-time evaluation is useful when the configuration is fixed before the product runs.

Used together, they can make invalid timer configuration harder to express while keeping the generated firmware simple.

That is the kind of abstraction I find useful in embedded code: it makes the interface harder to misuse, while leaving the target with only the value it needs.
