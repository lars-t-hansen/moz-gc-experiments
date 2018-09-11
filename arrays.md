# Arrays

This is WIP for v2.  Ignore for now.

## Sketch

I imagine an `(array T)` type constructor which constructs a ref type, ie it's like `(ref T)`
but there is more than one referenced value.  ((ref T) is a pointer to one T; (array T) is
a pointer to lots of Ts.  But in the latter case the Ts are restricted.)  T is any primitive
or reference type.

Arrays can only be heap-allocated (nothing local), so

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
          (i32.mul (array.get i32 (get_local $a) (get_local $n)) (array.get i32 (get_local $b) (get_local $n)))))
      (set_local $n (i32.add (get_local $n) (i32.const 1)))))
  (get_local $result))
```

 Operators are `(array.new <base-type> <length>)`, `(array.length <obj>)`, `(array.get <base-type> <obj> <index>)`,
 `(array.set <base-type> <obj> <index> <value>)`.
 
 Type compatibility is simple, certainly for v2 we want invariant typing (base types match exactly).
 
 ## TypedObject integration
 
 Cursorily it looks like TypedObject distinguishes arrays of different lengths as being of different types;
 this is a poor fit for the above but probably not terrible.
 
