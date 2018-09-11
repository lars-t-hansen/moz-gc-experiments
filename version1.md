# Version 1 - the Minimal Viable Alpha

Version 1 represents a simple system of GC types that is module-internal (types are entirely private to a module) but whose object instances can be passed between wasm modules and between wasm and JS.  It aims to be a "minimal viable alpha" (MVA): the minimal system that can do something useful and allow for experimentation, but which compromises on expressibility and performance in several ways.

## Brief summary

* new module section to opt-in to this experimental system
* new --wasm-gc / javascript.options.wasm_gc switches to enable in the engine (*will go away before MVA is complete*)
* structure type definitions `(struct (field T) ...)` w/o explicit inheritance
* reference types: `anyref` and `(ref T)` where T names a structure type
* nominal type equality for primitive and reference types
* simple prefix typing based on type equality for implict upcast
* shallow structural downcast from a type A to a type B where A is a prefix of B
* locals, parameters, globals, and return values of reference types
* restrictions on `(ref T)` types to avoid exposing types outside the module
* new instructions `ref.null`, `ref.is_null`, `ref.eq`, `struct.new`, `struct.get`, `struct.set`, `struct.narrow`
* instances of structure types are visible to JS as TypedObject instances

Vesion 1 is not yet in any Firefox build; follow [bug 1444925](https://bugzilla.mozilla.org/show_bug.cgi?id=1444925) and its blockers.

## Detailed scope for v1

There are only structure types, no array types yet.

The structure types are "plain" (no notion of methods or vtables) and there is no explicit inheritance among types.

Wasm structure types are instantiated as JS TypedObject instances.

Wasm struct instances can be passed between Wasm and JS as function arguments, function return values, and in global variables.

Struct types / ref types (other than anyref) are entirely private to a module and are not exposed in any way via function arguments, function returns, or globals, not even via functions in tables; nor are the constructors for the JS TypedObjects that represent the wasm objects available to JS.  Furthermore, JS can read object fields that have ref types (other than anyref) but cannot write them; JS sees them as if they were marked immutable.

Note: Code that needs to communicate with JS must accept anyref as parameters and return anyref as values, or use anyref globals, or anyref object fields, and must use struct.narrow to check type compatibility as appropriate.

We use name equivalence for our intra-module types: (ref T) = (ref U) iff T and U are the same type index

There is implicit inheritance among structures: if a structure B is a prefix of a structure D (ie D has at least as many fields as B and the corresponding types of the fields of B and D have types that are equal) then D is a subtype of B.  The instruction struct.narrow can be used to test whether something that is known to be a B is in fact a D.

## Feature control

The experimental GC feature is only available if a special section is present in each module.  Without this section, validation will fail.

The section has ID = 42 (GcFeatureOptIn), byte length 1, and the single byte in the section is the version number, which must be `1` at the moment.  As we move to later versions, older content may or may not remain compatible with newer engines; newer engines that cannot process older content will reject the content in validation.

The new section must be the first non-custom section in the module.

In the textual format accepted by SpiderMonkey's wasmTextToBinary(), write `(gc_feature_opt_in 1)` to create this section.

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

Field names represent integer field indices.  Field names can be reused but must always represent the same (index, type) pair.
The change to ValueType to incorporate (ref T) extends to all other uses of ValueType: Globals / Global imports, Parameters, Returns, Locals, Block/Loop/If, the Null operator

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
```

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

## Instructions

Instruction encodings are temporary.  0xFC is the "misc" prefix, previously "numeric".

`Instruction ::= Ref.Eq | Struct.New | Struct.Get | Struct.Set | Struct.Narrow`

### ref.eq

Compare two references for pointer equality.

_Synopsis:_ `Ref.Eq ::= ... Expr_0 Expr_1 <ref.eq>`

_Encoding:_ 0xD1

_Validation:_
* The types of Expr_0 and Expr_1 must unify in the normal way and must be subtypes of anyref

_Result type:_ i32

_Execution:_
* pop Expr_0 and Expr_1
* if Expr_0 and Expr_1 are the same pointer value return 1, otherwise return 0

### struct.new

Construct a new object of a given type, providing initial values for all fields.

`Struct.New ::= ... Expr_0 Expr_1 ... Expr_k <struct.new StructName>`

_Syntax:_ StructName is a type table index embedded in the instruction

_Encoding:_ 0xFC 0x50 StructName:varuint32; StructName references the TYPE section of the module

_Validation:_
* StructName must name a struct type T with K fields
* The number of values on the stack is greater than or equal to K
* The type of Expr_0 is the same as the type of field 0; the type of Expr_1 is the same as the type of field 1; etc 

_Result type:_ (ref T)

_Execution:_
* Create V, a new instance of T.
* If the allocation fails then trap.
* Pop K values off the stack and store them into the corresponding fields of V: the value of Expr_0 to field 0, Expr_1 to field 1, and so on.
* Return &V

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
* Return the value of field FieldName of *Ptr.

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
* If Ptr is NULL then return NULL
* If FromType is anyref then we must first unbox:
  * if the referent of Ptr is not a structure type instance then return NULL
* Let S be the concrete structure type of the referent of Ptr
* If S <: ToType then return Ptr
* Return NULL

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
