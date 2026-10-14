# Finding ranges

An index that is not unique can have many records with one key. An ordered index also keeps the records sorted, so records with keys from a span of values are next to each other. The library gives both as a range.

A range is `std::ranges::subrange` of the iterators of the index. It does not copy the records, and an empty range means that nothing was found.

## `equal_range_by`

All the records with the key:

```cpp
for (const auto& record : container.equal_range_by<by_name>("user"))
{
    // every aggregate named "user"
}
```

If there are no such records, the range is empty. There is no "absent" state, so the function does not start with `try_`.

The type of the range is `container_t::range_by_t<by_name, std::string>` for the key type `std::string`. It is the type to use in a signature when you need to write it:

```cpp
using range_t = container_t::range_by_t<by_name, std::string>;
```

## `range_between`

The records with the keys from `lower` to `upper`, **both included**:

```cpp
// the records of the index by_vd with the vd 1, 2 and 3
const auto range = container.range_between<by_vd>(1, 3);
```

It works from `lower_bound(lower)` to `upper_bound(upper)` of the index. If no key is between, or if `lower` is greater than `upper`, the range is empty.

For a composite key (`composite_key` of Boost.MultiIndex) the bounds can be partial tuples. This is how a range of "all the records of this parent" is taken.

## `find_range_by_and_do`

Like [`find_by_and_do`](lookup.md#find_by_and_do), for a range:

```cpp
const std::size_t count = container.find_range_by_and_do<by_name>(
    "user",
    // the range is not empty
    [](const auto& index, auto& range)
    {
        return static_cast<std::size_t>(std::ranges::distance(range));
    },
    // the range is empty
    [](const auto& index)
    {
        return std::size_t{ 0 };
    }
);
```

- `and_then_func(index, range)` is called when the range is not empty. `range` is a non-const lvalue.
- `or_else_func(index)` is called when it is empty.

Both functions must return the same type, and it is the result of `find_range_by_and_do`.

## Ranges and algorithms

The result is a standard range, so it works with the standard library and with range libraries:

```cpp
const auto range = container.equal_range_by<by_name>("user");

const bool is_any = !range.empty();
const auto count = std::ranges::distance(range);
```

!!! note
    A range is a view. It refers to the container, so do not keep it after the container changes. Erasing or modifying a record that the range covers can make the range invalid.
