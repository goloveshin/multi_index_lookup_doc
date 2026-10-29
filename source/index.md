# Looking up records in a Boost.MultiIndex container

## The problem

[Boost.MultiIndex](https://www.boost.org/doc/libs/release/libs/multi_index/doc/index.html) keeps records in one container and gives you several indices over them: ordered, hashed, random access. Each index has a tag, a type that names it.

To use an index you get it by its tag, search, and compare the result with `end()`:

```cpp
const auto& index = container.get<by_name>();

if (const auto it = index.find("user"); it != index.end())
{
    // found: use *it
}
else
{
    // not found
}
```

This is correct, but every lookup repeats the same steps, and a missing record is handled in a different way in every place. A project with many containers ends up with many copies of the same few lines.

Changing a record has its own trap. `index.modify(it, modifier)` returns `false` if the changed record breaks a unique index, and then **the record is erased from the container**. Nothing in the call says so.

## What the library does

`multi_index_lookup::container_extended_t` is your Boost.MultiIndex container plus a few lookup functions. You inherit from it, so everything the container could do stays available.

```cpp
// by tag, a missing record is an empty optional
const auto found = container.try_find_by<by_name>("user");

// by tag, a missing record calls the function you pass
const auto& record = container.find_by_or_throw<by_name>("user", on_absent);

// all records with the key
for (const auto& record : container.equal_range_by<by_name>("user")) { /* ... */ }

// change a record and learn what happened
const auto result = container.try_find_by_and_modify<by_name>("user", modifier);

// erase one record with the key
const bool is_erased = container.try_erase_by<by_name>("user");
```

What you get:

- **One way to ask.** Lookups take the tag as a template argument and return `std::optional`, a reference, or a range.
- **A choice of what a missing record means.** An empty result, a function that throws, or a pair of functions for "found" and "not found".
- **Ranges.** `std::ranges::subrange` of the index iterators, for one key or for a span of keys.
- **An honest modify.** The result says whether the record was absent, rejected or modified. With a rollback function a rejected record is not lost.
- **Erase by key.** `try_erase_by` erases one record with the key and returns whether a record was erased.

It is a thin layer. It adds no data and no virtual functions, and it does not change how the container stores records.

## How this documentation is arranged

1. [Getting started](getting-started.md): requirements, CMake and a first program.
2. [Your own container class](user-side.md): how the library is used in a project. A container becomes a class with its own name and its own methods, and Boost types stay hidden in `detail`.3. The guide: [finding a record](lookup.md), [finding ranges](ranges.md) and [modifying a record](modify.md).
4. [Reference](reference.md): every function, type and concept.
