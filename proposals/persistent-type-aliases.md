# Persistent type aliases

Champion issue: <link to the champion issue>

## Summary
[summary]: #summary

Introduce persistent type aliases over existing C# types. A persistent alias gives an existing representation type a distinct compile-time identity, so APIs can distinguish values such as `CustomerId`, `OrderId`, and `ProductId` even when all three are represented as `int` at run time.

```cs
alias CustomerId : int
{
    public bool IsTest => this < 0;
}

alias OrderId : int;

void LoadCustomer(CustomerId id);
void LoadOrder(OrderId id);

CustomerId customerId = ...;
OrderId orderId = customerId; // error
```

Persistent aliases are intended to be erased (a `CustomerId` is represented at run time as its underlying type) and round-tripped (a `CustomerId` in a public API is viewed as a `CustomerId`, not an `int`, by a capable compiler).  
Alias members are written as if they were members of the alias type and may use `this`, but are syntactic sugar for extension members over the alias type.

## Motivation
[motivation]: #motivation

C# programmers frequently use primitive, framework, or domain-neutral types to represent distinct domain concepts:

```cs
void Transfer(int fromAccountId, int toAccountId, decimal amount);
void Move(double distanceInMeters, double altitudeInMeters);
void SendEmail(string address, string subject, string body);
```

These signatures are compact, but they do not encode the intended roles of their values. Accidentally passing an order id where a customer id is expected, mixing values measured in different units, or mixing trusted and untrusted strings are all type-correct today if the underlying representations match.

Developers can create wrapper structs or records to recover nominal distinction:

```cs
readonly record struct CustomerId(int Value);
readonly record struct OrderId(int Value);
```

That pattern works, but it has costs:

- It introduces a new runtime type and representation, even when the desired model is purely a compile-time distinction.
- It requires forwarding or re-declaring common operations from the underlying type.
- It affects serialization, reflection, interop, generic constraints, default values, and overload resolution as a normal wrapper type.
- It is verbose enough that many APIs keep using primitives instead.

Developers can create type aliases:

```cs
using CustomerId = int;
```

This improves readability but:
- It must be repeated from file to file.
- It does not prevent mixing a `CustomerId` with any other `int`.

The goal of persistent type aliases is to provide nominal static checking with the runtime behavior and performance profile of the underlying representation.  
This is similar in spirit to [branded types](https://github.com/microsoft/TypeScript/wiki/FAQ#can-i-make-a-type-alias-nominal) in TypeScript, [opaque type aliases](https://docs.scala-lang.org/scala3/book/types-opaque-types.html) in Scala 3, or [newtypes](https://www.haskell.org/onlinereport/haskell2010/haskellch4.html#x10-710004.2.3) in Haskell.

## Detailed design
[design]: #detailed-design

This proposal introduces persistent alias types as erased, nominal source-level types over an underlying representation type:

- A persistent alias is a distinct type in source, but is not an ordinary wrapper type.
- A persistent alias is erased to its underlying representation.
- Alias members are specified in terms of `this`, with no named receiver parameter in source.
- Lowering preserves that abstraction boundary; any receiver parameter used by the implementation is an implementation detail and does not introduce a user-visible or metadata-visible synthesized name.

TODO phrase principle better
The guiding principle is that an alias value is transparent where its underlying value is expected. Anything that can be done with a value of `U` can also be done with a value of `A`, including invoking members and extension members of `U`, applying conversions from `U`, and satisfying constraints satisfied by `U`.

Throughout this section, ~~strikethrough~~ indicates text being removed from the existing specification, and **bold** indicates text being added.  Unchanged prose is quoted verbatim for context.

### Grammar

The following change is applied to the grammar in [§14.7](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/namespaces.md#147-type-declarations):

```diff
 type_declaration
    : class_declaration
    | struct_declaration
    | interface_declaration
    | enum_declaration
    | delegate_declaration
+    | alias_type_declaration
    ;
+
+alias_type_declaration
+    : attributes? alias_modifier* 'alias' identifier ':' type alias_type_body
+    ;
+
+alias_modifier
+    : 'new'
+    | 'public'
+    | 'protected'
+    | 'internal'
+    | 'private'
+    | unsafe_modifier
+    ;
+
+alias_type_body
+    : ';'
+    | '{' extension_member_declaration* '}' ';'?
+    ;
```

TODO review modifiers and body, do we need to introduce a new alias_modifier production?

### Alias type declarations

An alias type declaration specifies its underlying type:

```cs
alias CustomerId : int;
```

The identifier `CustomerId` is a type name in source. It may be used anywhere a type name could be used, including signatures, type arguments, pointer element type.

The type after `:` is the alias type's underlying type.

TODO: what are the restrictions on underlying type?

Two alias declarations with the same underlying type are distinct source-level types:

```cs
alias CustomerId : int;
alias OrderId : int;

CustomerId customerId = ...;
OrderId orderId = ...;

LoadCustomer(customerId); // ok
LoadCustomer(orderId);    // error
```

### Alias type members

Members can be declared in an alias type as long as they would be allowed in extension blocks.  
Instance members may be used on values of an alias type and static members may be used on the alias type just like extension block members would.  

Inside an alias member, `this` denotes the current alias-typed value:
```cs
alias CustomerId : int
{
    public bool IsTest => this < 0; // `this` is a `CustomerId`
}
```

The above member is a shorthand for:
```cs
extension(CustomerId id) // note: parameter identifier provided for clarity only
{
    public bool IsTest => id < 0;
}
```

Alias members follow the same applicability model as extension members.

The extension eligibility rule in [§12.8.10.3](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/expressions.md#128103-extension-method-invocations) is updated as follows.

- An implicit identity, **alias,** reference, or boxing conversion exists from the receiver expression to the type of the first parameter of `Me`.

Therefore extension members declared for the underlying type are applicable to values of the alias type:
```cs
static class IntExtensions
{
    extension(int value)
    {
        public bool IsNegative => value < 0;
    }
}

CustomerId customerId = (CustomerId)42;
bool isNegative = customerId.IsNegative; // ok: `CustomerId` converts to `int`
```

An alias member declared for `ProductId` should not become applicable to a `CustomerId` or an `int` receiver merely because they all use `int` at runtime:

```cs
alias CustomerId : int;

alias ProductId : int
{
    public string Format() => $"product:{this}";
}

CustomerId customerId = (CustomerId)42;
customerId.Format(); // error: `ProductId.Format` is not applicable to `CustomerId`

int intValue = 42;
intValue.Format(); // error: `ProductId.Format` is not applicable to `int`
```

### Signatures and overloading

The signature comparison rules in [§7.6](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/basic-concepts.md#76-signatures-and-overloading) determine which overloads may coexist.

For signature comparison only, an alias type is not distinguished from its underlying type.  
Therefore, members declared in a single type whose signatures differ only by replacing an alias type with its underlying type are not allowed.

```cs
alias CustomerId : int;

void Load(int id);
void Load(CustomerId id); // error
```

```cs
alias OrderId : int;

void Load(CustomerId id);
void Load(OrderId id); // error
```

### Conversions

For an alias type `A` with underlying type `U`, the predefined alias conversions are:
- From `A` to `U` is an implicit conversion.
- From `U` to `A` is an explicit conversion.

TODO: Should the implicit conversion from an alias type to its underlying type be a standard implicit conversion? This is observable in scenarios that permit standard implicit conversions before or after another conversion, such as user-defined conversions:

```cs
struct DatabaseId
{
    public static implicit operator string(DatabaseId id) => id.ToString();
}

alias CustomerId : DatabaseId;

CustomerId customerId = ...;
string text = customerId; // TODO: ok if `CustomerId` to `DatabaseId` is a standard implicit conversion before the user-defined conversion to `string`?
```

Constants and literals do not convert to alias types without an explicit alias conversion.

```cs
alias CustomerId : int;
alias OrderId : int;

CustomerId customerId = (CustomerId)42;
int raw = customerId;

OrderId orderId = customerId; // error
OrderId orderId2 = (OrderId)(int)customerId;

CustomerId id1 = 42;             // error
CustomerId id2 = (CustomerId)42; // ok

long rawLong1 = customerId;      // ok: `CustomerId` converts to `int`, then `int` converts to `long`
long rawLong2 = (long)customerId;

CustomerId id3 = (CustomerId)1L;       // TODO: allowed through explicit numeric conversion to `int`, then explicit alias conversion?
CustomerId id4 = (CustomerId)(int)1L;

void M<T>(T t) where T : int { ....}
M(customerId); // ok: `CustomerId` satisfies constraints satisfied by `int`
```

TODO: review below
Conversions from an alias type compose with conversions from its underlying type. For example, `CustomerId` converts implicitly to `long` because `CustomerId` converts implicitly to `int` and `int` converts implicitly to `long`.

TODO: Decide whether conversions to alias types compose with conversions to the underlying type. For example, if `CustomerId` is an alias for `int`, should a cast from `long` to `CustomerId` be permitted directly because `long` has an explicit numeric conversion to `int`, or should the conversion be written through the underlying type?

TODO: how much should an underlying value behave like an alias value? In terms of members, operators, conversions?

TODO: there's still some issues. Normally, a maximum of three conversions can stack: a standard conversion, a user-defined conversion and another standard conversion. By introducing a new conversion that also a standard conversion, are we going to hit weird walls?

### Member lookup on alias-typed values

An alias-typed value should expose:

- Members declared by the alias.
- Applicable extension members for the alias type.
- Depending on the conversion design, members of the underlying type either directly or via conversion.

For example, if `EmailAddress` is an alias over `string`, the language must decide whether `email.Length` binds directly to `string.Length`, requires conversion to `string`, or is available only when explicitly surfaced by the alias.

The discussion favored an erased alias philosophy, but did not settle all member-lookup consequences. A permissive design makes persistent aliases lightweight and ergonomic. A restrictive design makes alias boundaries stronger and avoids accidentally treating an alias type as just its representation.

### Operators

Persistent aliases should be able to use operations available on their underlying type inside alias members:

```cs
alias CustomerId : int
{
    public bool IsTest => this < 0;
}
```

The design must decide which of those operations are also available at use sites:

```cs
CustomerId id = ...;
bool test = id < 0; // allowed directly, allowed via conversion, or disallowed?
```

User-defined operators declared by an alias are also possible, but the precise declaration form and overload-resolution behavior are not yet specified.

### Generics and constraints

Because persistent aliases are source-level types, generic code can mention them as type arguments:

```cs
List<CustomerId> ids = ...;
```

The runtime representation of closed generic types involving persistent aliases needs careful specification. If alias types fully erase, then `List<CustomerId>` and `List<int>` may have the same runtime representation, but they must remain distinct to the C# compiler for type-checking purposes. This likely requires alias metadata on generic instantiations, or a restriction on where alias types can appear until such metadata is designed.

Constraints also need definition. An alias over `int` might satisfy constraints based on the alias type, the underlying type, both, or a new set of alias-specific rules.

### Nullable annotations

Persistent aliases over reference types need nullable semantics:

```cs
alias EmailAddress : string;

EmailAddress email;
EmailAddress? maybeEmail;
```

The likely model is that nullability composes with the alias type, while the underlying representation remains the same as the annotated underlying type. Details for nullable flow analysis, oblivious contexts, and annotations emitted for metadata consumers remain open.

### Pattern matching

Persistent aliases should be usable in source patterns only where the alias test can be defined without pretending that the alias type has runtime identity:

```cs
if (value is CustomerId id)
{
    ...
}
```

Because persistent aliases are erased, a runtime type test cannot distinguish `CustomerId` from `int`. The language must decide whether alias patterns are compile-time-only conversions, are disallowed in runtime type-test positions, or lower to tests against the underlying type plus an alias assertion.

### Reflection, attributes, and metadata

Since persistent aliases erase to the underlying representation, reflection over ordinary runtime values would normally observe the underlying type. The source compiler may still need attributes or custom metadata to recover alias information from assemblies.

Possible metadata needs include:

- Identifying an alias declaration and its underlying type.
- Marking alias-typed parameters, returns, fields, locals, properties, and generic arguments.
- Marking lowered alias members and their receiver parameter.
- Supporting documentation and IDE navigation from source alias declarations to lowered members.

The discussion specifically favored avoiding a synthesized name for the lowered receiver parameter. If metadata supports unnamed parameters for the chosen lowering shape, the receiver should be emitted unnamed.

## Drawbacks
[drawbacks]: #drawbacks

## Alternatives
[alternatives]: #alternatives

As mentioned earlier, developers can already define wrapper types and type aliases, but these have drawbacks.

## Open questions
[open]: #open-questions

### Interfaces and inheritance

Can persistent aliases implement interfaces? Can an alias type be constrained by interfaces implemented by the underlying type? Can aliases compose with each other, or can one alias be based on another alias?

### Generics

How should `List<CustomerId>` be represented and distinguished from `List<int>`? Can alias types appear in all generic type argument positions? Which constraints do alias types satisfy?

### Metadata representation

What metadata is required for public APIs involving persistent aliases? Can existing metadata represent all necessary source-level information, including unnamed receiver parameters for lowered alias members?

### Versioning

Is changing a parameter from `int` to `CustomerId` a source breaking change, a binary breaking change, both, or neither? What about changing an alias type's underlying type?

### Pattern matching and `is`/`as`

Should runtime type tests observe alias types or only underlying types? Since persistent aliases erase, `is CustomerId` likely cannot be a normal runtime type test without compiler-specific metadata rules.

### Dynamic

How do alias-typed values behave when converted to or from `dynamic`? Does alias identity disappear at the dynamic boundary?

### Nullable reference types

How exactly do nullable annotations and flow analysis apply to persistent aliases over reference types?

### Tooling and documentation

How should IDEs display alias-typed symbols whose metadata representation is the underlying type?
How should XML documentation, Go To Definition, signature help, and generated API docs preserve the alias abstraction?
