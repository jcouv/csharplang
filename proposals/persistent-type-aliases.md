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

TODO add one-line summary of conversion rule, as most behavior falls out of it
TODO The guiding principle is that anything that can be done with a value of `U` (underlying type) can also be done with a value of `A` (alias type on `U`).

## Motivation
[motivation]: #motivation

Common types are often used to represent distinct domain concepts:

```cs
void Transfer(int fromAccountId, int toAccountId, decimal amount);
void Move(double distanceInMeters, double altitudeInMeters);
void SendEmail(string address, string subject, string body);
```

It's possible to accidentally pass an order id for a customer id, to mix values of different units, or to mix trusted and untrusted strings. 

A possible solution is to use wrapper structs or records:

```cs
readonly record struct CustomerId(int Value);
readonly record struct OrderId(int Value);
```

That pattern works, but it has costs:

- It introduces a new runtime type and representation, even when the desired model is purely a compile-time distinction.
- It requires forwarding or re-declaring common operations from the underlying type.
- It affects serialization, reflection, interop, generic constraints, default values, and overload resolution as a normal wrapper type.
- It is verbose enough that many APIs keep using primitives instead.

Another possible solution is to use type aliases:

```cs
using CustomerId = int;
```

This improves readability but:
- It must be repeated from file to file.
- It does not prevent mixing a `CustomerId` with any other `int`.

The goal of persistent type aliases is to treat alias types as distinct types from the point of view of the language, but that are erased to the underlying type.  
This is similar in spirit to [branded types](https://github.com/microsoft/TypeScript/wiki/FAQ#can-i-make-a-type-alias-nominal) in TypeScript, [opaque type aliases](https://docs.scala-lang.org/scala3/book/types-opaque-types.html) in Scala 3, or [newtypes](https://www.haskell.org/onlinereport/haskell2010/haskellch4.html#x10-710004.2.3) in Haskell.

## Detailed design
[design]: #detailed-design

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

### Alias type declarations

An alias type declaration specifies its underlying type:

```cs
alias CustomerId : int;
```

The identifier `CustomerId` is a type name in source.

The type after `:` is the alias type's *direct underlying type*, which may itself be an alias type.  
The *ultimate underlying type* of an alias type is obtained by following the chain of direct underlying types until reaching a type that is not an alias type.  
For `alias A1 : int;` and `alias A2 : A1;`, the direct underlying type of `A2` is `A1` and its ultimate underlying type is `int`.

The chain of direct underlying types shall be acyclic. For example, `alias A1 : A2;` together with `alias A2 : A1;` is an error.

The underlying type shall be at least as accessible as the alias type itself. This ensures that the underlying type is usable everywhere the alias type is:

```cs
public alias CustomerId : int;          // ok: `int` is public
internal class C { }
public alias Handle : C;                // error: `C` is less accessible than `Handle`
```

The base classes rules in [15.2.4.2](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/classes.md#15242-base-classes) are augmented with:

> **A base class cannot be a alias type on its own.**

```cs
class Base;
alias Alias : Base;
class Derived : Alias; // error (otherwise the member lookup on Derived would have to interleave instance and extension members)
```

Two alias declarations with the same underlying type are distinct source-level types:

```cs
alias CustomerId : int;
alias OrderId : int;

CustomerId customerId = ...;
OrderId orderId = ...;

LoadCustomer(customerId); // ok
LoadCustomer(orderId);    // error
```

TODO: what are the restrictions on underlying type? this depends on metadata encoding

TODO: define which characteristics an alias type inherits from its underlying type, and which instance constructors an alias type is considered to declare. This determines how an alias type behaves with respect to the `new()` constraint and object creation.

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
extension(CustomerId id) // note: extension parameter name provided for clarity only (but would not be emitted)
{
    public bool IsTest => id < 0;
}
```

Alias members follow the same applicability model as extension members.

The extension eligibility rule in [§12.8.10.3](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/expressions.md#128103-extension-method-invocations) is updated as follows.

> - An implicit identity, **alias,** reference, or boxing conversion exists from the receiver expression to the type of the first parameter of `Me`.

Therefore extension members declared for the underlying type are applicable to values of the alias type:
```cs
CustomerId customerId = ...;
bool isTest = customerId.IsTest;
```

### Signatures and overloading

The signature comparison rules in [§7.6](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/basic-concepts.md#76-signatures-and-overloading) determine which overloads may coexist.  
They remain unchanged, allowing for overloads differing by alias types only:

```cs
alias CustomerId : int;

void Load(int id);
void Load(CustomerId id); // ok
```

```cs
alias OrderId : int;

void Load(CustomerId id);
void Load(OrderId id); // ok
```

TODO confirm whether the metadata erasure can support this (modopt). If not, then we'll have to disallow

### Conversions

The predefined alias conversions are:
- An implicit conversion exists from an alias type `A` to underlying types (direct or indirect) of `A`.
- An explicit conversion exists from a type `U` to every alias type `A` that has underlying type `U` (direct or indirect).

```cs
class Animal { }
class Dog : Animal { }

alias Creature : Animal;  // base type has one alias
alias Pet : Dog;          // derived type has two chained aliases
alias Companion : Pet;

Companion companion = ...;
Pet pet = companion;            // implicit: `Companion` is an alias over `Pet`
Dog dog = companion;            // implicit: `Companion` is an alias over `Dog`

Companion c1 = (Companion)dog;  // explicit: `Companion` is an alias over `Dog`
Pet p1 = (Pet)dog;              // explicit: `Pet` is an alias over `Dog`
Companion c2 = (Companion)pet;  // explicit: `Companion` is an alias over `Pet`

Creature creature = ...;
Pet pet2 = (Pet)creature;       // error: neither `Creature` nor `Pet` is an alias over the other
Creature creature2 = (Creature)pet; // error: same

Pet pet3 = (Pet)(Animal)creature;    // ok: cast to the shared base `Animal` first
Creature creature3 = (Creature)(Animal)pet; // ok
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

#### Standard conversions

TODO the implicit conversion from an alias type to its underlying types should be standard (we should update https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/conversions.md#104-standard-conversions) but this needs to be done conversion by conversion (ie. reviewed carefully)

TODO Not sure how to best model conversion... The current approach seems like it will lead to new conversion permutations. Alternatively, we could model the new conversion as a kind of reference conversion. But that would only help for aliases over reference types...

Scenarios:
- literals (42)
- numeric conversions (int -> long, long -> int)

### Member lookup on alias-typed values

Member lookup on an alias-typed value proceeds in two phases:

1. Instance members of the ultimate underlying type come first: Lookup on a value whose type is an alias `A` first considers the instance members of `A`'s *ultimate underlying type* `U` (and `U`'s base hierarchy), exactly as if the receiver had type `U`. As with extension members generally, the alias-declared members in the second phase are only considered when this phase finds no applicable instance member.
2. Alias-declared members then come into play as extensions: Each level of the alias and inheritance chain contributes its declared members as extension members whose receiver parameter is that level's alias type. All levels contribute candidates regardless of which level the static receiver type names.

The existing [betterness rules](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/expressions.md#12647-better-conversion-target) result in a more specific type being preferred: an alias type is a better conversion target than any of its underlying types or its base types.

### Operators

The behavior of operators falls out of other rules.

When resolving non-extension operators, the non-extension operators of the underlying type are considered operators on the alias type:

```cs
alias CustomerId : int
{
    public bool IsTest => this < 0; // `this` is a `CustomerId` so gets the `<` operator from `int`
}
```

When resolving extension operators, the operators defined in alias types contribute to the candidate set like all extension operators.

```cs
class Identifier;
alias CustomerId : Identifier
{
    public static bool operator<(CustomerId id1, CustomerId id2) { /* which one is oldest? */ }
}

CustomerId id1 = ...;
CustomerId id2 = ...;
bool isOlder = id1 < id2;
```

Binary operation are possible across alias types that share an underlying type:

```cs
alias CustomerId : int;
alias ProductId : int;

CustomerId customerId = ...;
ProductId productId = ...;
int result = customerId + productId; // built-in operator+(int, int)
```

TODO Is there something we could do to avoid this situation with binary operators?

### Type categories

An alias type belongs to the same type category as its underlying type. This classification is the foundation for the constraint behavior below: the reference type, value type, and unmanaged type constraints all inherit from it rather than naming alias types separately.

The reference type definition in [§8.2.1](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/types.md#821-general) is updated as follows:

> A reference type is a class type, an interface type, an array type, a delegate type, the `dynamic` type **, or an alias type whose underlying type is a reference type**.

The value type definition in [§8.3.1](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/types.md#831-general) is updated as follows:

> A value type is either a struct type, an enumeration type **, or an alias type whose underlying type is a value type**.

The unmanaged type definition in [§8.8](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/types.md#88-unmanaged-types) is updated as follows:

> An *unmanaged_type* is one of the following:
>
> - `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `char`, `float`, `double`, `decimal`, or `bool`.
> - Any *enum_type*.
> - Any user-defined *struct_type* that is not a constructed type and contains instance fields of *unmanaged_type*s only.
> - **Any alias type whose underlying type is an *unmanaged_type*.**

### Declaring constraints

Alias types can be used in type parameter constraints:

```cs
where T1 : A
where T2 : List<A>
```

The type parameter constraint rules in [§15.2.5](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/classes.md#1525-type-parameter-constraints) are updated as follows.

For the purposes of those rules, an alias type used as a type parameter constraint is *classified* according to its ultimate underlying type:
- it is a primary or secondary constraint, a *class_type* or *interface_type* constraint, exactly as its ultimate underlying type would be, and
- it contributes to the effective base class and the effective interface set exactly what its ultimate underlying type would.

Consequently the primary-constraint, secondary-constraint, *interface_type*-constraint, accessibility, effective-base-class, and effective-interface-set rules need no further changes for alias types.

Note: Because an alias type is classified according to its underlying type, an alias whose underlying type is a class type is a *class_type* constraint. The existing rule that at most one constraint may be a class type therefore counts such an alias as a class type. For instance, given `alias A : C;`, both `where T : C, A` and `where T : A1, A2` (with `alias A1 : C1;` and `alias A2 : C2;`) are errors.

Note: Because an alias type contributes its underlying type's dynamic erasure to the effective base class, a type parameter constrained to an alias whose underlying type is a class type is *known to be a reference type*, just as if it had been constrained to that class type directly.

> A *class_type* constraint shall satisfy the following rules:
>
> - The type shall be a class type.
> - ~~The type shall not be `sealed`.~~
> - The type shall not be one of the following types: `System.Array` or `System.ValueType`.
> - The type shall not be `object`.
> - At most one constraint for a given type parameter may be a class type.

Note: the removal of this restriction allows:
```cs
void M<T>(T t) where T : string { }
alias CustomerId : string;

M<CustomerId>((CustomerID)"Mads");
```
TODO this bothers me somewhat because one would have simply written `void M(string s)` and not bothered with a generic method.

We're not modifiying the interface_type constraint rules, so the following examples are allowed:

```cs
alias A : I;
alias A2 : A;

class C<T> where T : A, I { }
class D<T> where T : A2, A { }
```

```cs
interface I<T>;
class U;
alias A : U;
alias A2 : A;

class C<T> where T : I<A>, I<U> { }
class D<T> where T : I<A2>, I<A> { }
```

#### Satisfying constraints

Alias types can be used as type arguments:

```cs
List<CustomerId> ids = ...;
```

When an alias type is used as a *constraint*, the convertibility check in the first bullet of [§8.4.5](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/types.md#845-satisfying-constraints) (for a *class_type*, *interface_type*, or *type_parameter* constraint) is performed against the constraint type as written, not its underlying type. For example, given `alias A : C;`, the constraint `where T : C` is satisfied by both `C` and `A` (because `A` implicitly converts to `C`), whereas `where T : A` is satisfied by `A` but not by `C` (because `C` only *explicitly* converts to `A`).

With chained aliases `alias A1 : int;` and `alias A2 : A1;`, this asymmetry applies at each level: `where T : A1` is satisfied by `A2` and `A1` (each implicitly converts to `A1`) but not by `int`, and `where T : A2` is satisfied only by `A2`.

The satisfying-constraints checks for the reference type constraint and the value type constraint in [§8.4.5](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/types.md#845-satisfying-constraints) classify the supplied type *argument* `A`. These lists re-state the reference type  and value type categories inline, so they are updated to account for alias type arguments as follows:

> - If the constraint is the reference type constraint (`class`), the type `A` shall satisfy one of the following:
>   - `A` is an interface type, class type, delegate type, array type**,** or the dynamic type**, or an alias type whose underlying type satisfies the reference type constraint**.
>   - `A` is a type parameter that is known to be a reference type.
> - If the constraint is the value type constraint (`struct`), the type `A` shall satisfy one of the following:
>   - `A` is a `struct` type or `enum` type**, or an alias type whose underlying type satisfies the value type constraint,** but not a nullable value type.
>   - `A` is a type parameter having the value type constraint.

The constructor constraint `new()` requires no change for alias type arguments, since an alias type inherits its underlying type's abstractness and constructors.

```cs
class HasCtor { }
abstract class IsAbstract { }
alias Id : int;
alias Handle : HasCtor;
alias Marker : IsAbstract;

void M<T>() where T : new() { }
M<Id>();      // ok: underlying type `int` is a value type
M<Handle>();  // ok: underlying type `HasCtor` is non-abstract with a default constructor
M<Marker>();  // error: underlying type `IsAbstract` is abstract
```

### Nullable annotations

Persistent aliases over reference types need nullable semantics:

```cs
alias EmailAddress : string;

EmailAddress email;
EmailAddress? maybeEmail;
```

The likely model is that nullability composes with the alias type, while the underlying representation remains the same as the annotated underlying type. Details for nullable flow analysis, oblivious contexts, and annotations emitted for metadata consumers remain open.

The default value of an alias type is the default value of its underlying type, and `default` produces it.

### Pattern matching

Because persistent aliases are erased, a runtime type test cannot distinguish an alias type from its underlying type.
But alias types in patterns give access to extension properties. 
TODO we need to discuss this more. The type pattern may be misleading, but making it error or warn seems harsh/annoying.

```cs
object value = ...;
if (value is CustomerId id) { ... }         // disallowed (or warning): cannot test for an alias type at run time
if (value is List<CustomerId> list) { ... } // disallowed (or warning): same
if (value is int i) { ... }                 // ok: test against the underlying type
```

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

This proposal uses transparent rather than opaque alias types, which doesn't allow an alias to remove or replace a member of the underlying type.  
This proposal doesn't allow restricting the set of constructed values for an alias type. For example, `alias PositiveInt : int;`.
This proposal uses erasable alias types, so runtime type detection isn't possible.

## Alternatives
[alternatives]: #alternatives

As mentioned earlier, developers can already define wrapper types and type aliases, but these have drawbacks.

## Open questions
[open]: #open-questions

TODO implicit conversion from underlying type weakens the safeguard
TODO not being able to overload on alias differences may be a problem
TODO should we prevent `customerId + orderId`?

### Restrictions on underlying types?

Can one alias be based on another alias?

```csharp
alias Base : int;
alias Derived : Base;
```

Could you get an alias loop?

Some restrictions may come from the metadata encoding (TODO).
TODO can a pointer type be an underlying type?
TODO disallow nullable reference type as underlying type

### Metadata representation

What metadata is required for public APIs involving persistent aliases? Can existing metadata represent all necessary source-level information, including unnamed receiver parameters for lowered alias members?

### Versioning

Is changing a parameter from `int` to `CustomerId` a source breaking change, a binary breaking change, both, or neither? What about changing an alias type's underlying type?

### Dynamic

Resolved: alias types are erased at run time, so alias identity disappears at the `dynamic` boundary. Converting an alias-typed value to `dynamic` and back observes only the underlying type, and member binding through `dynamic` sees the underlying type's members, not alias members.

### Attributes

How do alias types interact with attributes?
- May an alias type be used as an attribute argument type, or as the operand of `typeof` in an attribute argument, given that it erases to its underlying type?
- May an alias declaration itself carry attributes, and how are they encoded so a capable compiler can recover them?

### Conversion classification and composition

Is the implicit conversion from an alias type to a type it is an alias over a *standard* implicit conversion? This is observable when alias values participate around user-defined conversions, which permit a standard conversion before and after the user-defined operator.

Relatedly, do conversions to and from alias types compose with conversions to the underlying type? For example, should a cast from `long` to `CustomerId` (with `alias CustomerId : int`) be permitted directly because `long` has an explicit numeric conversion to `int`, or must it be written through the underlying type? Introducing a new standard conversion interacts with the existing limit of at most one standard conversion on each side of a user-defined conversion, and the consequences need to be worked through.

### Definite assignment

How does definite assignment apply to alias types, in particular the difference between an alias over a struct type and an alias over a class type?

### Nullable reference types

How exactly do nullable annotations and flow analysis apply to persistent aliases over reference types?

### Tooling and documentation

How should IDEs display alias-typed symbols whose metadata representation is the underlying type?
How should XML documentation, Go To Definition, signature help, and generated API docs preserve the alias abstraction?

### Baseline standard sections needing attention
TODO
A review of the ECMA-334 baseline against this proposal surfaced the following sections that need new or revised normative text. Sections the proposal already edits are marked *(edited; verify)*. Cross-cutting drivers: (a) erasure vs. runtime identity, (b) integrating "alias conversions" into the conversion taxonomy, (c) operator availability on alias operands, (d) the alias body depends on full *extension members* (the baseline has only classic extension methods), (e) syntactic *type classification* predicates (struct/enum/array/delegate/interface type) vs. the proposal's *type category* (reference/value/unmanaged), (f) nullability over reference-type aliases.

- **Types (§8)**: §8.1 place aliases in the type taxonomy; §8.3.3/§8.3.13 default ctor & boxing of alias-over-struct; §8.3.12 alias over `int?`; §8.4.3 open/closed alias over a type parameter; §8.6/§8.7 expression trees & `dynamic` boundary; §8.8 *(edited; verify chaining)*; §8.9 nullable annotations over reference-type aliases.
- **Conversions (§10)**: §10.2.1/§10.3.1 add alias conversions to the implicit/explicit lists; §10.2.2 distinguish alias conversion from identity; §10.4.2/§10.4.3 is alias→underlying a *standard* conversion; §10.5.3–§10.5.5 the standard+user-defined+standard composition limit; §10.2.7/§10.6.1 null-literal & nullable composition.
- **Basic concepts (§7)**: §7.1 alias-over-`int` as `Main` return; §7.3/§7.4 enumerate alias declarations and "members of an alias type"; §7.5.2 permitted accessibilities; §7.5.5 & §7.6 *(edited; verify constructed types & chained aliases)*.
- **Expressions (§12)**: §12.4.3–§12.4.8 operator candidacy/promotion/lifting for alias operands; §12.5.1–§12.5.2 two-phase member lookup & "base types" of an alias; §12.6.3/§12.6.4 inference preserving alias identity & receiver betterness for chained aliases; §12.8.7 member access; §12.8.14/§12.8.15 `this`/`base` in alias members; §12.8.17 candidate constructors; §12.8.18 `typeof` under erasure *(critical)*; §12.8.19/§12.8.21/§12.8.23 `sizeof`/`default`/`nameof`; §12.9.8/§12.11/§12.14 cast & `is`/`as`/patterns under erasure; §12.17/§12.18 best-common-type across alias vs. underlying; §12.19/§12.20/§12.21 lambdas, query, compound assignment.
- **Classes (§15)**: §15.2.2 exclude `abstract`/`sealed`/`static`; §15.2.4 class deriving from an alias / alias-over-interface in base lists; §15.2.5 *(edited; verify)*; §15.3.1/§15.3.4/§15.3.7/§15.3.8 alias members are not class members & don't inherit; §15.6.10 *(blocking)* needs full extension members, not just methods; §15.10 user-defined/conversion operators vs. predefined alias conversions; §15.11 which constructors an alias declares.
- **Namespaces (§14)**: §14.4/§14.5.2 terminology clash with using-alias directives & lookup precedence; §14.6/§14.7 *(edited; verify)* prose still lists 5 type kinds, nesting/scope; §14.8 alias type vs. `::` qualifier.
- **Patterns (§11)**: §11.2.1 add alias conversions to pattern-compatibility & resolve erasure; §11.2.2–§11.2.4 runtime type test, constant-to-alias, `var` infers alias; §11.3/§11.4 subsumption & exhaustiveness (`case CustomerId` vs. `case int`).
- **Grammar**: add `alias_type_declaration`/`alias_modifier`/`alias_type_body` to `type_declaration`; *(blocking)* `extension_member_declaration` is absent from the consolidated grammar; verify `alias` contextual-keyword disambiguation.
- **Variables (§9)**: §9.3 default of alias = default of underlying; §9.4 definite assignment for alias-over-struct vs. -class; §9.6 atomicity for alias-over-primitive; §9.7 `ref`-to-alias.
- **Statements (§13)**: §13.6 alias locals/constants & `var`; §13.9.5 `foreach` enumerator/element resolution; §13.10.5/§13.15 `return`/`yield` underlying→alias is explicit-only; §13.11/§13.14/§13.13 `using`/`lock` over alias.
- **Type-specific pages**: structs §16.4 (ValueType inheritance, boxing, `this`, ref/readonly-struct underlying); enums §20.1/§20.6 (enum operators; not an "enum type"); arrays §17.2/§17.4/§17.6 (array type classification, Index/Range, covariance); delegates §21.2/§21.5/§21.6 (`System.Delegate`, `new`, `+`/`-`, invocation); ranges §18 (Index/Range pattern support).
- **Unsafe code (§23/§24)**: meaning of `unsafe alias`; alias-over-unmanaged as pointer element type & `T* ↔ alias*`; `fixed`/fixed-size buffers/`stackalloc`/`sizeof` admitting aliases.
- **Lexical & interfaces**: §6.4.4 `alias` contextual-keyword disambiguation; §19.2.4/§19.6.2/§19.6.5 alias-over-interface in base lists, explicit-implementation qualifier, interface mapping.
- **Library & portability**: Annex C a new required `System.Runtime.CompilerServices` attribute to round-trip alias info plus `System.Type`/reflection guidance; portability annex note implementation-defined alias metadata/reflection surfacing.
