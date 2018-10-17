Work in progress.

Version 2 will extend Version 1, ideally in a compatible fashion.  Here's what's going on.

### Tables-of-anyref

Type calculus is extended with `anyfunc` <: `anyref`.

Table can now be `anyref` in addition to `anyfunc`.  The code for `anyref` is 0x6F, its standard type code.

Below, denote table-of-anyref as `T(anyref)` and table-of-anyfunc as `T(anyfunc)`.

`anyref` can be used in the wasm text format for the type of the table.

`anyref` can be used as the type name passed to the JS WebAssembly.Table constructor.

Setting elements in a `T(anyref)` from JS will store JS objects.  TODO: What if the value being set is not an object?  Run ToObject on it a la what the stub does on the call boundary?

Element segments are *not* further extended, they can reference only function values.

We can `table.copy` only between tables of the same type (TODO: this is too strict since anyfunc <: anyref).

`table.init` can only init `T(anyfunc)` since the source in this case is an elem segment which can only reference function values. (TODO: this is too strict since anyfunc <: anyref)

(Eventually)  `T(anyref)` can be targeted by element segments holding function values,  and the values stored in such tables are the function values that would be obtained from the host side if the host side reached into a corresponding anyfunc table and extracted functions.

`call_indirect` requires `T(anyfunc)`; `T(anyref)` is not allowed.

table.get: on `T(anyref)` this is obvious; on `T(anyfunc)` what do we return?  probably a function object.  But under what restrictions?  Exported function?

table.set: on `T(anyref)` this is fairly obvious; value must be anyref.  on `T(anyfunc)` the static type must be anyfunc.

TODO: no downcast function at this point, so if we have an `anyref` we can't cast it dynamically to `anyfunc` or test whether we could do that safely.  

### Tentative cleanup

Change the current `(ref.null T)` construction into `ref.null` and introduce the nullref type.  For backward compatibility, continue to accept the old encoding?  Tricky, because then we'll need a new opcode.

Rename or create alias for `ref.is_null` as `ref.isnull` (as spec requires).  Superficial text format change.
