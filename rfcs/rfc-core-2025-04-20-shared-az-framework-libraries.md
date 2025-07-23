

## Summary:
O3DE manages a set of core framework libraries that are used by all of the executables and components. These libraries are mostly defined as static libraries and thus its contents are built into each of the modules and executables that comprises the O3DE environment. While this gives the flexibility of keeping the libraries simple, it creates a large overhard in terms of sizes of the modules, but also the debug symbols become bloated because of the large amount of unnecessary symbol duplication. The debug symbol size issue is more acute on Linux when loading the symbols through LLDB or GDB will overload the memory. This often triggers out of memory errors, making it very difficult to debug.  At also introduces an unnecessary build dependency such that any non-header changes to these libraries will trigger a re-link of all modules and executables that depend on it, which can be expensive. 

The purpose of this RFC is to propose updating the core framework libraries to be defined as shared libraries when building in non-monolithic build modes. For monolithic game builds, the framework libraries will behave as before.

## Advantages of this feature
There are several improvements that this feature will add to O3DE.

### Reduced Binary Sizes
The biggest advantage of this feature is the reduction of size for all of the compiled binaries, including the debug systems. AzCore and AzFramework are the fundamental build dependencies for almost every module in O3DE. When linking in these libraries as static libraries, both the binary implementation as well as the program/debug database for the libraries are added to each dependent target automatically. Considering there is currently over 500 targets that link in these libraries, the amount of duplication is significant. 

In a side-by-side comparison of [built binaries on windows/msvc](https://gist.github.com/spham-amzn/10a7e4b60cf2dc2e7630bb46c19ba5e8), the following total size reductions on different types of built binaries were observed:

| Type                  | Original Total Size | New Total Size |  Difference | Savings  | 
|-----------------------|--------------------:|---------------:|------------:|---------:|
| Shared Library/Module |           785.51 MB |      519.53 MB |  -265.98 MB |   33.86 %|
| Executable            |           374.96 MB |       40.05 MB |  -334.91 MB |   89.32 %|
| Program Database      |         18046.46 MB |     8180.04 MB | -9866.43 MB |   54.67 %|
| Total                 |            19.21 GB |        8.74 GB |   -10.47 GB |   54.50 %|

Similarly the same comparison on [Linux/Clang](https://gist.github.com/spham-amzn/721fa39a4aa05c808ca58047cdea2e22) yield the following results:

| Type                  | Original Total Size | New Total Size |   Difference | Savings  | 
|-----------------------|--------------------:|---------------:|-------------:|---------:|
| Shared Library/Module |          4382.62 MB |     1430.20 MB |  -2952.43 MB |   67.37 %|
| Executable            |           693.11 MB |      109.19 MB |   -583.92 MB |   84.25 %|
| Debug Symbols         |         35216.92 MB |     9619.26 MB | -25597.66 MB |   54.67 %|
| Total                 |            40.30 GB |        8.74 GB |    -10.47 GB |   72.31 %|

### Reduced unnecessary links
If there is any updates or fixes in these static libraries that do not involve updates to header files (i.e. Interface), all dependent targets are forced to perform at least a relink to the target library. However, if these libraries are shared libraries, and the header/interface has not changed, then a relink is not necessary.

### Reduced Debug Symbol Database
One of the biggest pain points with debugging O3DE on Linux is the size of the debug symbol database. The duplication of all the core Framework libraries causes a large memory footprint for every module that is loaded. The problem when debugging in Linux particularly is that when a module is loaded through LLDB, it will search for the debugging information for every module that is loaded. Considering the total sizes of just the debug symbol files was around 35 GB (see the aforementioned [Linux binary comparison](https://gist.github.com/spham-amzn/721fa39a4aa05c808ca58047cdea2e22)), this forced the debugger process to load multiple gigabytes of data from the program database while loading. On a typical O3DE project that may require many gems, it often triggers out of memory situations. The only work around for this at the moment is to delete the debug symbols for the modules whose symbols are not needed for debugging, which is not an ideal situation. Being able to reduce all the debug symbols for Linux by 72% makes it more likely that all the modules for a project can be loaded without running into memory constraints.

### Consolidate the O3DEKernel and AzCore into one
A previous [RFC](https://github.com/o3de/sig-core/blob/main/rfcs/rfc-core-2022-03-17-allocator-availability.md) that facilitated the introduction of the O3DEKernel module to manage stateful allocators would no longer be needed, as it will be combined into a single AzCore library.

## Disavdantages of this feature

### Additional Macros for RTTI / TypeInfo
The O3DE RTTI / TypeInfo system is managed by AzCore. Having the AzCore library become a shared library will require additional RTTI related macros that need to be use when creating any class in the core framework libraries. This adds more necessary use of macros in these situations. More details are described later in this RFC.

### Slight Increase in Build Times
Although it was originally thought that converting these static libraries into shared libraries would improve total build times, but in actuality it increased them, but not in a significant amount. In incremental build situations where the implementation code of these framework libraries will actually improve build times as it will no longer require a link across all targets (see this described in the advantages section). The following table shows the timings of a full non-unity build of O3DE with the AutomatedTesting project as the active project (On a AMD Ryzen Thread-ripper with 32 cores and 32 GB of RAM)


| Platform      | Original Time  | New Time  | Difference |
|---------------|----------------|-----------|------------|
| Windows       |        1h  4m  |    1h 11m |     +7 min |
| Linux         |           39m  |       39m |      0 min |


## Updates to support feature


### O3DEKernel 
The `O3DEKernel` library was introduced into O3DE (refer to this [RFC](https://github.com/o3de/sig-core/blob/main/rfcs/rfc-core-2022-03-17-allocator-availability.md) to address the stateful allocators that are defined in the AzCore static library by using a shared library to manage environment variables. Since this proposal is to make AzCore into a shared library, having the separate `O3DEKernel` library is no longer necessary and will be merged back into the AzCore shared library.

### Conditional API Export Define
The following API macros and their private control defines are defined as follows:

| Library          | Export define  | Module Control Define |
|------------------|----------------|-----------------------|
| AzCore           | AZCORE_API     | AZCORE_EXPORTS        |
| AzFramework      | AZF_API        | AZF_EXPORTS           | 
| AzToolsFramework | AZTF_API       | AZTF_EXPORTS          |

*NOTE* All of the export defines are pre-empted by the global `AZ_MONOLITHIC_BUILD` define which is used when bulding monolithic game launchers.


###  RTTI / TypeInfo support for exported O3DE Types

The `AZ_RTTI` macros are used to register types with O3DE's runtime type information. These helper macros simplifies registering type information for classes by providing a simplified and consistent way to declare and implement the required AZ_RTTI functions for any type. Since these functions were originally defined in the static AzCore library, there was no need to forward declare them to be visible to the linker. Ever since build performance improvements were introduced to reduce compile time (see [Typeinfo simplification](https://github.com/o3de/o3de/pull/14824) ), the implementation was moved from being inline to being compiled in respective `cpp` compilation units. There were only referred to as `friend` references to the original class. To the compiler, this not a full declaration, and in order for a forward declaration of the RTTI functions including its visibility attribute (i.e. `dll_export`), this needs to be done outside of the class declaration. 

The following API version of the original macros were added to support this:

| Original                              | API version                               |
|---------------------------------------|-------------------------------------------|
|AZ_TYPE_INFO_SPECIALIZE_WITH_NAME_DECL | AZ_TYPE_INFO_SPECIALIZE_WITH_NAME_DECL_API|
|AZ_TYPE_INFO_WITH_NAME_DECL            | AZ_TYPE_INFO_WITH_NAME_DECL_API           |

In addition to updating `AZ_TYPE_INFO_WITH_NAME_DECL` -> `AZ_TYPE_INFO_WITH_NAME_DECL_API`, an extra declaration is needed which is supported by the new `AZ_TYPE_INFO_WITH_NAME_DECL_EXT_API` macro. 

For example:
```
    class MY_API MyClass
    {
    public:
        AZ_CLASS_ALLOCATOR(MyClass, SystemAllocator);
        AZ_TYPE_INFO_WITH_NAME_DECL_API(MY_API, MyClass);
        AZ_RTTI_NO_TYPE_INFO_DECL();
    };
    AZ_TYPE_INFO_WITH_NAME_DECL_EXT_API(MY_API, MyClass)
```

### Exporting specialized templates
Template types and functions cannot be exported by themselves. Templates by themselves can be all inline and included in the header of a shared library. Specialized templates can be exported from shared libraries, and the following are some examples of the syntax to do so.

#### Templated Types
The declaration for exported templated types follows the pattern below:

```
AZCORE_API_EXTERN template struct AZCORE_API ClassNames<wchar_t>;
```

The private implementation file (cpp) declares the exported instantiation of the specialized type.
```
template struct AZCORE_API_EXPORT ClassNames<wchar_t>;
```

#### Templated Functions
The declaration for templated functions follows in a similar pattern.

Declaration
```
AZCORE_API_EXTERN template AZCORE_API AZ::Outcome<AZStd::string, AZStd::string> ReadFile(AZStd::string_view filePath, size_t maxFileSize);
```

Implementation
```
template AZCORE_API_EXPORT AZ::Outcome<AZStd::string, AZStd::string> ReadFile(AZStd::string_view filePath, size_t maxFileSize);
```


### Exporting EBuses
The [EBus](https://www.docs.o3de.org/docs/user-guide/programming/messaging/ebus/) system is the main publisher/subscriber system used within O3DE. Any EBus that is declared must be specialized if they are to be exported from a shared library. In addition, the way that these EBuses are declared require that some of the internal handler sub-classes, which are also templates, must be specialized and exported as well. New macros will be defined to support the declaration and instantiation of two types of EBus address policies: `Single` and `ById`. Also, additional macros are provided for the situation where the EBus trait is separate from the EBus interface class.

| Address Policy | Declaration Macro                          | Instantiation Macro                            | Arguments              |
|----------------|--------------------------------------------|------------------------------------------------|------------------------|
| Single         | AZ_DECLARE_EBUS_SINGLE_ADDRESS             | AZ_INSTANTIATE_EBUS_SINGLE_ADDRESS             | API, Interface         |
| Single         | AZ_DECLARE_EBUS_SINGLE_ADDRESS_WITH_TRAITS | AZ_INSTANTIATE_EBUS_SINGLE_ADDRESS_WITH_TRAITS | API, Interface         |
| By ID          | AZ_DECLARE_EBUS_MULTI_ADDRESS              | AZ_INSTANTIATE_EBUS_MULTI_ADDRESS              | API, Interface, Traits |
| By ID          | AZ_DECLARE_EBUS_MULTI_ADDRESS_WITH_TRAITS  | AZ_INSTANTIATE_EBUS_MULTI_ADDRESS_WITH_TRAITS  | API, Interface, Traits |

#### Examples

The following examples illustrates the different EBus macros and their scenarios. It assumes the ebus is located in a shared library call `FooLib` and that there is an API macro called `FOO_API` that controls the type of `declspec/__visibility` used for the export declarations.

##### Foo.h
```C++
#include <AzCore/EBus/EBus.h>
namespace Foo
{
    class FooByIDBusTraitOnly
        : public AZ::EBusTraits
    {
    public:
        static const EBusAddressPolicy AddressPolicy = EBusAddressPolicy::ById;
        typedef int BusIdType;
    };

    class FooSingleBusTraitOnly
        : public AZ::EBusTraits
    {
    public:
        static const EBusAddressPolicy AddressPolicy = EBusAddressPolicy::Single;
    };

    class FooByIDBus
        : public FooByIDBusTraitOnly
    {
    public:
        virtual void Bar() = 0;
    };

    class FooSingleBus
        : public FooByIDBusTraitOnly
    {
    public:
        virtual void Bar() = 0;
    };

    class FooInterface
    {
    public:
        virtual void Bar() = 0;
    };
}

AZ_DECLARE_EBUS_SINGLE_ADDRESS(FOO_API, Foo::FooSingleBus);
AZ_DECLARE_EBUS_SINGLE_ADDRESS_WITH_TRAITS(FOO_API, Foo::FooInterface, Foo::FooSingleBusTraitOnly);
AZ_DECLARE_EBUS_MULTI_ADDRESS(FOO_API, Foo::FooByIDBus);
AZ_DECLARE_EBUS_MULTI_ADDRESS_WITH_TRAITS(FOO_API, Foo::FooInterface, Foo::FooByIDBusTraitOnly);
```

##### Foo.cpp
```C++

#include "Foo.h"

AZ_INSTANTIATE_EBUS_SINGLE_ADDRESS(FOO_API, Foo::FooSingleBus);
AZ_INSTANTIATE_EBUS_SINGLE_ADDRESS_WITH_TRAITS(FOO_API, Foo::FooInterface, Foo::FooSingleBusTraitOnly);
AZ_INSTANTIATE_EBUS_MULTI_ADDRESS(FOO_API, Foo::FooByIDBus);
AZ_INSTANTIATE_EBUS_MULTI_ADDRESS_WITH_TRAITS(FOO_API, Foo::FooInterface, Foo::FooByIDBusTraitOnly);
```

### Additional general project changes

#### Force Inlining
For a number of EBuses, the templated handler subclasses gave problems with the MSVC compiler when attempting to use them across multiple modules for a single EBus. For example, if an export EBus is used by multiple modules, and if those same modules have any dependency on each other, the MSVC compiler thinks that the handlers violates the One-Definition Rule. For example, the `EntitySelectionEvents` EBus generates the following linker error:

```
73>AzToolsFramework.lib(AzToolsFramework.dll) : error LNK2005: "public: virtual __cdecl AZ::Internal::IdHandler<class AzToolsFramework::EntitySelectionEvents,class AzToolsFramework::EntitySelectionEvents,struct AZ::Internal::EBusContainer<class AzToolsFramework::EntitySelectionEvents,class AzToolsFramework::EntitySelectionEvents,1,1> >::~IdHandler<class AzToolsFramework::EntitySelectionEvents,class AzToolsFramework::EntitySelectionEvents,struct AZ::Internal::EBusContainer<class AzToolsFramework::EntitySelectionEvents,class AzToolsFramework::EntitySelectionEvents,1,1> >(void)" (??1?$IdHandler@VEntitySelectionEvents@AzToolsFramework@@V12@U?$EBusContainer@VEntitySelectionEvents@AzToolsFramework@@V12@$00$00@Internal@AZ@@@Internal@AZ@@UEAA@XZ) already defined in GradientSignal.Editor.Static.lib(GradientPreviewer.obj)
73>AzToolsFramework.lib(AzToolsFramework.dll) : error LNK2005: "public: virtual __cdecl AZ::Internal::IdHandler<class AzToolsFramework::EditorEntityVisibilityNotifications,class AzToolsFramework::EditorEntityVisibilityNotifications,struct AZ::Internal::EBusContainer<class AzToolsFramework::EditorEntityVisibilityNotifications,class AzToolsFramework::EditorEntityVisibilityNotifications,1,1> >::~IdHandler<class AzToolsFramework::EditorEntityVisibilityNotifications,class AzToolsFramework::EditorEntityVisibilityNotifications,struct AZ::Internal::EBusContainer<class AzToolsFramework::EditorEntityVisibilityNotifications,class AzToolsFramework::EditorEntityVisibilityNotifications,1,1> >(void)" (??1?$IdHandler@VEditorEntityVisibilityNotifications@AzToolsFramework@@V12@U?$EBusContainer@VEditorEntityVisibilityNotifications@AzToolsFramework@@V12@$00$00@Internal@AZ@@@Internal@AZ@@UEAA@XZ) already defined in GradientSignal.Editor.Static.lib(EditorSurfaceAltitudeGradientComponent.obj)
```
The only way to solve this linker error outside of resorting to the [FORCE:Multiple](https://learn.microsoft.com/en-us/cpp/build/reference/force-force-file-output?view=msvc-170) linker flag is use the `AZ_FORCE_INLINE` compiler-independent define for the EBus templated subclasses that handles the address policy.

#### AZ_DISABLE_COPY
When some of the classes where switched to a module-exported class, the compiler error `attempting to reference a deleted function" for a unique_ptr` would appear. This is generally caused by a class that uses a `unique_ptr` to another class that defines a copy & move constructor.  These errors do not show up when compiling a static library because they are not exposed unless the module that uses them actually attempts to a `std::move` on that class. They appear in shared libraries because nothing is dead-stripped (see https://devblogs.microsoft.com/oldnewthing/20190927-00/?p=102932 for an explaination).  This error is solved by prevent unique pointers from trying to 'copy' each other by deleting the default copy and assignment operators. On some compilers, this rule extends to any subclasses that have their copy constructors/operators deleted.  This is done by using the `AZ_DISABLE_COPY` and `AZ_DISABLE_COPY_MOVE` macros that explicitly deletes them.



#### API Externalizing CVARS

If a CVAR is declared in a shared library, and wants to be exported for use outside of the library, the `AZ_CVAR_API_EXTERNED` replaces `AZ_CVAR_EXTERNED`. The macro takes in an additional `API` macro based on the shared framework library in which it is declared. 

For example, `ed_useNewAssetBrowserListView` is declared in AzToolsFramework

```
AZ_CVAR_API(AZTF_API, bool, ed_useNewAssetBrowserListView, true, nullptr, AZ::ConsoleFunctorFlags::Null,
        "Use the new AssetBrowser ListView for searching assets.");
```

To access it outside of the module, it would be declared using the `AZTF_API` macro:

```
AZ_CVAR_API_EXTERNED_API(AZTF_API, bool, ed_useNewAssetBrowserListView);
```


#### Constants moved from AZ::SettingsRegistryInterface to AZ::SettingsRegistryConstants
Constants that were in `AZ::SettingsRegistryInterface` were moved to `AZ::SettingsRegistryConstants`, e.g. 

```
AZ::SettingsRegistryInterface::RegistryFolder -> AZ::SettingsRegistryConstants::RegistryFolder
AZ::SettingsRegistryInterface::DevUserRegistryFolder -> AZ::SettingsRegistryConstants::DevUserRegistryFolder
```

#### Move global variables to global accessors
The `AzToolsFramework::g_mainManipulatorManagerId` was moved to a global accessor method `AzToolsFramework::GetMainManipulatorManagerId()`

#### GetCurrentSerializeContextModule renamed to GetGlobalSerializeContextModule

`AZ::GetCurrentSerializeContextModule()` was renamed to `AZ::GetGlobalSerializeContextModule()` to provide an updated description of its new role.


## How will be change be integrated
The scope of this change is large and cannot be done incrementally. This change will first reside in a long lived feature branch called `shared_framework_libraries` that will be parallel to the development branch of O3DE. There will be incremental pull requests into this feature branch for the purpose of creating pull requests that will be based on the different category of changes needed. The branch itself will not be compilable until all the initial pull requests are approved and merged in. At that point, it will stay on this branch until a later TBD date to allow for testing. The branch will be periodically kept up to date with the main development branch.



## FAQ

### How large is this change?
The proof of concept branch that was used to prototype and test the updates necessary contains about 1450 updated files.

### What are the risks of having so many files updated?
The general guideline of the changes needed is to not change any logic. All the code changes are primarily focused on how the classes, types, and variables are declared and scope. There may be minor refactors necessary to solve items like type visibility.


> For any additional questions, please reach out to the #sig-core on [Discord](https://discord.com/invite/o3de).
