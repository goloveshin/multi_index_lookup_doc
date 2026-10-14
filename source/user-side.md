# Your own container class

`container_extended_t<...>` is not meant to be used directly in a project. A container usually has a meaning: "aggregates by name", "places by id". The meaning deserves a name, and the code that uses the container should not see Boost types.

This page shows the organization that the library is designed for. It is the recommended way to use it. The full code, which builds and runs, is the `example/` directory of the library.

## The layout

Every container lives in its own namespace and has the same three parts:

1. **`detail`**: the record, the keys and the raw Boost.MultiIndex container.
2. **A class `container_t`** that derives from `container_extended_t<detail::container_t>`.
3. **Methods of that class** that give the lookups names from the domain.

This is `aggregate_container` of the example. It keeps aggregates by name, and one of them is the root:

```cpp
// aggregates by name, including the root one
namespace example::aggregate_container
{

namespace detail
{

struct record_t
{
    std::string name;
    int vd;
    bool is_root;
};

// the key of the index by name: a field extraction
struct by_name
{
    using result_type = const std::string&;
    result_type operator()(const record_t& record) const
    {
        return record.name;
    }
};

struct by_vd{};

using namespace boost::multi_index;

using container_t = multi_index_container
<
    record_t,
    indexed_by
    <
        random_access<>,
        ordered_non_unique
        <
            tag<by_name>,
            by_name
        >,
        ordered_unique
        <
            tag<by_vd>,
            member<record_t, int, &record_t::vd>
        >
    >
>;

}// namespace detail

class container_t : public multi_index_lookup::container_extended_t<detail::container_t>
{
public:
    void add(const std::string& name, const int vd, const bool is_root)
    {
        push_back(detail::record_t{
            .name = name,
            .vd = vd,
            .is_root = is_root
        });
    }

    // ...

    template <typename ... Args>
    decltype(auto) try_find_range_by_name(Args&& ... args) const
    {
        return equal_range_by<detail::by_name>(std::forward<Args>(args)...);
    }
};

}// namespace example::aggregate_container
```

## Why it is arranged this way

### `detail` keeps Boost out of sight

The record, the key extractors and the raw container are in `namespace detail`. The line `using namespace boost::multi_index;` is also there, so the long Boost names are short inside `detail` and never leak to the code that includes the header.

The code that uses `aggregate_container::container_t` never writes `multi_index_container`, `indexed_by` or `tag`.

### The key is a type, and it is also the tag

`by_name` is a function object that takes a record and returns the key. This is called *field extraction*. The same type is passed to `tag<by_name>`, so the type that extracts the key is also the name of the index:

```cpp
ordered_non_unique
<
    tag<by_name>,   // the name of the index
    by_name         // how to get the key from a record
>
```

Later the tag is what you give to the library: `equal_range_by<detail::by_name>(...)`.

A key can be more than a field. `by_name` could return a computed value or a tuple. For the simple case Boost.MultiIndex has the standard `member<...>`; the example uses it for the index `by_vd`, where the tag is an empty struct.

### `container_t` is a class, not an alias

```cpp
class container_t : public multi_index_lookup::container_extended_t<detail::container_t>
```

A class lets you add methods that mean something: `add`, `root`, `rename`. Everything of the Boost container and of `container_extended_t` stays public, so inside the class (and outside) you can still iterate over the records, call `push_back` and so on.

The name `container_t` can be the same in every namespace. The namespace tells the containers apart: `aggregate_container::container_t`, `order_container::container_t`.

### A method gives a lookup its name

```cpp
template <typename ... Args>
decltype(auto) try_find_range_by_name(Args&& ... args) const
{
    return equal_range_by<detail::by_name>(std::forward<Args>(args)...);
}
```

The caller writes `aggregates.try_find_range_by_name(name)`. It does not know about tags. The forwarding keeps the lookup open to every key the index accepts, including tuples for composite keys.

The class can have any other methods. For example, it finds the root aggregate and throws when there is none:

```cpp
const detail::record_t& root() const
{
    for (const auto& record : *this)
    {
        if (record.is_root)
        {
            return record;
        }
    }

    throw std::runtime_error("cannot find root aggregate");
}
```

## A lookup that throws

A method can report a missing record in the way the project wants. The library does not choose the exception type, the project does:

```cpp
const detail::record_t& find_by_vd(const int vd) const
{
    return find_by_or_throw<detail::by_vd>(
        vd,
        [vd]
        {
            throw std::runtime_error("cannot find aggregate " + std::to_string(vd));
        }
    );
}
```

`find_by_or_throw` gets the function that reports the problem, and the function must throw. A project with its own exception type throws it here, with its own information. See [Finding a record](lookup.md).

## A lookup that handles both outcomes

When a missing record is not an error, `find_by_and_do` takes two functions:

```cpp
std::string describe(const int vd) const
{
    return find_by_and_do<detail::by_vd>(
        vd,
        [](const auto&, const auto&, const detail::record_t& record)
        {
            return record.name + (record.is_root ? " (root)" : "");
        },
        [](const auto&)
        {
            return std::string("no such aggregate");
        }
    );
}
```

The same is available for ranges with `find_range_by_and_do`. See [Finding ranges](ranges.md).

## A modification with a rollback

The modification is written once in the class, and the callers pass only what to change. `renumber` changes the key `vd` of a unique index, so the new value may be taken. The rollback function keeps the record in that case:

```cpp
// the new vd may be taken: then the record stays as it was
modify_result_et renumber(const int vd, const int new_vd)
{
    return try_find_by_and_modify<detail::by_vd>(
        vd,
        [new_vd](detail::record_t& record){ record.vd = new_vd; },
        [vd](detail::record_t& record){ record.vd = vd; }
    );
}
```

`modify_result_et` is available in the class without a qualification. See [Modifying a record](modify.md).

## Summary

| Part | Where | What it holds |
| --- | --- | --- |
| record, keys, raw container | `namespace detail` | everything that is Boost.MultiIndex |
| the class `container_t` | the namespace of the container | the public name and the methods of the domain |
| lookup methods | the class | one name per lookup, forwarding to the library |
