# Arrays

This is WIP for v2.  Ignore for now.

## Sketch

I imagine an `(array T)` type constructor which constructs a ref type.  T is any primitive
or reference type but not a struct or function type.

Arrays can only be heap-allocated.

Example:

```
(func $dot (param $a (array i32)) (param $b (array i32))
  (local $result i32)
  (local $n i32)
  (block $exit
    (loop $again
      (br_if $exit (i32.eq (get_local $n) (array.len (get_local $a))))
      (set_local $result
        (i32.add
          $result
          (i32.mul (array.get i32 (get_local $a) (get_local $n)) 
                   (array.get i32 (get_local $b) (get_local $n)))))
      (set_local $n (i32.add (get_local $n) (i32.const 1)))
      (br $again)))
  (get_local $result))
```

 Operators are:
 
 `(array.new <base-type> <length> <initial-value>)`  
 `(array.len <obj>)`  
 `(array.get <base-type> <obj> <index>)`  
 `(array.set <base-type> <obj> <index> <value>)`

(There will be some bikeshedding around "len" vs "length" and "get" vs "ref".)

Type compatibility for parameters/returns is simple, certainly for v2 we want invariant typing - base types must match exactly; even if T is a subtype of S we can't pass (array (ref T)) to something expecting (array (ref S)) or vice versa, not even with a runtime check.

Type compatibility for get can probably be more relaxed, but why?  Type compatibility for set must be as for parameters.
 
I think that until we have a notion of type import/export we might as well allow array objects to be exported/imported as anyref, but not allow functions that dabble in array types in their signatures to be exported or imported, nor have exported or imported globals of these types.

Structs with array fields will flow out of wasm to js through anyref interfaces, but a struct with an array field will be unassignable just as if it were a ref field.


## TypedObject integration
 
Cursorily it looks like TypedObject distinguishes arrays of different lengths as being of different types;
this is a poor fit for the above but probably not terrible.  Worse for JS than for wasm, for sure.

The current syntax to create a constructor for an array of a given base type and length is `new TypedObject.ArrayType(basetype, length)`, which is not what the explainer has, but it works in FF64 for primitive array types too.  For ref types we'd want to use TypedObject.Object as the base type.
