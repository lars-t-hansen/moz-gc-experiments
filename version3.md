Ignore this for now.

Things that won't happen in v2.

## anyfunc as a value type, and fallout from that

(Work in progress)

We introduce a new value type `anyfunc`, where `anyfunc` <: `anyref`.

In every text context, `funcref` has the same meaning as `anyfunc`, and probably `funcref` should be the canonical name by and by.

We can `table.copy` from table T1 to table T2 if the element type of T1 is a subtype of the element type of T2.  Specifically, we can `table.copy` from `T(anyfunc)` to `T(anyref)`.  TODO: under what conditions?  Exported functions?

We can `table.init` a `T(anyref)` since anyfunc <: anyref.  TODO: under what conditions?  Exported functions?

(Eventually)  `T(anyref)` can be targeted by element segments holding function values,  and the values stored in such tables are the function values that would be obtained from the host side if the host side reached into a corresponding anyfunc table and extracted functions.

When `table.get` targets a `T(anyfunc)` what do we return?  probably a function object.  But under what restrictions?  Exported function?

this is fairly obvious; value must be anyref.  on `T(anyfunc)` the static type must be anyfunc.

TODO: no downcast function at this point, so if we have an `anyref` we can't cast it dynamically to `anyfunc` or test whether we could do that safely.  

## Currently out of scope for Version 2

### table-of-ref

Along with anyfunc and table-of-anyref we can extend this to table-of-ref in general, with fairly obvious semantics.  This is slightly tricky though because this entails either (a) those tables are not exported or (b) we need to talk about imported / exported types.  We can do (a) for V2 easily, however.
