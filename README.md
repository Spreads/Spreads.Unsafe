Spreads.Unsafe
==============

This library adds several methods to `System.Runtime.CompilerServices.Unsafe` [package](https://github.com/dotnet/corefx/tree/master/src/System.Runtime.CompilerServices.Unsafe)
that are used in [Spreads library](https://github.com/Spreads/Spreads) to get the maximum performance.
It could be compiled from a working repo of [corefx](https://github.com/dotnet/corefx) if placed alogside 
with `S.R.CS.U` folder.


The added methods emit a constrained call to instance methods of known interfaces on instances of a generic type `T` 
without a type constraint `where T : IKnownInterface<T>`.

For example, calling the `IComparable<T>.CompareTo` method is implemented like this:


```
  .method public hidebysig static int32 CompareToConstrained<T>(!!T& left, !!T& right) cil managed aggressiveinlining
  {
        .custom instance void System.Runtime.Versioning.NonVersionableAttribute::.ctor() = ( 01 00 00 00 )
        .maxstack 8
        ldarg.0
        ldarg.1
        ldobj !!T
        constrained. !!T
        callvirt instance int32 class [System.Runtime]System.IComparable`1<!!T>::CompareTo(!0)
        ret 
  } // end of method Unsafe::CompareToConstrained
```

In addition to the `IComparable<T>` interface there are `IEquatable<T>` and the following custom ones:


```
public interface IInt64Diffable<T> : IComparable<T>
{
    T Add(long diff);
    long Diff(T other);
}

public interface IDelta<T>
{
    T AddDelta(T delta);
    T GetDelta(T other);
}

```


`KeyComparer<T>`
---------------------

The main use case/sample is `KeyComparer<T>` implemented [here](https://github.com/Spreads/Spreads/blob/master/src/Spreads.Core/KeyComparer.cs). 
A benchmark shows that the unsafe `CompareToConstrained` method and the `KeyComparer<T>` that uses it are c.2x faster than the `Comparer<T>.Default`
when called via the `IComparer<T>` interface and are c.1.6x faster when the default comparer is called directly as a class.


[ComparerInterfaceAndCachedConstrainedComparer](https://github.com/Spreads/Spreads/blob/11625d1632ec5b8ce62c40c4215b1e6e48a6998d/tests/Spreads.Core.Tests/Collections/KeyComparerTests.cs#L15)


 Case                |    MOPS |  Elapsed |   GC0 |   GC1 |   GC2 |  Memory 
------------         |--------:|---------:|------:|------:|------:|--------:
Unsafe               |  403.23 |   248 ms |   0.0 |   0.0 |   0.0 | 0.000 MB
KeyComparer*         |  396.83 |   252 ms |   0.0 |   0.0 |   0.0 | 0.000 MB
Default              |  255.75 |   391 ms |   0.0 |   0.0 |   0.0 | 0.000 MB
Interface            |  211.42 |   473 ms |   0.0 |   0.0 |   0.0 | 0.000 MB


\* `KeyComparer<T>` uses the [JIT compile-time constant optimization](https://github.com/dotnet/corefx/blob/master/src/System.Numerics.Vectors/src/System/Numerics/Vector.cs#L14-L20) 
for known types and falls back to the `Unsafe.CompareToConstrained` method for types that implement `IComparable<T>` interface.
On .NET 4.6.1 there is no visible difference with and without the special cases: `Unsafe.CompareToConstrained` 
performs as fast as the `if (typeof(T) == typeof(Int32)) { ... }` pattern. See the discussion [here](https://github.com/Spreads/Spreads/issues/100#issuecomment-298184971) and
implementation with comments [here](https://github.com/Spreads/Spreads/blob/62639cea51a3df0010501e3dcba8d7a85f2e3022/src/Spreads.Core/KeyComparer.cs#L177-L226) explaining why 
the special cases could be needed on some platforms.


Unsafe methods could only be called on instances of a generic type `T` when the type implements a relevant interface. `KeyComparer<T>`
has a `static readonly` field that (in theory) allows to use the same JIT optimization mentioned above:

```
private static readonly bool IsIComparable = typeof(IComparable<T>).GetTypeInfo().IsAssignableFrom(typeof(T));

public int Compare(T x, T y)
{
    ...
    if (IsIComparable) // JIT compile-time constant 
    {
    return Unsafe.CompareToConstrained(ref x, ref y);
    }
    ...
}

```

But even if such optimizatoin breaks in this particular case (see the linked discussion) then checking a static bool field is 
still much cheaper than a virtual call, especially given that its value is constant for the lifetime of a program and branch 
prediction should be 100% effective.



FastDictionary
---------------

Another use case is [`FastDictionary<TKey,TValue>`](https://github.com/Spreads/Spreads/blob/master/src/Spreads.Core/Collections/Generic/FastDictionary.cs) 
that uses unsafe methods via [`KeyEqualityComparer<T>`](https://github.com/Spreads/Spreads/blob/master/src/Spreads.Core/KeyEqualityComparer.cs), 
which is very similar to `KeyComparer<T>` above. FastDictionay is a rewrite of `S.C.G.Dictionary<TKey,TValue>` that avoids virtual calls
to an equality comparer.

A benchmark for `<int,int>` types shows that `FastDictionary<int,int>` is c.70% faster than `S.C.G.Dictionary<int,int>`:

[CompareSCGAndFastDictionaryWithInts](https://github.com/Spreads/Spreads/blob/8de8e7c5077002fd3d212bb8b2331e3802554e1f/tests/Spreads.Core.Tests/Collections/FastDictionaryTests.cs#L17)


 Case                |    MOPS |  Elapsed |   GC0 |   GC1 |   GC2 |  Memory
---------------      |--------:|---------:|------:|------:|------:|--------:
FastDictionary       |  120.48 |   415 ms |   0.0 |   0.0 |   0.0 | 0.000 MB
Dictionary           |   71.63 |   698 ms |   0.0 |   0.0 |   0.0 | 0.000 MB


Such implementation is much simpler than one with an additoinal generic parameter for a comparer, as recently discussed in this [blog post](https://ayende.com/blog/177377/fast-dictionary-and-struct-generic-arguments).
It is also more flexible than constraining `TKey` to `where TKey : IEquatable<TKey>` and gives the same performance.

Another benchmark with a key as a [custom 16-bytes `Symbol` struct](https://github.com/Spreads/Spreads/blob/master/src/Spreads.Core/DataTypes/Symbol.cs)** shows c.50% performance gain:


[CompareSCGAndFastDictionaryWithSymbol](https://github.com/Spreads/Spreads/blob/8de8e7c5077002fd3d212bb8b2331e3802554e1f/tests/Spreads.Core.Tests/Collections/FastDictionaryTests.cs#L65)

 Case                |    MOPS |  Elapsed |   GC0 |   GC1 |   GC2 |  Memory
---------------      |--------:|---------:|------:|------:|------:|--------:
FastDictionary       |   63.69 |   157 ms |   0.0 |   0.0 |   0.0 | 0.000 MB
Dictionary           |   43.29 |   231 ms |   0.0 |   0.0 |   0.0 | 0.000 MB

\*\* Note that the `Symbol` struct also uses unsafe methods, but the used ones are copied from `S.R.CS.Unsafe` package and are not specific to
this library. However, this library could be a replacement and due to different namespaces they will not cause any conflicts if used together.


Status and version
---------------

Verion 1.0 is available on NuGet as [Spreads.Unsafe](https://www.nuget.org/packages/Spreads.Unsafe).

There is an [interesting discussion](https://github.com/dotnet/coreclr/issues/6520) about intrinsifying `EqualityComparer<T>`,  
aiming to achieve a similar goal of inlining calls when possible. However, the [last comment](https://github.com/dotnet/coreclr/issues/6688#issuecomment-295340599) from a CoreCLR member says that:

> I am not aware of anybody working on this right now, so it is pretty unlikely [that it has a chance to appear in .NET Core 2.0].

Future versions of .NET Core may have much faster comparers, but for existing code and platforms `Spreads.Unsafe` library gives the required performance right here and now.



License
=============

Spreads.Unsafe is licensed under the MIT license.