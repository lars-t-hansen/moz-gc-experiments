(Work in progress.)

## Version opt-in

Version 2 will extend Version 1, ideally in a compatible fashion.  Here's what's going on.

The module must declare `(gc_feature_opt_in 2)` to use any new table facility that mentions or makes use of `anyref`; this includes most new instructions, which require `anyref` at the moment.  Multi-table functionality that does not use `anyref` (eg having multiple function tables) does not require opting in.

At this time, existing version 1 content continues to work in systems that support v2.  This will change when we implement full `nullref` semantics, probably.

## Tables-of-anyref + table manipulation (up for review per Nov 6)

A table can now be of type `anyref` in addition to type `anyfunc`.  In the following, denote table-of-anyref as `T(anyref)` and table-of-anyfunc as `T(anyfunc)`.

Element segments are *not* further extended, they can reference only function values.

Encoding of `T(anyref)`:

* The code for `anyref` is 0x6F, its standard type code.
* `anyref` can be used in the wasm text format for the type of the table.
* `"anyref"` can be used as the element name in the descriptor passed to the JS `WebAssembly.Table` constructor.

### Wasm instructions that can't operate on table-of-anyref

`table.init` can only be used to init `T(anyfunc)` since the source in this case is an elem segment which can only reference function values.

`call_indirect` requires `T(anyfunc)`.

### New wasm instructions for table manipulation

#### table.get

`(table.get index)` can target only `T(anyref)`, the result is `anyref`.

Encoding: (0xFC 0x10 0x00) where the last byte is a flags byte that will eventually accomodate a table index.  Traps on OOB with RangeError.

#### table.set

`(table.set index value)` can target only `T(anyref)` and the static type of the value must be some `ref` type; the result is void.  Traps on OOB with RangeError.

Encoding: (0xFC 0x11 0x00) where the last byte is a flags byte that will eventually accomodate a table index

#### table.grow

`(table.grow delta init-value)` can target only `T(anyref)`.  It exposes the existing JS-level mechanism to wasm and lets even non-exported tables grow. The argument is i32.  The result is i32, the old size of the table; the result will appear negative for memories > 2GB.  Traps on negative arguments with RangeError, so the maximum growth increment is 2GB; not ideal.  Traps with RuntimeError if the grow fails.

The `init-value` is the value to fill the new slots with, it must be a subtype of anyref.  (The reason `table.grow` can target only `T(anyref)` is that we don't have a notion of a function value yet.)

Encoding: (0xFC 0x0F 0x00) where the last byte is a flags byte that will eventually accomodate a table index

#### table.size

`(table.size)` returns the size of the table as an i32.

#### table.fill

TBD - not yet implemented

### Changed JS API

Setting elements in a `T(anyref)` with `WebAssembly.Table.prototype.set(value)` from JS will store JS objects.  If the value being set is not already an object we perform the same coercion as we do on a function boundary when JS passes a non-object to a wasm function that takes an anyref.  (Currently this is a ToObject operation but this is subject to change.)

## Multiple tables (up for review per Nov 6)

There can be several tables, with indices starting at zero.  As usual, imports are numbered before local tables.

In the text format, tables can be named, `(table $tablename length type)`.

An active element segment still can only target one specific table, since the table index is baked into the element.  But it can target any of the tables in the module: `(elem $tablename init function ...)`.  A passive segment cannot carry a table index.

Encoding: The reserved flags byte in the instruction has bit 0x04 set, in which case there is a table index immediately following (two indices, in the case of `table.copy`).  This encoding will change for sure.

The table index is always baked into the wasm instructions.  In the text format, the table index is optional for backwards compatibility reasons.

Fully parenthesized syntax:

```
(call_indirect table-ref type-ref argument-expr ...)
(call_indirect type-ref argument-expr ...)
(table.get element-index-expr)
(table.get table-ref element-index-expr)
(table.set element-index-expr value-expr)
(table.set table-ref element-index-expr value-expr)
(table.init dest-table-ref src-segment-ref dest-index-expr src-index-expr len-expr)
(table.init src-segment-ref dest-index-expr src-index-expr len-expr)
(table.copy dest-table-ref dest-index-expr src-table-ref src-index-expr len-expr)
(table.copy dest-index-expr src-index-expr len-expr)
(table.grow table-ref delta-expr init-value-expr)
(table.grow delta-expr init-value-expr)
(table.size table-ref)
(table.size)
;; table.fill TBD
```

In the RPN text format, if the instructions carry table indices then they follow the opcodes in the same
way we now handle blahblah, eg,

```
element-index-expr
table.get table-index

argument-expr
...
call_indirect type-index table-index

dest-index-expr
src-index-expr
len-expr
table.copy dest-table-index src-table-index
```

(TODO: it's possible we need to parenthesize the multi-word operators, investigate.)

(TODO: it's possible we want to support disambiguating syntax, (type T) and (table T), or type=T and table=T / src=T, dest=T to be used in these instructions, optionally.)

We can `table.copy` only between tables of the same type.

## anyfunc as value type, and fallout from that

(Work in progress)

We introduce a new value type `anyfunc`, where `anyfunc` <: `anyref`.

In every text context, `funcref` has the same meaning as `anyfunc`, and probably `funcref` should be the canonical name by and by.

We can `table.copy` from table T1 to table T2 if the element type of T1 is a subtype of the element type of T2.  Specifically, we can `table.copy` from `T(anyfunc)` to `T(anyref)`.  TODO: under what conditions?  Exported functions?

We can `table.init` a `T(anyref)` since anyfunc <: anyref.  TODO: under what conditions?  Exported functions?

(Eventually)  `T(anyref)` can be targeted by element segments holding function values,  and the values stored in such tables are the function values that would be obtained from the host side if the host side reached into a corresponding anyfunc table and extracted functions.

When `table.get` targets a `T(anyfunc)` what do we return?  probably a function object.  But under what restrictions?  Exported function?

this is fairly obvious; value must be anyref.  on `T(anyfunc)` the static type must be anyfunc.

TODO: no downcast function at this point, so if we have an `anyref` we can't cast it dynamically to `anyfunc` or test whether we could do that safely.  

## nullref

Change the current `(ref.null T)` construction into `ref.null` and introduce the `nullref` type.  For backward compatibility, continue to accept the old encoding?  Tricky, because then we'll need a new opcode.

## Tentative cleanup, other

Rename or create alias for `ref.is_null` as `ref.isnull` (as spec requires).  Superficial text format change.

## Currently out of scope for Version 2

### table-of-ref

Along with anyfunc and table-of-anyref we can extend this to table-of-ref in general, with fairly obvious semantics.  This is slightly tricky though because this entails either (a) those tables are not exported or (b) we need to talk about imported / exported types.  We can do (a) for V2 easily, however.

