---
ms.topic: include
ms.date: 05/23/2023
---

## Preserve code

When you use the linker, it sometimes remove code that you might have called dynamically, even indirectly. You can instruct the linker to preserve members by annotating them with the [`DynamicDependency`](xref:System.Diagnostics.CodeAnalysis.DynamicDependencyAttribute) attribute. This attribute can be used to express a dependency on either a type and subset of members, or at specific members.

> [!IMPORTANT]
> Every member in the BCL that can't be statically determined to be used by the app is subject to be removed.

The [`DynamicDependency`](xref:System.Diagnostics.CodeAnalysis.DynamicDependencyAttribute) attribute can be applied to constructors, fields, and methods:

```csharp
[DynamicDependency("Helper", "MyType", "MyAssembly")]
static void RunHelper()
{
    var helper = Assembly.Load("MyAssembly").GetType("MyType").GetMethod("Helper");
    helper.Invoke(null, null);
}
```

In this example, the [`DynamicDependency`](xref:System.Diagnostics.CodeAnalysis) ensures that the `Helper` method is kept. Without the attribute, linking would remove `Helper` from `MyAssembly` or remove `MyAssembly` completely if it's not referenced elsewhere.

The attribute specifies the member to keep via a `string` or via the [`DynamicallyAccessedMembers`](xref:System.Diagnostics.CodeAnalysis.DynamicallyAccessedMembersAttribute) attribute. The type and assembly are either implicit in the attribute context, or explicitly specified in the attribute (by `Type`, or by `string`s for the type and assembly name).

The type and member strings use a variation of the C# documentation comment ID string [format](/dotnet/csharp/language-reference/language-specification/documentation-comments#d42-id-string-format), without the member prefix. The member string shouldn't include the name of the declaring type, and may omit parameters to keep all members of the specified name. The following examples show valid uses:

```csharp
[DynamicDependency("Method()")]
[DynamicDependency("Method(System,Boolean,System.String)")]
[DynamicDependency("MethodOnDifferentType()", typeof(ContainingType))]
[DynamicDependency("MemberName")]
[DynamicDependency("MemberOnUnreferencedAssembly", "ContainingType", "UnreferencedAssembly")]
[DynamicDependency("MemberName", "Namespace.ContainingType.NestedType", "Assembly")]
// generics
[DynamicDependency("GenericMethodName``1")]
[DynamicDependency("GenericMethod``2(``0,``1)")]
[DynamicDependency("MethodWithGenericParameterTypes(System.Collections.Generic.List{System.String})")]
[DynamicDependency("MethodOnGenericType(`0)", "GenericType`1", "UnreferencedAssembly")]
[DynamicDependency("MethodOnGenericType(`0)", typeof(GenericType<>))]
```

## Preserve assemblies

It's possible to specify assemblies that should be excluded from the linking process, while allowing other assemblies to be linked. This approach can be useful when you can't easily use the [`DynamicDependency`](xref:System.Diagnostics.CodeAnalysis.DynamicDependencyAttribute) attribute, or don't control the code that's being linked away.

When it links all assemblies, you can tell the linker to skip an assembly by setting the `TrimmerRootAssembly` MSBuild property in an `<ItemGroup>` tag in the project file:

```xml
<ItemGroup>
  <TrimmerRootAssembly Include="MyAssembly" />
</ItemGroup>
```

> [!NOTE]
> The `.dll` extension isn't required when setting the `TrimmerRootAssembly` MSBuild property.

If the linker skips an assembly, it's considered *rooted*, which means that it and all of its statically understood dependencies will be kept. You can skip additional assemblies by adding more `TrimmerRootAssembly` MSBuild properties to the `<ItemGroup>`.

## Preserve assemblies, types, and members

You can pass the linker an XML description file that specifies which assemblies, types, and members need to be retained.

To exclude a member from the linking process when linking all assemblies, set the `TrimmerRootDescriptor` MSBuild property in an `<ItemGroup>` tag in the project file to the XML file that defines the members to exclude:

```xml
<ItemGroup>
  <TrimmerRootDescriptor Include="MyRoots.xml" />
</ItemGroup>
```

The XML file then uses the trimmer [descriptor format](https://github.com/dotnet/linker/blob/main/docs/data-formats.md#descriptor-format) to define which members to exclude from linking:

```xml
<linker>
  <assembly fullname="MyAssembly">
    <type fullname="MyAssembly.MyClass">
      <method name="DynamicallyAccessedMethod" />
    </type>
  </assembly>
</linker>
```

In this example, the XML file specifies a method that's dynamically accessed by the app, which is excluded from linking.

When an assembly, type, or member is listed in the XML, the default action is preservation, which means that regardless of whether the linker thinks it's used or not, it's preserved in the output.

> [!NOTE]
> The preservation tags are ambiguously inclusive. If you don’t provide the next level of detail, it will include all the children. If an assembly is listed without any types, then all the assembly’s types and members will be preserved.

## Mark an assembly as linker safe

If you have a library in your project, or you're a developer of a reusable library and you want the linker to treat your assembly as linkable, you can mark the assembly as linker safe by adding the `IsTrimmable` MSBuild property to the project file for the assembly:

```xml
<PropertyGroup>
    <IsTrimmable>true</IsTrimmable>
</PropertyGroup>
```

This marks your assembly as "trimmable" and enables trim warnings for that project. Being "trimmable" means your assembly is considered compatible with trimming and should have no trim warnings when the assembly is built. When used in a trimmed app, the assembly's unused members will be removed in the final output.

Setting the `IsTrimmable` MSBuild property to `true` in your project file inserts the [`AssemblyMetadata`](xref:System.Reflection.AssemblyMetadataAttribute) attribute into your assembly:

```csharp
[assembly: AssemblyMetadata("IsTrimmable", "True")]
```

Alternatively, you can add the [`AssemblyMetadata`](xref:System.Reflection.AssemblyMetadataAttribute) attribute into your assembly without having added the `IsTrimmable` MSBuild property to the project file for your assembly.

> [!NOTE]
> If the `IsTrimmable` MSBuild property is set for an assembly, this overrides the `AssemblyMetadata("IsTrimmable", "True")` attribute. This enables you to opt an assembly into trimming even if it doesn't have the attribute, or to disable trimming of an assembly that has the attribute.

### Suppress analysis warnings

When the linker is enabled, it will remove IL that's not statically reachable. Apps that uses reflection or other patterns that create dynamic dependencies may be broken as a result. To warn about such patterns, when marking an assembly as linker safe library authors should set the `SuppressTrimAnalysisWarnings` MSBuild property to `false`:

```xml
<PropertyGroup>
  <SuppressTrimAnalysisWarnings>false</SuppressTrimAnalysisWarnings>
</PropertyGroup>
```

This will include warnings about the entire app, including your own code, library code, and SDK code.

### Show detailed warnings

Trim analysis produces at most one warning for each assembly that comes from a `PackageReference`, indicating that the assembly's internals aren't compatible with trimming. When marking an assembly as linker safe, library authors should enable individual warnings for all assemblies by setting the `TrimmerSingleWarn` MSBuild property to `false`:

```xml
<PropertyGroup>
  <TrimmerSingleWarn>false</TrimmerSingleWarn>
</PropertyGroup>
```

This will show all detailed warnings, instead of collapsing them to a single warning per assembly.

## See also

- [ILLink](https://github.com/dotnet/linker/tree/main/docs)
