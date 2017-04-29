Spreads.Unsafe
==============

This repo adds several methods to `System.Runtime.CompilerServices.Unsafe` [package](https://github.com/dotnet/corefx/tree/master/src/System.Runtime.CompilerServices.Unsafe) and repackages it. 
It could be compiled from a working repo of [corefx](https://github.com/dotnet/corefx) if placed alogside 
with `S.R.CS.U` folder.


The added methods emit a constrained call to methods of known interfaces on instances of a generic type `T` 
without a type constraint `where T : IKnownInterface<T>`.

For example, calling the `IComparable<T>.CompareTo` method:


```
  .method public hidebysig static int32 CompareToConstrained<T>(!!T left, !!T right) cil managed aggressiveinlining
  {
        .custom instance void System.Runtime.Versioning.NonVersionableAttribute::.ctor() = ( 01 00 00 00 )
        .maxstack 8
        ldarga.s left
        ldarg.1
        constrained. !!T
        callvirt instance int32 class [System.Runtime]System.IComparable`1<!!T>::CompareTo(!0)
        ret 
  } // end of method Unsafe::CompareConstrained
```

In addition to `IComparable<T>` there are `IEquatable<T>` interface and the following custom ones:


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

Available on NuGet as `Spreads.Unsafe`. Tests and a compiled dll are in the main Spreads project.
A use case/sample is [here](https://github.com/Spreads/Spreads/blob/master/src/Spreads.Core/KeyComparer.cs).
