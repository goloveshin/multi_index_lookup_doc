# Getting started

## Requirements

- C++23. The library uses `std::optional::transform` and `std::ranges::subrange`.
- Boost.MultiIndex. If the target `Boost::multi_index` does not exist when the library is configured, the library fetches Boost itself, limited to MultiIndex.

The library is header-only. The only target is the `INTERFACE` library `multi_index_lookup::multi_index_lookup`.

## CMake

The library lives at [github.com/goloveshin/multi_index_lookup](https://github.com/goloveshin/multi_index_lookup). Fetch it with `FetchContent`:

```cmake
include(FetchContent)

FetchContent_Declare(
    multi_index_lookup
    GIT_REPOSITORY https://github.com/goloveshin/multi_index_lookup.git
    GIT_TAG main
)
FetchContent_MakeAvailable(multi_index_lookup)

target_link_libraries(my_app PRIVATE multi_index_lookup::multi_index_lookup)
```

Replace `main` with a release tag to pin a version. To use a local copy instead, replace `GIT_REPOSITORY` and `GIT_TAG` with `SOURCE_DIR <path>`.

The library has an option `MULTI_INDEX_LOOKUP_BUILD_TESTS`. It builds the unit tests and the example of the library. It is `ON` only when `multi_index_lookup` is the top-level project, so a project that uses the library does not build them. You can set it yourself before `FetchContent_MakeAvailable`:

```cmake
set(MULTI_INDEX_LOOKUP_BUILD_TESTS OFF)
```

If your project already provides Boost with MultiIndex, define the target `Boost::multi_index` before `FetchContent_MakeAvailable` and the library does not fetch another copy.

## The first program

```cpp
#include <multi_index_lookup.hpp>

#include <boost/multi_index/member.hpp>
#include <boost/multi_index/ordered_index.hpp>
#include <boost/multi_index_container.hpp>

#include <iostream>
#include <stdexcept>
#include <string>

namespace bmi = boost::multi_index;


struct record_t
{
    int id;
    std::string name;
};

struct by_id{};

using records_t = bmi::multi_index_container
<
    record_t,
    bmi::indexed_by
    <
        bmi::ordered_unique
        <
            bmi::tag<by_id>,
            bmi::member<record_t, int, &record_t::id>
        >
    >
>;

using container_t = multi_index_lookup::container_extended_t<records_t>;

int main()
{
    const container_t container{
        record_t{ 1, "one" },
        record_t{ 2, "two" }
    };

    // a missing record is an empty optional
    if (const auto found = container.try_find_by<by_id>(2))
    {
        std::cout << "found: " << found->get().name << "\n";
    }

    if (!container.try_find_by<by_id>(9))
    {
        std::cout << "9 is absent\n";
    }

    // a missing record calls the function, which must throw
    try
    {
        container.find_by_or_throw<by_id>(9, []{ throw std::runtime_error("no record 9"); });
    }
    catch (const std::runtime_error& error)
    {
        std::cout << "error: " << error.what() << "\n";
    }
}
```

It prints:

```text
found: two
9 is absent
error: no record 9
```

What happens here:

1. `records_t` is an ordinary Boost.MultiIndex container with one index. The struct `by_id` is its tag.
2. `container_extended_t<records_t>` is the same container with the lookup functions added. The constructors of `records_t` are inherited, so the container can be built from a list of records.
3. `try_find_by<by_id>(2)` returns `std::optional<std::reference_wrapper<const record_t>>`. Use `->get()` to reach the record.
4. `find_by_or_throw<by_id>(9, ...)` returns the record, or calls the function when there is none. The function must throw; see [Finding a record](lookup.md).

In a real project you do not use `container_extended_t<...>` directly. You wrap it in a class of your own: see [Your own container class](user-side.md).
