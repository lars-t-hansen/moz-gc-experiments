Work in progress.

Version 2 will extend Version 1, ideally in a compatible fashion.  Here's what's going on.

## Generalized tables

We'll do this in increments, as follows.

### Tables-of-anyref

A table can now be `anyref` in addition to `anyfunc`.  The code for `anyref` is 0x6F, its standard type code.

Denote table-of-anyref as `T(anyref)` and table-of-anyfunc as `T(anyfunc)`.

`anyref` can be used in the wasm text format for the type of the table.

`"anyref"` can be used as the type name passed to the JS `WebAssembly.Table` constructor.

Setting elements in a `T(anyref)` from JS will store JS objects.  If the value being set is not already an object we perform the same coercion as we do on a function boundary when JS passes a non-object to a wasm function that takes an anyref.  (Currently this is a ToObject operation but this is subject to change.)

Element segments are *not* further extended, they can reference only function values.

We can `table.copy` only between tables of the same type.

`table.init` can only init `T(anyfunc)` since the source in this case is an elem segment which can only reference function values.

`call_indirect` requires `T(anyfunc)`.

`(table.get index)` can target only `T(anyref)`, the result is `anyref`.

`(table.set index value)` can target only `T(anyref)` and the static type of the value must be some `ref` type.

### Multiple tables

There can be several tables, with indices starting at zero.  As usual(?), imports are numbered before local tables.

In the text format, tables can be named, `(table $t length type)`.

An active element segment still can only target one specific table, since the table index is baked into the element.  But it can target any of the tables in the module.  Presumably there is opcode space for that at the moment.

The table index is always baked into the instruction.  In the text format, the table index is optional for backwards compatibility reasons; this is not a problem (yet), provided the table index follows existing constant indices.

Fully parenthesized syntax, here "type" and "table" can be literal indices or names in the appropriate namespaces:

```
(call_indirect type table argument-expr ...)
(call_indirect type argument-expr ...)
(table.get element-index-expr)
(table.get table-index element-index-expr)
(table.set element-index-expr value-expr)
(table.set table-index element-index-expr value-expr)
// todo: table.init
// todo: table.copy
// todo: table.fill
```

In the RPN text format, if the instructions carry table indices then they follow the opcodes in the same
way we now handle blahblah, eg,

```
element-index-expr
table.get table-index

argument-expr
...
call_indirect type-index table-index
```

(TODO: it's possible we need to parenthesize the multi-word operators, investigate.)

(TODO: it's possible we want to support disambiguating syntax, (type T) and (table T) to be used in these instructions, optionally.)

### anyfunc as value type, and fallout from that

(Work in progress)

We introduce a new value type `anyfunc`, where `anyfunc` <: `anyref`.

In every text context, `funcref` has the same meaning as `anyfunc`, and probably `funcref` should be the canonical name by and by.

We can `table.copy` from table T1 to table T2 if the element type of T1 is a subtype of the element type of T2.  Specifically, we can `table.copy` from `T(anyfunc)` to `T(anyref)`.  TODO: under what conditions?  Exported functions?

We can `table.init` a `T(anyref)` since anyfunc <: anyref.  TODO: under what conditions?  Exported functions?

(Eventually)  `T(anyref)` can be targeted by element segments holding function values,  and the values stored in such tables are the function values that would be obtained from the host side if the host side reached into a corresponding anyfunc table and extracted functions.

When `table.get` targets a `T(anyfunc)` what do we return?  probably a function object.  But under what restrictions?  Exported function?

this is fairly obvious; value must be anyref.  on `T(anyfunc)` the static type must be anyfunc.

TODO: no downcast function at this point, so if we have an `anyref` we can't cast it dynamically to `anyfunc` or test whether we could do that safely.  

### nullref

Change the current `(ref.null T)` construction into `ref.null` and introduce the `nullref` type.  For backward compatibility, continue to accept the old encoding?  Tricky, because then we'll need a new opcode.

### Tentative cleanup, other

Rename or create alias for `ref.is_null` as `ref.isnull` (as spec requires).  Superficial text format change.
