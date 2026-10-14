# Modifying a record

The records of a Boost.MultiIndex container are `const`, because a key that changes under an index would break it. To change a record you ask the index to do it: `index.modify(iterator, modifier)`. The index takes the record out of its position, calls the modifier and puts the record back where the new keys say.

The library finds the record by the key and does this for you, and reports what happened.

## What can go wrong

Changing a record can make it clash with another one. For example, a unique index already has a record with the new key. Boost.MultiIndex cannot keep both, so `modify` returns `false`.

What happens to the record depends on the form of `modify`:

| call | on a clash |
| --- | --- |
| `modify(it, modifier)` | **the record is erased** from the container |
| `modify(it, modifier, rollback)` | `rollback` is called to restore the record, and the record stays |

If the modifier itself throws, Boost.MultiIndex erases the record and the exception goes on.

The library has both forms.

## `try_find_by_and_modify`

```cpp
const auto result = container.try_find_by_and_modify<by_vd>(
    3,
    [](record_t& record){ record.name = "orders"; }
);
```

It finds the record by the key and calls the modifier with the record. The result is `modify_result_et`:

| result | meaning |
| --- | --- |
| `absent` | there is no record with the key, nothing is changed |
| `modified` | the record is changed |
| `rejected` | the change broke an index, see below |

The enumeration is `multi_index_lookup::modify_result_et`. It is also available in the class as `container_t::modify_result_et`, so a class that derives from `container_extended_t` can write `modify_result_et::absent`.

!!! warning
    With this form, `rejected` means that **the record is gone**. Boost.MultiIndex erased it. If that is not acceptable, use the form with the rollback function.

## With a rollback function

```cpp
const auto result = container.try_find_by_and_modify<by_vd>(
    3,
    // the change
    [](record_t& record){ record.vd = 4; },
    // return the record to the state before the change
    [](record_t& record){ record.vd = 3; }
);
```

If the new `vd` is already taken, the rollback function returns the record to `vd` 3, and the record stays in the container. The result is `rejected`: the change was refused, and nothing is lost.

The rollback function must really restore the record. If the indices are still broken after it, Boost.MultiIndex erases the record, as in the form without a rollback.

The rollback function is called only for `rejected`. It is not called for `absent` and `modified`.

## `find_by_and_modify`

Calls a function for each outcome that is not `modified`:

```cpp
container.find_by_and_modify<by_vd>(
    3,
    // on_absent
    [&]{ throw std::runtime_error("no aggregate 3"); },
    // on_rejected
    [&]{ throw std::runtime_error("cannot change aggregate 3"); },
    // the change
    [](record_t& record){ record.vd = 4; }
);
```

- `on_absent()` is called when there is no record.
- `on_rejected()` is called when the change was rejected.
- Nothing is called when the record is modified.

Unlike `find_by_or_throw`, these functions do not have to throw. If they return, `find_by_and_modify` returns, and the caller can look at its own state.

There is an overload with the rollback function as the last argument:

```cpp
container.find_by_and_modify<by_vd>(
    3,
    on_absent,
    on_rejected,
    [](record_t& record){ record.vd = 4; },
    [](record_t& record){ record.vd = 3; }
);
```

With it, a rejected change leaves the record in the container, and `on_rejected` is called after the rollback.

## Which form to use

- If the changed fields are not keys of a unique index, a change cannot be rejected. The simple form is enough.
- If a change can clash with another record and losing the record is a bug, use a rollback function.
- If a clash means that the record must go, the form without a rollback does exactly that, and `rejected` tells you that it happened.

## Changing a field that is not a key

When no index uses the field as a key, you do not need `modify`. See `to_non_const` in [Finding a record](lookup.md#to_non_const).
