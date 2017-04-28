Spreads.Unsafe
==============

This repo adds a single method to `System.Runtime.CompilerServices.Unsafe` [package](https://github.com/dotnet/corefx/tree/master/src/System.Runtime.CompilerServices.Unsafe) and repackages it. 
It could be compiled from a working repo of [corefx](https://github.com/dotnet/corefx) if placed alogside 
with `S.R.CS.U` folder.

The added method is:


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

It emits a constrained call to `IComparable<T>.CompareTo()` method on instances of a generic type `T` 
without a type constraint `where T : IComparable<T>`.

Available on NuGet as `Spreads.Unsafe`.
