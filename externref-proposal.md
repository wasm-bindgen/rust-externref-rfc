# Boxed Externrefs in Rust

We want to add externref support to Rust. Rust's operational semantics assume
that any Rust value can be serialized to a "bag of bytes with provenance".
However, a true `externref` value is an opaque reference to an object owned by
the host garbage collector. Thus, adding to Rust a type that is lowered to an
externref violates the operational semantics of Rust. Instead, Rust will only
handle pointers to `externref`.

We define a raw pointer type and two user-facing types built on it:
* `externref_ptr` which is a raw pointer to a table slot holding an externref,
* `Externref` which represents an owned externref pointer, and
* `ExternrefRef<'a>` which represents a borrowed externref pointer that is
   guaranteed to be valid for lifetime `'a`.

These are all passed as a slot pointer to/from `extern "C"` functions. (Or any
other non "rust" ABI functions, I'm going to call them `extern "C"` functions
throughout.)

It is a key requirement for us that it should be possible to call C functions
that take and return `__externref_t` directly. For this reason, we define a
`ByValue<_>` struct that can be used to wrap `Externref` and `ExternrefRef<'a>`.
It marks the parameters and return values of `extern "C"` functions as
representing `__externref_t` instead of a slot pointer. If any of the
parameters or the return values have the `ByValue<_>` marker, the compiler emits
appropriate wrappers to adapt between the slot pointer and `__externref_t`.

The only place that we materialize a true `externref` is at the boundary of an
`extern "C"` function when a parameter or return type is explicitly marked
`ByValue<_>`.

The table holding the slots can be owned by Rust, so the proposal does not
depend on any new backend support. There is an LLVM RFC for pointers to
externref which makes the same table visible to LLVM and to C. This enables
optimization of the table operations, and sharing of slot pointers with C as
`__externref_t*`. The proposal is structured so that adopting it is a change of
representation only, in two [implementation phases](#implementation-phases).

## Background

### Wasm `externref`

WebAssembly's `externref` is a reference to a host object, introduced by the
Reference Types proposal. It represents a JavaScript value or a handle from the
embedder. It is not a WasmGC type: it sits outside the WasmGC struct and array
hierarchy as a dedicated host type with its own singular hierarchy. It is opaque
and garbage collected by the host, can be held in locals and passed through
calls, and it is nullable, with `ref.null` as a valid value.

A Wasm module cannot inspect it, cannot store it in linear memory, and cannot
turn it into an integer. The only places `externref` can live are the operand
stack, locals, globals, and *tables*: typed arrays of references, indexed by
integer, accessed with `table.get` and `table.set`. Holding an externref in a
table slot is therefore the only way a module can keep one in a long-lived data
structure or pass its address around, by representing it as a slot index that
*can* be stored in memory.

### LLVM RFC to add support for pointer to externref

There is an
[LLVM RFC](https://discourse.llvm.org/t/rfc-webassembly-externref-t-pointers-in-clang-and-llvm/91914)
to add support for pointer to externref, denoted `__externref_t*` in C.
Here I summarize the relevant details of the LLVM proposal for this proposal.

A blessed table is laid out with a bss section, a stack section, and a heap
section. A pointer to externref is represented in LLVM IR as `ptr addrspace(2)`.
Dereferencing a pointer to externref is lowered as a `table.get`. Assigning to
one is lowered as a `table.set`.

A pointer to externref can be allocated on the heap with
`__externref_table_alloc()` and `__externref_table_free()`. They have the
following LLVM attributes:

`__externref_table_alloc`:
    `allockind("alloc,zeroed")` `allocsize(0)` `"alloc-family"="__externref_table"`
    `mustprogress` `nofree` `nounwind` `willreturn` `memory(inaccessiblemem: readwrite)`

`__externref_table_free`:
    `allockind("free")` `allocsize(0)` `"alloc-family"="__externref_table"`
    `mustprogress` `nounwind` `willreturn` `memory(argmem: readwrite, inaccessiblemem: readwrite)`

This are almost the same attributes as are present on `malloc` and `free`. These
attributes allow a paired `__externref_table_alloc` and `__externref_table_free`
to be optimized out.

With this, C and Rust objects linked together share the same table, so no glue
code is needed to pass externref pointers between them.

## Implementation Phases

The specification below is written against an opaque slot pointer and a small
set of functions, so nothing in it depends on how the backend represents a slot
pointer or on who provides the table. That allows the work to land in two
phases, with the second changing only the representation. No user code changes
between them.

### Phase 1: a Rust-owned table

Everything needed already exists in LLVM: the `externref` IR type,
`externref`-typed function parameters and returns, and the
`llvm.wasm.table.get/set/grow.externref`, `ref.null`, and `ref.is_null`
intrinsics over a table global and an integer index. Rust supplies the rest
itself:

* `externref_ptr` is represented as a plain `usize` index into the table. The
  type stays opaque, so this is invisible to users.
* Rust defines the table and ships the slot allocator in compiler-builtins,
  written in Rust over the table intrinsics. Slot `0` is never allocated.
* All symbols are private. The table and allocator are internal to the linked
  module under Rust-private symbol names, and nothing is exported from or
  imported into the Wasm module. They therefore cannot collide with symbols that
  existing toolchains already export.
* There is no table stack. A slot that is scoped to a call is a heap slot which
  is freed on exit.

This is the model that toolchains such as wasm-bindgen implement today, moved
into the compiler. By-value `externref` interop with C and with the host is
complete in this phase. What it lacks is optimization of the table operations,
since an integer index carries no provenance and the allocator calls are opaque,
and sharing of slot pointers with C, since C has no pointer to externref type.

Dynamic linking is out of scope for this phase. Each linked module has its own
private table, so a slot pointer is only meaningful within it. Between
separately linked modules externrefs are passed by value, which needs nothing
shared.

### Phase 2: with the LLVM RFC

Once `ptr addrspace(2)` is available in LLVM, `externref_ptr` becomes that
pointer, the table becomes the one blessed by LLVM, and the allocator functions
forward to compiler-rt's `__externref_table_alloc` and `__externref_table_free`,
declared with all the attributes that LLVM normally emits. This adds:

* **Optimization.** LLVM performs alias analysis on `addrspace(2)` pointers just
  as it does on memory pointers, and the default `-O2` pipeline removes paired
  allocations whose slot does not escape.
* **A table stack.** A slot that is scoped to a call becomes an `alloca` in
  `addrspace(2)`.
* **Pointer interop with C.** A slot pointer is an `__externref_t*`, so slot
  pointers pass between C and Rust with no glue, and a slot allocated in C may
  be freed in Rust or vice versa.
* **Dynamic linking.** Passing slot pointers between separately linked modules
  requires them to share the table and its allocator, as they do the function
  table and `malloc`. How these are imported and exported is defined by LLVM and
  the WebAssembly tool conventions, rather than by this proposal.

A backend other than LLVM can stay at the phase 1 representation indefinitely;
`__externref_t*` is an `i32` at the wasm level either way.

## Specification

### The `externref_ptr` type

```rs
// core::arch::wasm32, on wasm32/wasm64 with the `reference-types` feature.

/// A pointer to a table slot holding an externref.
#[lang = "externref_ptr"]
pub struct externref_ptr {
    _private: (),
}
```

This type is a lang item representing a pointer to a table slot. It is a thin
pointer whose pointee cannot be read or written directly from Rust since the
pointee is not made of bytes; the functions below are the only way to read or
write the slot. It is `Copy` and pointer-sized. Like raw pointers, it is
`!Send + !Sync`. In phase 1 it is represented as a table index, and in phase 2
as `ptr addrspace(2)` with the ABI of `__externref_t*`.

Its semantics are those of a pointer with provenance in both phases, following
the same rules as `*mut T`, so that code written against phase 1 remains valid
in phase 2. Slot `0` is reserved as a null niche. Each `externref_alloc` call
produces a distinct allocation. It is not allowed to cast `externref_ptr`
directly to or from an integer, but it is possible to use `expose_provenance()`
and `with_exposed_provenance()` to do so.

It is only available on `wasm32`/`wasm64` targets with the `reference-types`
target feature, and lives in `core::arch::wasm32` alongside the types below.

### Intrinsic Helpers

```rs
// core::arch::wasm32

/// Allocates a slot holding `ref.null`.
pub fn externref_alloc() -> externref_ptr;

/// Frees an allocation from `externref_alloc`.
pub unsafe fn externref_free(ptr: externref_ptr);

/// `*to = *from`
pub unsafe fn externref_assign(from: externref_ptr, to: externref_ptr);

/// `*ptr = ref.null`
pub unsafe fn externref_set_null(ptr: externref_ptr);

/// `ref.is_null(*ptr)`
pub unsafe fn externref_is_null(ptr: externref_ptr) -> bool;
```

These are functions of `core::arch::wasm32`, like `memory_grow`, rather than
`extern "C"` symbols. What they link against is an implementation detail which
differs between the phases, and no symbol name is part of the interface.

A freshly allocated slot contains `ref.null` and `externref_free` nulls the slot
before returning it to the free list, so a freed slot never pins a host object.
The allocator is provided by compiler-builtins in phase 1 and by compiler-rt in
phase 2.

Each allocation is a single slot. Contiguous runs of slots, which the LLVM RFC
supports for C arrays, are left out of this proposal.

The remaining functions are single table operations, which the compiler emits
inline:

| Function | Wasm |
|---|---|
| `externref_assign(from, to)` | `table.set(to, table.get(from))` |
| `externref_set_null(ptr)` | `table.set(ptr, ref.null extern)` |
| `externref_is_null(ptr)` | `ref.is_null(table.get(ptr))` |

### Raw Pointer Safety Model

A table slot is an allocation, and `externref_ptr` is a raw pointer to it. The
rules are those of `*mut T` with the table in place of memory:

* **Undefined behaviour**: calling any helper on a null or freed slot; freeing a
  slot twice; freeing a slot that did not come from `externref_alloc`.
  The `unsafe` on the helpers gates these.
* **Not undefined behaviour**: leaking a slot. Like `mem::forget`, it pins the
  host object but violates no Rust invariant. A slot *containing* `ref.null` is
  an ordinary value, and a freshly allocated slot is valid, non-null, and holds
  `ref.null`.

### From Pointers to By-Value Externrefs

`externref_ptr` and its helpers are enough to hold externrefs and to pass their
slots between Rust functions. They are not enough to call a host import or a C
function that takes or returns a bare `__externref_t`. For that, the compiler must
convert between slot and value at the `extern "C"` boundary, and some type in
the signature has to mark the positions where it does so.

The smallest design would be one more raw type for those positions: the same
slot pointer as `externref_ptr`, passed as a bare `__externref_t` when it is a
top-level parameter or return.

```rs
#[repr(transparent)]
#[derive(Clone, Copy)]
pub struct externref(externref_ptr);

unsafe extern "C" {
    // __externref_t c_import(__externref_t x);
    fn c_import(x: externref) -> externref;
}

// __externref_t c_export(__externref_t x);
#[unsafe(no_mangle)]
extern "C" fn c_export(x: externref) -> externref { /* ... */ }
```

Owned and borrowed types would then be an ordinary library over the raw types,
as `OwnedFd` and `BorrowedFd<'a>` are over `RawFd`.

This is not sound, because the conversions act on the slots they are given:

* **A by-value parameter is read.** Passing an externref by value means loading
  it from the argument slot, so the slot must be live for the call. A raw
  pointer can dangle in safe code, so every function taking one by value would
  have to be `unsafe` to call.
* **A by-value return is freed.** The Rust body returns a slot, and the exported
  function must return the value in it. Nothing else can free that slot
  afterwards, so the boundary must, and the body must therefore return a heap
  slot that nothing else refers to. With a `Copy` raw pointer, safe code can
  return a slot that is still owned elsewhere:
  ```rs
  extern "C" fn get() -> externref {
      CACHE.with(|c| externref::from(c.as_externref().as_raw()))
  }
  ```

Rust cannot express either obligation on a raw type. The first needs a rule that
a function is rejected unless it is `unsafe` because of the types in its
signature, including function pointer types after generic substitution, and
Rust never derives the safety of a function from its signature. The second is an
obligation on the body rather than the caller, which `unsafe fn` does not
cover, so it needs a new unsafe attribute. Every exported function handling an
externref would then be `unsafe` to define and to call, even if its body only
used safe types.

The two obligations are a borrow and a transfer of ownership: a parameter slot
must be live for the duration of the call, and a return slot must be uniquely
owned. Rust already expresses both of these in types. So instead of a raw
by-value type, this proposal marks by-value positions on types which carry these
invariants: `ExternrefRef<'a>` for parameters and `Externref` for returns.

This is why the owned and borrowed types are part of this proposal in
`core::arch::wasm32` rather than left to libraries. The compiler's conversions
establish their invariants and rely on them, so they must be types that the
compiler knows.

### Owned and Borrowed Externref Pointers

We define two types `Externref` and `ExternrefRef<'a>`. The first represents an
owned externref pointer that will be freed on drop. The second represents a
borrowed externref pointer that is valid for lifetime `'a`.

The design is analogous to the `std::os::fd` safety types: `externref_ptr`
corresponds to `RawFd`, `Externref` corresponds to `OwnedFd` and
`ExternrefRef<'a>` corresponds to `BorrowedFd<'a>`.

Both are `repr(transparent)` over `externref_ptr`. They are both assumed
non-null so `0` is available as a niche. Because `externref_ptr` is
`!Send + !Sync`, so are they.

Their invariants are that `Externref` uniquely owns a live heap slot from
construction to drop, and `ExternrefRef<'a>` points at a slot that is live for
`'a`. Given these, every method on both types is safe.

`Externref` and `ExternrefRef<_>` have interior mutibility. Because Rust does
not represent `__externref_t`, it is not possible for the pointer owned by an
`ExternrefRef` to ever alias with a Rust reference. For this reason, the
interior mutability poses no safety problems.

#### Borrowed Externref Pointers

The type `ExternrefRef<'a>` is defined as follows:

```rs
/// A borrowed externref pointer, valid for `'a`.
#[repr(transparent)]
#[derive(Clone, Copy)]
pub struct ExternrefRef<'a> {
    ptr: externref_ptr,
    _marker: PhantomData<&'a ()>,
}

/// Anything that can lend out a slot.
pub trait AsExternref {
    fn as_externref(&self) -> ExternrefRef<'_>;
}
impl AsExternref for Externref { /* ... */ }
impl AsExternref for ExternrefRef<'_> { /* ... */ }
impl<T: AsExternref + ByValueRepr> AsExternref for ByValue<T> { /* ... */ }
impl<T: AsExternref + ?Sized> AsExternref for &T { /* ... */ }

impl ExternrefRef<'_> {
    pub fn is_null(self) -> bool {
        unsafe { externref_is_null(self.ptr) }
    }

    pub fn set(self, src: impl AsExternref) {
        unsafe { externref_assign(src.as_externref().ptr, self.ptr) }
    }

    pub fn set_null(self) {
        unsafe { externref_set_null(self.ptr) }
    }

    /// Copies the referenced value into a new owned slot.
    pub fn to_owned(self) -> Externref {
        let result = Externref::null();
        result.set(self);
        result
    }

    /// `ptr` must point to a slot that stays live for `'a`.
    pub unsafe fn from_raw(ptr: externref_ptr) -> Self { /* ... */ }
    pub fn as_raw(self) -> externref_ptr { self.ptr }
}
```

#### `Externref`

The type `Externref` is defined as follows:

```rs
/// An owned externref pointer. Its contents may be `ref.null`.
#[repr(transparent)]
pub struct Externref { ptr: externref_ptr }

impl Externref {
    /// Allocates an externref pointer with value initialized to `ref.null`.
    pub fn null() -> Externref {
        Externref { ptr: externref_alloc() }
    }

    pub fn is_null(&self) -> bool { self.as_externref().is_null() }
    pub fn set(&self, src: impl AsExternref) { self.as_externref().set(src) }
    pub fn set_null(&self) { self.as_externref().set_null() }

    /// `ptr` must be a live allocation from `externref_alloc` that
    /// nothing else owns.
    pub unsafe fn from_raw(ptr: externref_ptr) -> Self { /* ... */ }

    /// Releases ownership without freeing. Leaking is safe.
    pub fn into_raw(self) -> externref_ptr { /* ... */ }
}

impl Drop for Externref {
    fn drop(&mut self) {
        unsafe { externref_free(self.ptr) };
    }
}

impl Clone for Externref {
    fn clone(&self) -> Self { self.as_externref().to_owned() }
}
```

### Passing Raw Externrefs to/from foreign functions

We need a value that has C ABI `__externref_t`. For this purpose, we define the
`ByValue<T>` langitem. When this appears as an argument or a return value to an
`extern "C"` function, the compiler emits special wrapper code to convert
to/from raw `__externref_t`.

In an `extern "C"` function signature, it is only legal for `ByValue<T>` to
appear as a top level type. If it appears as a reference or pointer target, or
as a tuple, struct, or array element, a compiler error is emitted. Furthermore
each position admits exactly one `T`: arguments must be
`ByValue<ExternrefRef<'_>>` and returns must be `ByValue<Externref>`. If a
parameter has type `ByValue<Externref>` or a return value has type
`ByValue<ExternrefRef<'_>>` a compile error is emitted.

```rs
// Trait bound to limit T to one of Externref or ExternrefRef
pub trait ByValueRepr: AsExternref + sealed::Sealed {}
impl ByValueRepr for Externref {}
impl ByValueRepr for ExternrefRef<'_> {}

/// Marker: at a non-`"Rust"` ABI boundary the C side sees a bare
/// `__externref_t`; the Rust side sees `T`.
#[lang = "externref_by_value"]
#[repr(transparent)]
pub struct ByValue<T: ByValueRepr>(T);

impl<T: ByValueRepr> ByValue<T> {
    pub fn new(t: T) -> Self { ByValue(t) }
    pub fn into_inner(self) -> T { self.0 }
}
impl<T: ByValueRepr> From<T> for ByValue<T> { /* free */ }
```

The four by-value positions and the slot each one works with are:

| Position | Type | Wrapper | Slot ownership |
|---|---|---|---|
| import parameter | `ByValue<ExternrefRef<'_>>` | `table.get` the slot, pass the value | caller lends for the call |
| import return | `ByValue<Externref>` | `externref_alloc()`, `table.set` the value | caller owns |
| export parameter | `ByValue<ExternrefRef<'_>>` | `table.set` into a slot scoped to the call, pass its address | wrapper lends for the call |
| export return | `ByValue<Externref>` | `table.get`, `externref_free`, return the value | body transfers, wrapper frees |

The by-value translation happens inside the Rust object, so the wasm the linker
sees uses plain `externref` parameters and results. Clang, GCC, or hand-written
wat link against it without knowing anything about Rust.

The conversions below are shown in C, using the notation of the LLVM RFC in
which an `__externref_t*` is a slot pointer, `*p` is a `table.get` and `*p = x`
is a `table.set`. They are the same in both phases.

#### Import ABI Wrappers

For a foreign function declared in an `extern` block, calls from Rust resolve
to a compiler-generated wrapper that translates each `ByValue` position:

* `ByValue<Externref>` argument: Compile error. Pass
  `ByValue::new(x.as_externref())` instead.
* `ByValue<ExternrefRef<'_>>` argument: The wrapper loads the externref and
  passes it by value. In C the conversion code would look like:
  ```C
  __externref_t convert_borrowed_arg(__externref_t* arg) {
    return *arg;
  }
  ```

* `ByValue<Externref>` return: The wrapper allocates a fresh externref pointer,
  stores the returned externref into it, and returns the slot as an owned
  `Externref`. In C the conversion code would look like:
  ```C
  __externref_t* convert_return_value(__externref_t retval) {
    __externref_t* result = externref_alloc();
    *result = retval;
    return result;
  }
  ```

* `ByValue<ExternrefRef<'_>>` return: Compile error.

Example:

```rs
unsafe extern "C" {
    fn c_import(x: ByValue<ExternrefRef<'_>>, y: ExternrefRef<'_>) -> ByValue<Externref>;
}
```

corresponds to the C declaration

```C
__externref_t c_import(__externref_t x, __externref_t* y);
```

Only the `ByValue` parameter and the return are translated; `y` is already a
pointer, which C can receive in phase 2.

Calls to `c_import` from Rust resolve to an implicit wrapper that loads the
argument slot, allocates a slot for the return value, and stores into it. In
pseudo code:

```rs
fn c_import(x: ByValue<ExternrefRef<'_>>, y: ExternrefRef<'_>) -> ByValue<Externref> {
    let result = Externref::null();
    unsafe { c_import_do_call(x.0.ptr, y.ptr, result.ptr) };
    ByValue(result)
}
```

with the C helper

```C
void c_import_do_call(__externref_t* x, __externref_t* y, __externref_t* result) {
    *result = orig_c_import(*x, y);
}
```

#### Export ABI Wrappers

For a Rust-defined `extern` function, the exported symbol is a
compiler-generated wrapper with the C by-value signature that calls the real
Rust body with slot pointers:

* `ByValue<Externref>` argument: Compile error. Declare the parameter as
  `ByValue<ExternrefRef<'_>>` and call `.to_owned()` in the body if an owned
  slot is needed.
* `ByValue<ExternrefRef<'_>>` argument: Store the incoming `__externref_t` into
  a slot scoped to the call and pass its address. In phase 2 this is an alloca
  on the table stack. In C the conversion looks like:
  ```C
  __externref_t* wrapper(..., __externref_t by_value_arg, /* ... other args */) {
    return rust_func(..., &by_value_arg, ...);
  }
  ```
* `ByValue<Externref>` return: The Rust body returns an owned externref
  pointer. The wrapper loads the externref, frees the pointer, and returns the
  externref by value. In C the conversion code would look like:
  ```C
  __externref_t convert_return_value(__externref_t* retval) {
    __externref_t result = *retval;
    externref_free(retval);
    return result;
  }
  ```
* `ByValue<ExternrefRef<'_>>` return: Compile error.

Example:

```rs
#[unsafe(no_mangle)]
extern "C" fn c_export(x: ByValue<ExternrefRef<'_>>) -> ByValue<Externref> {
    // ... do something
}
```

exports the C signature `__externref_t c_export(__externref_t x)` via the
wrapper:

```C
__externref_t c_export(__externref_t x) {
    __externref_t* result_ptr = orig_c_export(&x);
    __externref_t result = *result_ptr;
    externref_free(result_ptr);
    return result;
}
```

Direct calls to `c_export` from Rust resolve to `orig_c_export`, i.e. they
pass table slots and never go through the wrapper. The symbol
`c_export` (or the name given by `#[export_name]`) always refers to the
wrapped export.

#### Export Parameter Lifetimes

The slot behind a `ByValue<ExternrefRef<'a>>` parameter of an export belongs to
the wrapper, so it is only live for the duration of the call. The caller is the
host, which cannot be checked against a lifetime, so the definition must accept
any lifetime. Otherwise safe code could keep the slot beyond the call:

```rs
extern "C" fn stash(x: ByValue<ExternrefRef<'static>>) {
    KEEP.set(Some(x.into_inner()))
}

extern "C" fn stash_into<'a>(x: ByValue<ExternrefRef<'a>>, v: &mut Vec<ExternrefRef<'a>>) {
    v.push(x.into_inner())
}
```

Both are compile errors. The rule is structural, and is checked on the signature
of the definition after type aliases are expanded. The lifetime of a by-value
parameter must:

* be a late-bound lifetime parameter of the function itself, and
* not appear anywhere in the signature outside of by-value parameters.

Late-bound is the compiler's existing classification of the lifetime parameters
that are chosen per call. It excludes `'static`, the lifetime parameters of an
enclosing `impl`, and any lifetime mentioned in a `where` clause or bound. An
elided `'_` always satisfies the rule. Several by-value parameters may share one
lifetime, since all of the wrapper's slots are live for the same call.

`Drop` impls have the same restriction for the same reason. Drop glue is a
compiler-generated caller that cannot be checked against a specialized
signature, so `impl Drop for Foo<'static>` and `impl<'a> Drop for Foo<'a, 'a>`
are rejected (E0366).

The rule is conservative. A lifetime shared with a parameter that gives the body
nowhere to keep the slot, such as `&'a u8`, is rejected as well. It can later be
relaxed to requiring that the body's signature is a subtype of the signature the
wrapper calls it with, which accepts strictly more definitions.

The rule applies only to definitions. Foreign declarations and function pointer
types may name any lifetime, since there it is a requirement on the Rust caller,
and the wrapper only reads the slot.

Lifetimes are erased before a generic function is instantiated, so none of this
can be checked through a type parameter:

```rs
extern "C" fn f<T: Stash>(x: T) { x.stash() }

let p: extern "C" fn(ByValue<ExternrefRef<'static>>) = f;
```

By-value positions must therefore be written out in the definition. If a type
parameter is a top-level parameter or return type of an `extern "C"` definition,
instantiating it with `ByValue<_>` is an error, reported at monomorphization.

#### `ByValue<_>` and Function Pointers

An `extern "C"` function pointer mentioning `ByValue<_>` always has the C
by-value ABI.

For isntance, a function `f` declared like
```rs
extern "C" fn f(ByValue<ExternrefRef<'_>>) -> ByValue<Externref>
```
has C signature:
```C
__externref_t f(__externref_t)
```
Coercing `f` to a pointer type yields the address of the export wrapper. Calling
through such a pointer from Rust goes through an import wrapper. This ensures
that the pointer type has the same ABI whether the pointer came from Rust or
from C.

#### Example

```rs
unsafe extern "C" {
    fn host_lookup(key: *const c_char) -> ByValue<Externref>;
    fn host_process(v: ByValue<ExternrefRef<'_>>);
}

/// Import return: the wrapper allocated the slot, and we own it.
pub fn lookup(key: &CStr) -> Externref {
    unsafe { host_lookup(key.as_ptr()) }.into_inner()
}

/// Import argument: lend the slot for the duration of the call.
pub fn process(v: ExternrefRef<'_>) {
    unsafe { host_process(ByValue::new(v)) }
}

/// Export: the parameter is borrowed, so returning it takes a copy.
#[unsafe(no_mangle)]
extern "C" fn rust_id(x: ByValue<ExternrefRef<'_>>) -> ByValue<Externref> {
    ByValue::new(x.into_inner().to_owned())
}
```

#### Structs

No translation is applied to struct fields. In phase 2, a `repr(C)` struct
containing `Externref` fields corresponds to a C struct containing
`__externref_t*` fields.

```rs
#[repr(C)]
struct Pair {
    x: Externref,
    y: Externref,
}
```
corresponds to:
```C
struct Pair {
    __externref_t* x;
    __externref_t* y;
};
```

No Rust type corresponds to a C struct with a bare `__externref_t` field,
because no such C struct can exist.

#### Borrowed slots across the boundary

In phase 2, C code that passes the address of a table-stack local, e.g.

```C
void c_fn(__externref_t* p);
__externref_t local = ...;
c_fn(&local);
```

is declared in Rust as `fn c_fn(p: ExternrefRef<'_>)` or, for a Rust export,
`extern "C" fn c_fn(p: ExternrefRef<'_>)`.

#### `with_stack_slot`

```rs
pub fn with_stack_slot<R>(f: impl for<'a> FnOnce(ExternrefRef<'a>) -> R) -> R;
```

A scoped intrinsic providing a slot for the duration of the closure: the slot is
set to `ref.null` on entry, the closure receives its address, and it is cleared
again on exit so that it does not pin host objects after the scope ends. In
phase 2 it is lowered to an `alloca` of one `__externref_t` in `addrspace(2)`,
and in phase 1 to a heap slot which is freed on exit. The HRTB lifetime prevents
the slot from escaping; code that needs the value afterwards calls `to_owned()`
and gets an explicit heap `Externref`.

The use case is scratch slots in small, frequently-called functions, where an
`Externref::null()` cannot be hoisted out of a loop because the function
boundary is the loop body:

```rs
fn swap(a: ExternrefRef<'_>, b: ExternrefRef<'_>) {
    with_stack_slot(|tmp| {
        tmp.set(a);
        a.set(b);
        b.set(tmp);
    });
}
```

The other common case, once slot pointers can be shared with C in phase 2, is a
C function that returns through an out-pointer, where the received value is only
forwarded:

```C
void lookup(const char* key, __externref_t* out);
void process(__externref_t x);
```

```rs
unsafe extern "C" {
    fn lookup(key: *const c_char, out: ExternrefRef<'_>);
    fn process(x: ByValue<ExternrefRef<'_>>);
}

fn lookup_and_process(key: &CStr) {
    with_stack_slot(|tmp| unsafe {
        lookup(key.as_ptr(), tmp);
        process(ByValue::new(tmp));
    });
}
```

Note this only helps for out-pointer returns. A C function that returns
`__externref_t` by value is declared `-> ByValue<Externref>` and the import
wrapper allocates a heap slot for the result; Rust has no way to express
"return into this slot", so that allocation stays unless the optimizer removes
it.

### Optimizing out allocations

This section applies to phase 2, where the allocator has the attributes given in
the LLVM RFC. Most `externref_alloc`/`externref_free` pairs introduced by the
wrappers have the shape

```C
__externref_t* p = externref_alloc();
*p = x;
use(*p);
externref_free(p);
```
where the slot never escapes as a pointer. The default `-O2` compilation
pipeline will optimize this to the following:
```C
use(x);
```

An import return passed straight into another import therefore compiles to two
calls with no table traffic. An export parameter is a table-stack slot of the
wrapper, so there is no allocation to remove, including when the export is
called from the host. A body that keeps its parameter pays for the `to_owned()`
copy.

Only pointers that escape into a callee survive. For those, eliminating the
allocation requires the `AAHeapToStack` pass, which runs only under the
Attributor and is not in the default compilation pipeline. Using
`with_stack_slot` manually ensures that no allocation occurs in these cases. For
example, here the pointer escapes:
```C
__externref_t* p = externref_alloc();
*p = x;
use(p);
externref_free(p);
```
Ideally after optimizations the output would be the same as for the following:
```C
use(&x);
```
In order for this to happen, we must compile with the attributor and it must
prove one of the following two conditions:
1. `use()` never unwinds and never runs forever, or
2. `use()` does not capture or free the pointer.

Unfortunately, in order for `use()` to do anything useful with `p` it probably
has to give it to a host function and the Attributor probably can't prove
anything about what happens from there.

In phase 1 none of this applies: an integer index carries no provenance and
the allocator calls are opaque, so every wrapper allocation is executed, as it
is in existing toolchains today.
