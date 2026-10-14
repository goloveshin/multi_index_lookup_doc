# Reference

The header is `<multi_index_lookup.hpp>`. Everything is in `namespace multi_index_lookup`.

## `container_extended_t`

```cpp
template <typename MultiIndexContainer>
class container_extended_t : public MultiIndexContainer;
```

A Boost.MultiIndex container plus lookup by index tag. `MultiIndexContainer` is a `boost::multi_index_container`. The class has no data of its own, and the constructors of `MultiIndexContainer` are inherited.

In the tables below `ByTag` is the tag of an index, `Value` is a key (anything the index accepts), and `Index` is `index_by_t<ByTag>`.

### Types

| name | type |
| --- | --- |
| `base_t` | `MultiIndexContainer` |
| `record_t` | the type of the records, `MultiIndexContainer::value_type` |
| `record_crt` | `std::reference_wrapper<const record_t>` |
| `record_ocrt` | `std::optional<record_crt>` |
| `record_rt` | `std::reference_wrapper<record_t>` |
| `record_ort` | `std::optional<record_rt>` |
| `index_by_t<ByTag>` | the type of the index with the tag |
| `iterator_by_t<ByTag>` | the iterator of that index |
| `iterator_by_ot<ByTag>` | `std::optional<iterator_by_t<ByTag>>` |
| `range_by_t<ByTag, Value>` | the range that `equal_range_by` returns for the key type `Value` |
| `modify_result_et` | `multi_index_lookup::modify_result_et`, see below |

The suffixes: `_t` a type, `_et` an enumeration, `_crt` a `std::reference_wrapper` to `const`, `_rt` a `std::reference_wrapper`, `_ot` a `std::optional`, `_ocrt` and `_ort` a `std::optional` of the corresponding wrapper.

### Finding a record

| function | result | a missing record |
| --- | --- | --- |
| `try_find_iterator_by<ByTag>(value)` | `iterator_by_ot<ByTag>` | empty |
| `find_iterator_by_or_throw<ByTag>(value, on_absent)` | `iterator_by_t<ByTag>` | `on_absent()` is called, it must throw |
| `try_find_by<ByTag>(value)` | `record_ocrt` | empty |
| `find_by_or_throw<ByTag>(value, on_absent)` | `const record_t&` | `on_absent()` is called, it must throw |
| `find_by_and_do<ByTag>(value, and_then_func, or_else_func)` | what the called function returns | `or_else_func(index)` |

`and_then_func` is called as `and_then_func(index, iterator, record)`. If `on_absent` returns, `std::bad_optional_access` is thrown.

See [Finding a record](lookup.md).

### Finding ranges

| function | result |
| --- | --- |
| `equal_range_by<ByTag>(value)` | a range of all the records with the key; empty if there are none |
| `range_between<ByTag>(lower, upper)` | a range from `lower_bound(lower)` to `upper_bound(upper)`; `upper` is included |
| `find_range_by_and_do<ByTag>(value, and_then_func, or_else_func)` | what the called function returns |

The ranges are `std::ranges::subrange` of the iterators of the index. `find_range_by_and_do` calls `and_then_func(index, range)` for a non-empty range and `or_else_func(index)` for an empty one.

See [Finding ranges](ranges.md).

### Modifying a record

| function | result |
| --- | --- |
| `try_find_by_and_modify<ByTag>(value, modify_func)` | `modify_result_et` |
| `try_find_by_and_modify<ByTag>(value, modify_func, rollback_func)` | `modify_result_et` |
| `find_by_and_modify<ByTag>(value, on_absent, on_rejected, modify_func)` | `void` |
| `find_by_and_modify<ByTag>(value, on_absent, on_rejected, modify_func, rollback_func)` | `void` |

`modify_func(record)` changes the record. `rollback_func(record)` restores it if the change broke an index. `on_absent()` and `on_rejected()` do not have to throw.

See [Modifying a record](modify.md).

### `to_non_const`

```cpp
static record_t& to_non_const(const record_t&);
static record_rt to_non_const(const record_crt&);
static record_ort to_non_const(const record_ocrt&);
```

Removes `const` from a found record. Change only the fields that no index uses as a key.

## `modify_result_et`

```cpp
enum class modify_result_et { absent, rejected, modified };
```

| value | meaning |
| --- | --- |
| `absent` | there is no record with the key, nothing is changed |
| `rejected` | the change broke an index. Without a rollback function the record is erased. With a rollback function it is restored. |
| `modified` | the record is changed |

## Concepts

The concepts are in `namespace multi_index_lookup::detail`. They check the functions that you pass, so a wrong signature is reported at the call.

| concept | signature of the function | used for |
| --- | --- | --- |
| `can_be_and_then_function` | `(Index&, Index::iterator, const Record&)` | `and_then_func` of `find_by_and_do` |
| `can_be_or_else_function` | `(Index&)` | `or_else_func` of `find_by_and_do` and `find_range_by_and_do` |
| `can_be_range_and_then_function` | `(Index&, Range&)` | `and_then_func` of `find_range_by_and_do` |
| `can_be_modify_function` | `(Record&)` | `modify_func` |
| `can_be_rollback_function` | `(Record&)` | `rollback_func` |
| `can_be_throw_function` | `()` | `on_absent` and `on_rejected` |
