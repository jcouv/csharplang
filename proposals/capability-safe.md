# Capability-safe code

Champion issue: https://github.com/dotnet/csharplang/issues/10217

## Summary
[summary]: #summary


In a system with strict encapsulation, memory safety and no ambient authority, the object graph and design of the APIs for those objects determines which capabilities are available to an object.  
This follows the principle of least authority, where objects have no capabilities by default and are sandboxed by the design of the object graph.  
This contrasts with current C# programs where any misbehaving object has as much access as the process.

This proposal allows types or static members to be declared **capability-safe**, a contract enforced by the runtime and the compiler.  
Capability-safe code is limited to a subset of C#.  
It may only perform operations through explicit object references it already has, such as parameters, fields reachable from `this` or objects returned by those references.  
It cannot use escape hatches that might violate encapsulation or memory safety (pointers, native interop, reflection, `dynamic`) or static APIs not certified capability-safe.

### Illustration

As the following (valid) code illustrates, a capability-safe method may exercise arbitrarily powerful capabilities such as reading or writing a file through a reference that it is given:
```csharp
public interface IFile
{
    string ReadAllText();
    void WriteAllText(string contents);
}

[CapabilitySafe]
public static string ReadConfig(IFile configFile)
{
    return configFile.ReadAllText(); // valid
}
```

This capability-safe method can perform I/O through `configFile`.

If the caller doesn't directly or indirectly expose a reference for a certain capability, it is not possible for the capability-safe code to get it. For instance, an implementation that uses ambient authority, such as a static method that isn't capability-safe, causes a compiler error and a runtime type load error:

```csharp
[CapabilitySafe]
public static string ReadConfig()
{
    return File.ReadAllText("config.json"); // error
}
```

It is the responsibility of the caller to hand out capabilities that are restrained (not more powerful than necessary):
```csharp
public interface IFileSystem
{
    string ReadAllText(string path);
    void WriteAllText(string path, string contents);
    void Delete(string path);
}

[CapabilitySafe]
public static string ReadConfig(IFileSystem fileSystem)
{
    ...
}

IFileSystem fileSystem = ...;
ReadConfig(fileSystem); // the capability is too broad as it exposes arbitrary path access
```

The most direct way to restrain a capability is to use encapsulation to attenuate a more powerful capability:

```csharp
public sealed class ReadOnlyFile : IFile
{
    private readonly IFile _inner;

    public ReadOnlyFile(IFile inner)
    {
        _inner = inner;
    }

    public string ReadAllText()
    {
        return _inner.ReadAllText();
    }

    public void WriteAllText(string contents)
    {
        throw new NotSupportedException();
    }
}

IFile file = GetFile();
ReadOnlyFile readOnlyFile = new ReadOnlyFile(file);
ReadConfig(readOnlyFile); // implementation of ReadOnlyFile restricts write access and encapsulates underlying read/write file capability
```

Such a `ReadOnlyFile` wrapper attenuates an `IFile` capability by preserving read access and refusing write access. This provides fine-grained control over what capabilities `ReadConfig` can exercise.

A caller may require that an API be capability-safe. This can be achieved by using the capability-safe contract within trusted code:
```csharp
[CapabilitySafe]
static string ReadConfig(IFile file)
{
    return External.ReadConfig(file);
}
```

Similarly, the capability-safe contract can be used within trusted code to require that dynamically-loaded code is capability-safe. Consider a host application that defines the extension contract:
```csharp
[CapabilitySafe]
public interface IExtension
{
    int CountLines(IFile input);
}
```

The caller can dynamically load an extension implementing this interface with the guarantee that it has access to no capability except for the file it is given a reference to:

```csharp
IExtension extension = LoadExtension(path); // note: the runtime ensures that the loaded extension types satisfy the required capability-safe contract
if (extension != null)
{
    ReadOnlyFile readOnlyFile = GetReadOnlyFile();
    int count = extension.CountLines(readOnlyFile); // note: untrusted extension is limited in the damage it can do. Unable to overwrite or exfiltrate the file through the network, for example.
}
```

In summary, code that can be verified to be capability-safe starts with no capabilities and the caller controls the extent of capabilities exposed.  

## Motivation
[motivation]: #motivation

Currently, any part of the program executes with the full authority of the process.
This makes software composition dramatically riskier and increases the impact of supply-chain attacks, confused agents and compromised plugins.
Mitigations are complex and add significant overhead.

Object-capability discipline, where authority flows through explicit and unforgeable object references only, allows granular and flexible control. Parts of the program are entrusted only with limited and specific capabilities.  
For instance, one object can have read-only access to a file while another object has write access to a directory.  
This also allows auditing of exposure. If a portion of the object graph is never directly or indirectly handed a reference to a specific object-capability, then we know that capability cannot be reached.

## Limitations
[limitations]: #limitations

With all the security problems that come with ambient access to files, network, and other resources through static APIs and configured permissions, such API design is undeniably convenient.
The object-capability discipline creates a burden of passing references around and attenuating capabilities in order to create a program that is both robust and useful.
Further language features, such as easier wrapper generation, could mitigate this burden.

Another downside is that existing APIs were not designed with object-capability discipline in mind. This means that new usage patterns and APIs may have to be developed.
Even if we simply annotate existing APIs conservatively, the contract can be adopted incrementally. For example, only dynamically-loaded extensions might be subject to the object-capability discipline, while helpers can be developed locally (application-specific `Directory` or `File` abstractions, for example) or in dedicated packages or namespaces where shared implementations are useful.

This proposal does not address build-time supply-chain attacks.

Finally, it remains possible for a misbehaving object to exhaust resources by consuming CPU, allocating memory in a loop, blocking indefinitely or exhausting the stack.

## Detailed design
[design]: #detailed-design

The proposal introduces a `CapabilitySafe` contract declared with an attribute on types (including interfaces) and on static methods.  
When applied to a non-interface type, the contract covers the type's member bodies.  
When applied to an interface, the contract requires implementing types (and their members) to satisfy the contract.  
When applied to a static method, the contract covers that method body as a standalone entry point.

The runtime and the C# compiler both enforce that types and members declared capability-safe abide by the capability-safe contract.
Although only the runtime certification matters technically (the attribute is a claim which must not be trusted, the runtime must verify it), the compiler enforcement and diagnostics help good faith actors author capability-safe code.

--------------------------------------------
THE DETAILS BELOW ARE STILL WORK-IN-PROGRESS
--------------------------------------------

The C# source checks are intended to be an early and precise diagnostic layer. For example, within a capability-safe member the compiler reports an error for:

- pointer types, function pointer types;
- uses of static members that are not themselves capability-safe;
- construction through constructors that are not capability-safe;
- implementations of capability-safe interfaces or virtual members whose bodies do not satisfy the contract;
- dynamic dispatch, reflection-based activation or invocation, native interop, and other known ambient authority roots.

Instance members may be used since they use explicit object references. Constructors and static members may be used only when they are capability-safe.

A capability-safe member does not mint, discover, or obtain new authority except through object references already available to it.
It may not use ambient escape hatches such as statics, reflection, dynamic code, native interop, and unsafe pointer operations.

### IL/runtime verification

The runtime or loader must verify the actual IL and metadata for capability-safe members. Source checks and build-time verification are useful for diagnostics and early failure, but the trusted artifact is the implementation assembly.

The verifier checks two things:
1. the member body uses only IL instructions allowed by this specification; and
2. the member's IL and metadata do not contain pointer, reflection, or native escape hatches.

Capability-safe methods, types, fields, locals, signatures, generic constraints, custom modifiers, and metadata must not contain pointer types, function pointer types, reflection-related types, typed references, P/Invoke metadata, or native interop shapes.

The verifier should reject at least the following metadata escape hatches:

| Escape hatch | Metadata indication |
|---|---|
| P/Invoke methods | A method marked `pinvokeimpl`, `MethodAttributes.PinvokeImpl`, or a corresponding `ImplMap` row. |
| Runtime-provided native calls | A method marked `internalcall` or `MethodImplAttributes.InternalCall`, unless it is a runtime-owned member that is itself capability-safe. |
| Raw pointer types | Signatures, locals, fields, generic constraints, custom modifiers, or metadata containing `ELEMENT_TYPE_PTR`. |
| Function pointer types | Signatures, locals, fields, generic constraints, custom modifiers, or metadata containing `ELEMENT_TYPE_FNPTR`, including unmanaged calling-convention markers. |
| Reflection-related types | Signatures, locals, fields, generic constraints, custom modifiers, or metadata containing `System.Type`, `System.RuntimeTypeHandle`, `System.TypedReference`, or `System.RuntimeArgumentHandle`. Method bodies must not produce these values, including through `ldtoken` for a type handle. |
| Typed references | Signatures, locals, fields, generic constraints, custom modifiers, or metadata containing `ELEMENT_TYPE_TYPEDBYREF`, or method bodies using `mkrefany`, `refanytype`, or `refanyval`. |
| Native interop attributes | `DllImportAttribute`, `LibraryImportAttribute`, `UnmanagedCallersOnlyAttribute`, `SuppressGCTransitionAttribute`, marshalling attributes, COM interop attributes, WinRT activation/projection metadata, and similar attributes that expose native code, OS handles, or native marshalling. |

#### IL instruction classification

A capability-safe body may use only permitted IL instructions. Instructions that reference metadata are valid only when the referenced member, field, type, signature, override, or interface mapping satisfies the rules below. Instructions classified as forbidden are never valid in capability-safe code. Any instruction not classified by this table is forbidden.

The following instructions are permitted unconditionally, assuming the body is otherwise valid IL:

| Group | Instructions |
|---|---|
| Stack and return | `nop`, `dup`, `pop`, `ret` |
| Arguments and locals | `ldarg`, `ldarg.*`, `ldarga`, `ldarga.*`, `starg`, `starg.*`, `ldloc`, `ldloc.*`, `ldloca`, `ldloca.*`, `stloc`, `stloc.*` |
| Constants | `ldnull`, `ldc.i4`, `ldc.i4.*`, `ldc.i8`, `ldc.r4`, `ldc.r8`, `ldstr` |
| Arithmetic and bitwise operations | `add`, `add.*`, `sub`, `sub.*`, `mul`, `mul.*`, `div`, `div.*`, `rem`, `rem.*`, `neg`, `and`, `or`, `xor`, `not`, `shl`, `shr`, `shr.*` |
| Comparisons and conversions | `ceq`, `cgt`, `cgt.*`, `clt`, `clt.*`, `ckfinite`, `conv.*` |
| Control flow | `br`, `br.*`, `brtrue`, `brtrue.*`, `brfalse`, `brfalse.*`, `beq`, `beq.*`, `bne.un`, `bne.un.*`, `bge`, `bge.*`, `bge.un`, `bge.un.*`, `bgt`, `bgt.*`, `bgt.un`, `bgt.un.*`, `ble`, `ble.*`, `ble.un`, `ble.un.*`, `blt`, `blt.*`, `blt.un`, `blt.un.*`, `switch`, `leave`, `leave.*` |
| Exception flow | `endfilter`, `endfinally`, `throw`, `rethrow` |
| Type, object, and array operations | `box`, `unbox`, `unbox.any`, `castclass`, `isinst`, `newarr`, `ldelem`, `ldelem.*`, `readonly.ldelema`, `ldelema`, `stelem`, `stelem.*`, `ldlen`, `initobj`, `cpobj`, `ldobj`, `stobj`, `sizeof`, `ldtoken` |
| Indirect managed loads and stores | `ldind.*`, `volatile.ldind.*`, `stind.*`, `volatile.stind.*` |

The following instructions are permitted only with the specified checks:

| Group | Instructions | Required check |
|---|---|---|
| Calls and method handles | `call`, `tail.call`, `callvirt`, `tail.callvirt`, `constrained.callvirt`, `ldftn`, `ldvirtftn` | Instance method calls and instance method handles are valid. Static method calls and static method handles must target members that are capability-safe. |
| Object construction | `newobj` | The constructor must be capability-safe. |
| Fields | `ldfld`, `volatile.ldfld`, `ldflda`, `stfld`, `volatile.stfld`, `ldsfld`, `volatile.ldsfld`, `ldsflda`, `stsfld`, `volatile.stsfld` | Instance fields are valid. Static fields are forbidden. |

The following instructions are forbidden in capability-safe code:

| Group | Instructions |
|---|---|
| Native calls, stack allocation, raw memory, and method jumps | `calli`, `jmp`, `localloc`, `cpblk`, `initblk` |
| Varargs and typed references | `arglist`, `mkrefany`, `refanytype`, `refanyval` |
| Debugger, unmanaged-address, and reserved prefixes | `break`, `unaligned.`, `prefix*`, `prefixref` |

### C# enforcement

Instead of waiting to detect bad implementations at runtime, we want to have those same checks at compile-time.
Members that claim to be capability-safe should emit IL that the runtime will recognize as capability-safe.

#### Allowed authority sources

Within a capability-safe member, authority may be exercised through:

- `this`;
- instance fields reachable from `this`;
- explicit parameters;
- delegates supplied through parameters or fields;
- virtual, interface, or delegate dispatch on explicitly reachable receivers;
- objects returned from explicitly reachable receivers;
- local objects constructed by constructors that themselves satisfy the contract;
- static members, including platform static members, that themselves satisfy the contract.

Stashing object references in instance fields is allowed. The receiver is an explicit object reference, and object-capability design permits objects to retain capabilities they were constructed with.

Invoking an instance member on an explicitly reachable receiver is capability-safe even if the invoked member is not itself `CapabilitySafe`. For example, `[CapabilitySafe] void M(object o) { o.ToString(); }` is capability-safe: `ToString` is invoked only through the object reference supplied by the caller. The receiver, not the member name, is the capability.

Delegates are ordinary object references for the purpose of this contract. Invoking a delegate supplied through a parameter or reachable field is capability-safe because the callee is exercising authority it was explicitly given, not authority it discovered ambiently. `Delegate.Invoke` does not augment authority beyond the delegate object reference already held by the caller. Passing explicit capabilities onward, including through delegate calls, is not itself a violation. For example:

```csharp
[CapabilitySafe]
public static string Invoke(Func<string> action)
{
    return action();
}
```

This method is capability-safe even if the particular delegate instance passed by a caller reads a file, accesses the network, or invokes other authority-bearing code. In that case the authority is contained in the delegate object supplied by the caller. `CapabilitySafe` means the member does not acquire new authority except through references already available to it.

Constructing a delegate or lambda inside capability-safe code is checked like any other code in the same body or type. A lambda that closes over or calls a forbidden ambient root is rejected:

```csharp
[CapabilitySafe]
public sealed class Actions
{
    public static Func<string> ReadFile()
    {
        return () => File.ReadAllText("x"); // error
    }
}
```

#### Inheritance and interface implementation

A type that derives from a `[CapabilitySafe]` class must itself be `[CapabilitySafe]` and satisfy the contract.
Likewise, a type that implements a `[CapabilitySafe]` interface must itself be `[CapabilitySafe]` and satisfy the contract.

For example:

```csharp
[CapabilitySafe]
public interface IInterface
{
    void M();
}

public class C : IInterface // error: C must satisfy the capability-safe contract
{
    public void M()
    {
        ReadFile(); // error
    }
    private void ReadFile()
    {
        File.ReadAllText("secret.txt");
    }
}
```

#### Forbidden ambient roots

The exact forbidden set needs specification, but it should include at least:

- file-system and path authority, such as `File`, `Directory`, and temp/current-directory APIs;
- network and DNS authority;
- process creation and process inspection;
- environment variables, command-line state, user/machine identity, and application context switches;
- console, registry, event log, clipboard, credential stores, certificate stores, and other OS-global resources;
- ambient time, randomness, and culture, such as `DateTime.Now`, `DateTimeOffset.UtcNow`, `Stopwatch.GetTimestamp`, `Random.Shared`, `Guid.NewGuid`, and `CultureInfo.CurrentCulture`, unless exposed through explicit capability objects;
- reflection, type discovery, activation, assembly loading, and metadata-based invocation;
- `dynamic` and DLR-based dispatch;
- runtime code generation, expression compilation, scripting, and assembly load contexts;
- global service location, such as unrestricted `IServiceProvider` roots or static service locators;
- static mutable state that can serve as a hidden authority or communication channel;
- native interop, COM, WinRT activation, P/Invoke, `LibraryImport`, `NativeLibrary`, and marshalling APIs;
- unsafe pointer operations and unverifiable IL.

Static APIs are not categorically forbidden. Authority-free static APIs such as many `Math` and string operations can be marked or certified as `CapabilitySafe`. The distinction is between ambient static authority roots, such as `Console.Out`, `DateTime.Now`, `Random.Shared`, and `CultureInfo.CurrentCulture`, and authority-free computation.

#### Pointer, reflection-related, and unsafe code

The contract should be strict around pointers, reflection-related types, and unverifiable code. The following should be forbidden in capability-safe code:

- C# `unsafe` blocks;
- raw pointer types and operations;
- unmanaged function pointers;
- `System.Type`, `System.RuntimeTypeHandle`, `System.TypedReference`, and `System.RuntimeArgumentHandle`, whether used directly or through signatures, locals, fields, generic arguments, arrays, attributes, or other metadata;
- `System.Runtime.CompilerServices.Unsafe`;
- `Marshal`, `NativeMemory`, `GCHandle`, `SafeHandle`, and handle-like APIs unless explicitly marked or certified as `CapabilitySafe`;
- P/Invoke, `LibraryImport`, `NativeLibrary`, COM, and WinRT interop;
- the IL `calli` instruction;
- unverifiable IL.

Managed byref features such as `ref`, `in`, `out`, `ref struct`, `Span<T>`, and `ReadOnlySpan<T>` are not inherently authority-bearing and may be allowed if they remain verifiable and do not provide pointer or native escape hatches.

#### Static initialization

Static constructors, static field initializers, and module initializers are a major risk. Calling an apparently harmless static member may trigger type initialization that performs ambient effects. Runtime or loader enforcement must verify relevant static initialization before it can run as part of a `CapabilitySafe` dependency or entry point.

Options include:

- forbid static constructors and module initializers in certified assemblies;
- require static constructors and field initializers to be independently verified as `CapabilitySafe`;
- allow only static readonly data initialized from constants or verified authority-free code.

#### Scenario: dynamically loaded extension

Consider a host application that defines the extension contract and controls the capabilities that extensions receive:

```csharp
[CapabilitySafe]
public interface IExtension
{
    void Run(IFile input);
}
```

The host can dynamically load an extension assembly using ordinary plugin-loading code and invoke it with only the explicit capabilities it wants to provide:

```csharp
static IExtension LoadExtension(string path)
{
    Assembly assembly = AssemblyLoadContext.Default.LoadFromAssemblyPath(path);
    Type extensionType = assembly.GetTypes().Single(type => typeof(IExtension).IsAssignableFrom(type));

    return (IExtension)Activator.CreateInstance(extensionType)!;
}

IExtension extension = LoadExtension(path);
extension.Run(readOnlyInputFile);
```

An extension author can provide a type that implements the host-owned interface.
Because `IExtension` is marked as capability-safe, the implementation type must also be capability-safe and must satisfy the contract.  The compiler and runtime verification reject an implementation that claims to implement the capability-safe interface but violates the contract.
The extension cannot cheat by implementing `Run` with ambient access such as static `File` APIs for example.

So despite the extension not being fully trusted, the host can trust that:
1. the capability-safe contract declared in the host-owned interface is enforced, and thus
2. the plugin only has access to capabilities it was explicitly given.

### BCL impact

Platform APIs participate in the same model as user code. Platform instance members can be used through explicit object references, such as invoking `Delegate.Invoke` on a delegate supplied by the caller. Platform static members can be referenced from capability-safe code only if they are marked or certified as capability-safe; authority-free static members, such as many static `Math` APIs, can satisfy the contract because they do not augment authority. Ambient statics such as `Console.Out`, `DateTime.Now`, `Random.Shared`, and `CultureInfo.CurrentCulture` are not capability-safe.

## Open questions
[open]: #open-questions

- Delegates
- Expression trees

### Declaration, metadata, and verification boundary

- If applied to a type, should it also apply to nested types?
- How should partial types and partial methods compose?
- Should we differentiate mutable from immutable static state?

### Static state and static initialization

- Which BCL members should initially be marked or certified as `CapabilitySafe`?
- Should ambient time, randomness, and culture have standard explicit capability interfaces?
- Are static constructors allowed if verified?
- Static initialization can mint authority before an apparently safe static member runs. For example:

  ```csharp
  [CapabilitySafe]
  static int Abs(int value) => Helper.Abs(value);

  static class Helper
  {
      static readonly Stream Log = File.OpenWrite("log.txt");
      internal static int Abs(int value) => Math.Abs(value);
  }
  ```

  If referencing `Helper.Abs` triggers `Helper`'s type initializer, the capability-safe method has caused ambient file authority to be acquired even though the called method body is pure. The spec must decide whether to forbid type initializers, verify them as part of certification, or allow only constant/static-readonly data that cannot run authority-bearing code.
- How are beforefieldinit semantics handled? A type initializer may run earlier than the source location that appears to reference the type, which complicates both diagnostics and host reasoning.

### Runtime and library surface

- Are `Thread`, `Task.Run`, timers, and schedulers ambient authority?
- How should `async` methods and `Task` values be modeled?
- How should reflection-adjacent APIs such as `Type`, attributes, serializers, `Expression`, and `ResourceManager` be classified?
- How should `IServiceProvider` be treated when passed explicitly?
- How are dependencies certified transitively?
- Can unverifiable IL ever be certified?
- Should there be a separate contract for not leaking capabilities to callbacks or returned objects?

### IL instructions needing more precise capability rules

- `ldtoken`: this can produce runtime handles for types, methods, or fields. The handle itself is not file or network authority, but it can feed `Type.GetTypeFromHandle`, reflection, serializers, or activation APIs. Should all `ldtoken` be forbidden, or only handles consumed by reflection-like APIs?

  ```csharp
  [CapabilitySafe]
  static Type GetTypeObject() => typeof(File); // likely lowers through ldtoken
  ```

- `isinst` and `castclass`: type tests and casts are ordinary C# operations, but they also reveal runtime type information and can route execution based on concrete implementation type. Are pattern matching and `GetType`-like behavior allowed when reflection is otherwise forbidden?

  ```csharp
  [CapabilitySafe]
  static bool IsFileCapability(object value) => value is IFile;
  ```

- `box`, `unbox`, and `unbox.any`: boxing is usually authority-free, but boxed values can expose object identity, interfaces, and virtual dispatch. The spec should say whether these are always allowed or only allowed for capability-safe value types and interfaces.
- `newarr`, `ldelem.*`, `stelem.*`, `ldelema`, `ldlen`: arrays are explicit objects and therefore seem safe, but array covariance, element type handles, and byref element access interact with type checks and mutation. The verifier needs rules for arrays of capability-bearing references.
- `ldftn` and `ldvirtftn`: method handles can create delegates or feed indirect calls. Instance delegates over explicit receivers are likely safe; static method handles must require certified static targets; open instance delegates and virtual method handles need rules.

  ```csharp
  [CapabilitySafe]
  static Func<string> Capture(IFile file) => file.ReadAllText;
  ```

- `newobj`: constructor calls are authority boundaries. The spec says constructors must be capability-safe, but generic `new()` constraints, synthesized constructors, exception constructors, delegate constructors, and tuple/record construction need explicit treatment.
- `throw` and `rethrow`: exceptions are not authority by themselves, but exception objects can carry stack traces, target-site information, inner exceptions, and arbitrary payload references. The spec should decide whether throwing arbitrary exception instances can leak capabilities or ambient execution information.
- `ldstr`: string literals are probably authority-free, but string interning is global process state. The spec should say whether interning matters for the capability model.
- `sizeof`, `initobj`, `ldobj`, `stobj`, and `cpobj`: these can be harmless for managed value types but become suspicious around unmanaged, byref-like, generic, or custom-modifier-shaped data. The metadata shape rules need to be precise enough to distinguish those cases.
- Prefixes such as `tail.`, `constrained.`, `readonly.`, and `volatile.` are not standalone operations, but they change call, field, and byref semantics. The instruction table should state whether prefixed forms are shorthand for opcode-plus-prefix validation rather than pretending `tail.call` or `volatile.ldfld` are separate opcodes.

## References

- [Robust Composition: Towards a Unified Approach to Access Control and Concurrency Control (Mark Miller's thesis)](https://papers.agoric.com/assets/pdf/papers/robust-composition.pdf)
- [Midori: Objects as Secure Capabilities](https://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/): Midori eliminated ambient authority and access control in favor of capabilities represented by object references.
- [Capability-based security](https://en.wikipedia.org/wiki/Capability-based_security)
- [.NET Framework Code Access Security (CAS)](https://learn.microsoft.com/en-us/dotnet/framework/misc/code-access-security): CAS attempted to limit partially trusted code through permission sets and stack-walk demands.
- [Secure ECMAScript (SES) (2016)](https://github.com/tc39/proposal-ses) and related ecosystem ([Google Caja (2008)](https://en.wikipedia.org/wiki/Caja_project), [Hardened JavaScript (2021)](https://hardenedjs.org/), [Endo (2020)](https://endojs.github.io/endo/))
- [Joe-E (2004)](https://people.eecs.berkeley.edu/~daw/papers/joe-e-ndss10.pdf)
- [E language (1997)](https://en.wikipedia.org/wiki/E_%28programming_language%29)
