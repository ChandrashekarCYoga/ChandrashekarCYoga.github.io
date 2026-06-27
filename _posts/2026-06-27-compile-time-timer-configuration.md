---
title: "Compile-Time Timer Configuration with Strong Types"
description: "Using C++20 strong types and consteval to catch invalid hardware configuration before firmware reaches the target."
reading_time: "7 min read"
---

Raw integers make embedded APIs compact, but they also make it easy to mix units,
swap arguments, or defer validation until runtime.

Consider this interface:

```cpp
void configure_timer(std::uint32_t frequency,
                     std::uint32_t prescaler);
```

It says very little about the meaning of each value.

A stronger model makes the domain visible:

```cpp
#include <cstdint>

struct Hertz {
    std::uint32_t value;
};

struct Prescaler {
    std::uint16_t value;
};

consteval std::uint32_t timer_reload(
    Hertz clock,
    Prescaler divider,
    Hertz target)
{
    if (target.value == 0 || divider.value == 0) {
        throw "Invalid timer configuration";
    }

    return clock.value / (divider.value * target.value);
}

constexpr auto reload =
    timer_reload(Hertz{48'000'000},
                 Prescaler{48},
                 Hertz{1'000});
```

## What changed?

The calculation is now required to succeed during compilation. Invalid constant
configuration cannot silently reach the target.

The generated firmware can still contain only the final reload value. The abstraction
exists primarily for the engineer and compiler—not as additional runtime machinery.

## What I would measure

A credible embedded conclusion should include:

1. Compiler and version
2. Optimization flags
3. Target architecture
4. Generated assembly
5. Text, data, and BSS size
6. Runtime timing where applicable

For example:

```bash
arm-none-eabi-g++ -std=c++20 -O2 -mcpu=cortex-m4 -mthumb   -ffunction-sections -fdata-sections timer.cpp -S -o timer.s
```

## Trade-off

Compile-time APIs can produce difficult diagnostics when templates and constraints become
too elaborate. The design is only successful if it improves correctness without making the
codebase inaccessible to the team.

## Recommendation

Use strong types to express physical meaning, move invariant work to compile time, and
verify the generated result rather than assuming an abstraction is free.
