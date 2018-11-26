# Version 2 - reftypes + GCTypes Minimal Viable Alpha

Version 2 extends Version 1 to represent two overlapping feature sets: the full [reftypes proposal](https://github.com/WebAssembly/reference-types) (minus the `funcref` type), and a simple system of GC types that is module-internal (types are entirely private to a module) but whose object instances can be passed between wasm modules and between wasm and JS.  For the GC types it still aims to be a "minimal viable alpha" (MVA): the minimal system that can do something useful and allow for experimentation, but which compromises on expressibility and performance in several ways.

Note that Version 2 is not backward compatible with Version 1 due to changes to the `ref.null` instruction.

**Table of contents:**

[Overview](#overview)  
[Feature control](#feature-control)  
[Struct and Ref Types](#struct-and-ref-types)  
[Generalized tables](#generalized-tables)  
[Instructions](#instructions)  
[JS Interface](#js-interface)

## Overview

* Feature control
  * New module section for content to opt-in to this experimental system
  * New --wasm-gc / javascript.options.wasm_gc switches to enable in the engine
* Reference types
  * Types `anyref` and `(ref T)` where T names a structure type
  * Locals, parameters, globals, and return values of reference types
  * Restrictions on `(ref T)` types to avoid exposing types outside the module
  * New instructions `ref.null`, `ref.is_null`, `ref.eq` to manipulate references
* Generalized tables
  * Tables of base type anyref
  * Multiple tables
  * New instructions `table.get`, `table.set`, `table.grow`, `table.size` for manipulating tables
* Structure types
  * Type definitions `(struct (field T) ...)` w/o explicit inheritance
  * Nominal type equality for primitive and reference types
  * Simple prefix typing based on type equality for implict upcast
  * Shallow structural downcast from a type A to a type B where A is a prefix of B
  * New instructions `ref.null`, `ref.is_null`, `ref.eq`, `struct.new`, `struct.get`, `struct.set`, `struct.narrow`
  * Instances of structure types are visible to JS as TypedObject instances

## Feature control

The experimental GC feature is only available if a special section is present in each module.  Without this section, validation will fail.  Currently, all reftypes features also require the special section.

The section has ID = 42 (GcFeatureOptIn), byte length 1, and the single byte in the section is the version number.  As we move to later versions, older content may or may not remain compatible with newer engines; newer engines that cannot process older content will reject the content in validation.

The version number must be `2` for nightlies built on November 26 2018 and later.  Older versions are rejected.

The new section must be the first non-custom section in the module.

In the textual format accepted by SpiderMonkey's wasmTextToBinary(), write `(gc_feature_opt_in 2)` to create this section.

## Struct and Ref Types

Source syntax:

```
Struct     ::= (type Name? (struct Field ...))
Field      ::= (field Name? FieldType)
FieldType  ::= ValueType | (mut ValueType)
ValueType  ::= i32 | i64 | f32 | f64 | anyref | (ref Name)  // the parens are required in the "ref" syntax
Name       ::= $whatever
```

Struct types can be mutually recursive.

Struct names represent type table indices.

The extension of ValueType to incorporate (ref T) extends to all other uses of ValueType: Globals / Global imports, Parameters, Returns, Locals, Block/Loop/If, the Null operator

Field names represent integer field indices.  Field names must be unique in the module; a useful trick to make this restriction bearable is to prepend the structure name to every field name along with a legal separator such as `_` or `.`, eg, `$mystruct.x` for a field logically called `x` inside a structure called `$mystruct`.

### Encoding draft

In the Types section, struct and function types can be interleaved, and the "form" field of each entry in the section determines what we look at.

In the Types section, a struct type looks like this (temporary encoding):

```
form         varint7        0x50 = SLEB(-0x30), represents "struct constructor"
flags        varint7        no flags at present
field_count  varuint32
field_types  field_type*
```

where a field_type is a pair:

```
flags        varint7
type         value_type    // i32 etc, anyref, ref_type
```

and the encoding of a ref_type is straightforward:

```
ref_type     0x6E varuint32    the parameter is a reference into the type section
```

### Validation

The only valid flag in a field_type is 0x01, which indicates a mutable field.

For ref_type, the parameter must name a struct type.

### Type semantics / compatibility:

Equality is nominal:

```
T = U if T and U are the same primitive type (i32, i64, f32, f64)
anyref = anyref
(ref T) = (ref U) if T = U, where T and U are indices into the same type space
```

Type spaces are per-instance.  Thus, the phrase "the same type space" above entails that ref types from two different instances never are equal.  In particular, if a module is instantiated twice and a reference to a struct object created in one instance is passed to the other instance, the latter will not recognize the object reference as an instance of any of its types.

Common subtyping rules:

```
(ref T) <: anyref
(ref T) <: (ref T)
(ref T) <: (ref U) if T <: U
T <: U if there is a type W s.t. T <: W and W <: U
nullref <: anyref
nullref <: (ref T)
```

Note that `nullref` is not expressible in the text or binary formats, it is the result of a `ref.null` expression only.  In the future, there will be reference types of which `nullref` is not a subtype.

Prefix subtyping rule for structures:

```
T <: U if T and U are struct types, T has at least as many fields as U, 
          T does not have an extends clause [currently there is no such thing as an extends clause],
          and the types of the corresponding fields of U and T are the same (as determined by =)
```

Nominal subtyping rule for structures (not yet meaningful because no *extends* clause):

```
T <: U if T and U are struct types and T extends U or T extends W where W <: U by the nominal subtyping rule
```

There are automatic upcasts to supertypes.  Currently these upcasts always use the prefix rule.

## Generalized tables

### Tables-of-anyref

A table can now be of type `anyref` in addition to type `anyfunc`.  In the following, denote table-of-anyref as `T(anyref)` and table-of-anyfunc as `T(anyfunc)`.

Element segments are *not* further extended, they can reference only function values.

Encoding of `T(anyref)`:

* The code for `anyref` is 0x6F, its standard type code.
* `anyref` can be used in the wasm text format for the type of the table.
* `"anyref"` can be used as the element name in the descriptor passed to the JS `WebAssembly.Table` constructor.

### Multiple tables

There can be several tables, with indices starting at zero.  As usual, imports are numbered before local tables.

In the text format, tables can be named, `(table $tablename length type)`.

An active element segment still can only target one specific table, since the table index is baked into the element.  But it can target any of the tables in the module: `(elem $tablename init-index-expr function ...)`.  A passive segment cannot carry a table index.

Encoding: The reserved flags byte in the instruction has bit 0x04 set, in which case there is a table index immediately following (two indices, in the case of `table.copy`).  This encoding will change for sure.

The table index is always baked into the wasm instructions.  In the text format, the table index is optional for backwards compatibility reasons.

Fully parenthesized syntax for instructions defined below:

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

In the RPN text format, if the instructions carry table indices then they follow the opcodes, eg, 

```
element-index-expr
table.get table-ref
```

```
argument-expr
call_indirect table-ref type-ref
```

```
dest-index-expr
src-index-expr
len-expr
table.copy dest-table-ref src-table-ref
```

We can `table.copy` only between tables of the same type.

## Instructions

Instruction encodings are temporary.  0xFC is the "misc" prefix, previously "numeric".

`Instruction ::= CallIndirect | Ref.Null | Ref.IsNull | Ref.Eq | Table.Init | Table.Get | Table.Set | Table.Grow | Table.Size | Table.Fill | Struct.New | Struct.Get | Struct.Set | Struct.Narrow`

### call_indirect

The existing `call_indirect` instruction can operate on any table, but requires the base type of that table to be `anyfunc` and will fail validation if the base type is `anyref`.

### ref.null

Create a null reference.

_Synopsis:_ `Ref.Null ::= ... <ref.null>`

_Encoding:_ 0xD0

_Result type_: NullRef

_Execution_:
* Push a null reference

### ref.is_null

Compare a reference to null.

_Synopsis:_ `Ref.IsNull ::= ... Expr <ref.is_null>`

_Encoding:_ 0xD1

_Validation:_
* The type of Expr must be anyref or (ref T) where T is a struct type

_Result type:_ i32

_Execution:_
* pop Expr
* if Expr is a null reference, push 1, otherwise push 0

### ref.eq

Compare two references for pointer equality.

_Synopsis:_ `Ref.Eq ::= ... Expr_0 Expr_1 <ref.eq>`

_Encoding:_ 0xD2

_Validation:_
* The types of Expr_0 and Expr_1 must unify in the normal way and must be subtypes of anyref

_Result type:_ i32

_Execution:_
* pop Expr_0 and Expr_1
* if Expr_0 and Expr_1 are the same pointer value push 1, otherwise push 0

### table.init

The existing `table.init` instruction (from the [bulk memory proposal](https://github.com/WebAssembly/bulk-memory-operations/)) can only be used to initialize tables with base type anyfunc since the source in this case is an elem segment, and currently our elem segments can only reference function values.

### table.get

`(table.get index-expr)` can target only `T(anyref)`, the result is `anyref`.

Encoding: (0xFC 0x10 0x00) where the last byte is a flags byte that will eventually accomodate a table index.  Traps on OOB with RangeError.

### table.set

`(table.set index-expr value-expr)` can target only `T(anyref)` and the static type of the value must be some `ref` type; the result is void.  Traps on OOB with RangeError.

Encoding: (0xFC 0x11 0x00) where the last byte is a flags byte that will eventually accomodate a table index

### table.grow

`(table.grow delta-expr init-value-expr)` can target only `T(anyref)`.  It exposes the existing JS-level mechanism to wasm and lets even non-exported tables grow. The argument is i32.  The result is i32, the old size of the table; the result will appear negative for memories > 2GB.  Traps on negative arguments with RangeError, so the maximum growth increment is 2GB; not ideal.  Traps with RuntimeError if the grow fails.

The `init-value` is the value to fill the new slots with, it must be a subtype of anyref.  (The reason `table.grow` can target only `T(anyref)` is that we don't have a notion of a function value yet.)

Encoding: (0xFC 0x0F 0x00) where the last byte is a flags byte that will eventually accomodate a table index

### table.size

`(table.size)` returns the size of the table as an i32.

### table.fill

TBD - not yet implemented

### struct.new

Construct a new object of a given type, providing initial values for all fields.

`Struct.New ::= ... Expr_0 Expr_1 ... Expr_k <struct.new StructName>`

_Syntax:_ StructName is a type table index embedded in the instruction

_Encoding:_ 0xFC 0x50 StructName:varuint32

_Validation:_
* StructName must name a struct type T with K fields
* The number of values on the stack is greater than or equal to K
* The type of Expr_0 is the same as the type of field 0; the type of Expr_1 is the same as the type of field 1; etc 

_Result type:_ (ref T)

_Execution:_
* Create V, a new instance of T.
* If the allocation fails then trap.
* Pop K values off the stack and store them into the corresponding fields of V: the value of Expr_0 to field 0, Expr_1 to field 1, and so on.
* Push &V

### struct.get

Read the value of a field from an object via a typed reference.

_Synopsis:_ `Struct.Get ::= ... Expr <struct.get StructName FieldName>`

_Syntax:_ StructName is a type table index; FieldName is a field index

_Encoding:_ 0xFC 0x51 StructName:varuint32 FieldName:varuint32

_Validation:_
* StructName must name a struct type T with L fields
* FieldName must be less than L
* The type of Expr must be (ref U) for some struct type U s.t. U <: T

_Result type:_ The type of field FieldName in T

_Execution:_
* Let Ptr be the value of Expr
* If Ptr is NULL then trap
* Push the value of field FieldName of *Ptr.

### struct.set

Write a value to a field of an object via a typed reference.

_Synopsis_: `Struct.Set ::= ... Expr_0 Expr_1 <struct.set StructName FieldName>`

_Syntax:_ StructName is a type table index; FieldName is a field index

_Encoding:_ 0xFC 0x52 StructName:varuint32 FieldName:varuint32

_Validation:_
* StructName must name a struct type T with L fields
* FieldName must be less than L
* The type of Expr_0 must be (ref U) for some struct type U s.t. U <: T
* Let M be the mutability of field FieldName of T
* We must have M = "mut"
* Let V be the type of Expr_1 and W be the type of field FieldName of T
* We must have V <: W

_Result type:_ void

_Execution:_
* Let Ptr be the value of Expr_0
* Let Value be the value of Expr_1
* If Ptr is NULL then trap.
* Set the value of field FieldName of *Ptr to Value

### struct.narrow

This is a *structural* downcast.  It is slow.

_Synopsis_: `Struct.Narrow ::= ... Expr <struct.narrow FromType ToType>`

_Syntax:_ FromType and ToType are ValTypes, indicating known source type and target type

_Encoding:_ 0xFC 0x53 FromType:ValType ToType:ValType

_Validation:_
* FromType must be anyref or (ref T) where T is a structure
* ToType must be anyref or (ref U) where U is a structure
* If ToType is anyref then FromType must be anyref
* If FromType is (ref T) and ToType is (ref U) then we must have U <: T

_Result type:_ ToType

_Execution:_
* Let Ptr be the value of Expr
* If FromType is anyref and ToType is anyref then return Ptr
* If Ptr is NULL then push NULL and exit
* If FromType is anyref then we must first unbox:
  * if the referent of Ptr is not a structure type instance then push NULL and exit
* Let S be the concrete structure type of the referent of Ptr
* If S <: ToType then push Ptr and exit
* Push NULL

## JS interface

### Objects

Wasm struct type instances can escape to JS via anyref parameters/returns/globals and are seen as opaque TypedObjects by JS code.  The fields of these instances are named `_0`, `_1`, and so on; the naming is stopgap, and the naming system will change eventually.

A struct type defined in Wasm is not directly exportable from Wasm and is available to JS only through reflection.  In particular, since each struct instance has a `constructor` property, JS can normally construct new instances by means of `new (obj.constructor)(...)`.

Fields that are immutable in the wasm struct definition are not writable from JS; an attempt to write such a field will cause an exception to be thrown.

As JS does not yet have an int64 or BigInt type, Wasm `i64` fields are reflected in the object instance as two `i32` fields.  If the `i64` field would normally have been named `_n`, the two fields are named `_n_low` and `_n_high`.  These fields are always immutable to JS, and a struct with `i64` fields cannot be constructed from JS -- the constructor is accessible but will throw.

Fields of type `anyref` can be written from JS if they are mutable; fields of type `(ref T)` are however always immutable to JS, and a struct with `(ref T)` fields cannot be constructed from JS -- the constructor is accessible but will throw.

(The "TypedObjects" are not a standard thing, but a Firefox rendition of an evolution of what was once the proposal for TypedObjects in JS.  The best available resource is [here](https://github.com/tschneidereit/typed-objects-explainer), but it too is probably not accurate or complete.  For our purposes, TypedObjects are sealed objects with type-constrained properties and private storage.)

### Export/import restrictions

As types are module-internal for the time being, entities that would reveal types or would require types to be revealed cannot be imported or exported.  Types are revealed by `(ref T)` types, so these entities cannot be exported or imported directly:

* functions whose parameters or result are `(ref T)` types
* globals that hold a `(ref T)` type

In addition, if a table is imported or exported, a function whose parameters or result are `(ref T)` types cannot be stored in the table by means of an initializing element segment.

Finally, we can think of a struct instance escaping to JS as exporting it, and a struct instance flowing into a wasm module as importing it.  As described in the previous section, there are run-time restrictions on JS storing values into `(ref T)` typed  fields of "exported" objects by assignment or construction.  Furthermore, a module that "imports" an object that was constructed from a type defined in another module (or for that matter in JS) can only downcast the object to one of its own types if it does not have a `(ref T)` field, a consequence of our current nominal type equivalence.

### WebAssembly.Table.prototype.set

Setting elements in a `T(anyref)` with `WebAssembly.Table.prototype.set(value)` from JS will store JS objects.  If the value being set is not already an object we perform the same coercion as we do on a function boundary when JS passes a non-object to a wasm function that takes an anyref.

In the future, the conercion operation will be a BoxAsAnyref operation, which is not the same as ToObject.

### WebAssembly.Table.prototype.grow

This function takes an optional second argument (note, not implemented as of November 26).  If the argument is not present it is taken to be `null`.  Otherwise it is coerced to an Object value by ToObject (note this fails for `undefined`).  The value is used as the default argument with which to initialize the table.  For a `T(anyfunc)` it must be an appropriate function (TBD - I think this means an exported wasm function).  For a `T(anyref)` any object value will do.

In the future, the conercion operation will be a BoxAsAnyref operation, which is not the same as ToObject.

