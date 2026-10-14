# Finding a record

All the examples use the container of `example/aggregate_container.hpp`: a container of aggregates with the index `by_name` (not unique) and the index `by_vd` (unique).

Every lookup takes the tag of the index as a template argument and the key as the first argument:

```cpp
container.try_find_by<by_vd>(3);
```

The key is passed to `find` of that index, so it can be anything the index accepts.

## What a missing record means

The functions differ in what they do when there is no record. The name tells which:

| name | a missing record |
| --- | --- |
| `try_find_*` | the result is an empty `std::optional` |
| `find_*_or_throw` | the function you pass is called, and it must throw |
| `find_*_and_do` | you pass two functions: one for found, one for not found |

## `try_find_by`

```cpp
const auto found = container.try_find_by<by_vd>(3);

if (found)
{
    const auto& record = found->get();
}
```

It returns `std::optional<std::reference_wrapper<const record_t>>`. The type is also available as `container_t::record_ocrt`. The reference points into the container, there is no copy.

If the index is not unique, the result is one of the records with the key. To get all of them use [`equal_range_by`](ranges.md).

## `try_find_iterator_by`

```cpp
const auto found = container.try_find_iterator_by<by_vd>(3);
```

The same, but the result is `std::optional` of the iterator of the index `by_vd`. Use it when you need the position, not only the record: for example to erase the record or to pass it to `modify`.

## `find_by_or_throw`

```cpp
const auto& record = container.find_by_or_throw<by_vd>(
    3,
    [&]{ throw std::runtime_error("no aggregate 3"); }
);
```

It returns `const record_t&`. If there is no record, the function is called.

!!! warning
    The function **must throw**. If it returns, the library throws `std::bad_optional_access`, because it has no record to return. The library does not choose the exception for you, so a project can throw its own type, with its own information.

`find_iterator_by_or_throw` does the same and returns the iterator.

## `find_by_and_do`

```cpp
const std::string text = container.find_by_and_do<by_vd>(
    3,
    // found
    [](const auto& index, const auto& iterator, const record_t& record)
    {
        return record.name;
    },
    // not found
    [](const auto& index)
    {
        return std::string("no such aggregate");
    }
);
```

It calls one of the two functions and returns what it returned.

- `and_then_func(index, iterator, record)` is called when the record is found. `index` is the index chosen by the tag, `iterator` points to the record, `record` is the record.
- `or_else_func(index)` is called when it is not found.

Both functions must return the same type. That type is the result of `find_by_and_do`. Both can return `void`.

The names are the same as `and_then` and `or_else` of `std::optional`.

## `to_non_const`

The records of a Boost.MultiIndex container are `const`. `to_non_const` removes `const` from a record that you found:

```cpp
auto& record = container_t::to_non_const(container.find_by_or_throw<by_vd>(3, on_absent));
record.is_root = false;
```

It has three forms: for `const record_t&`, for `std::reference_wrapper<const record_t>`, and for the `std::optional` of it that `try_find_by` returns.

!!! warning
    Change only the fields that no index uses as a key. If you change a key, the index is not updated and it breaks. To change a key use [`try_find_by_and_modify`](modify.md).
