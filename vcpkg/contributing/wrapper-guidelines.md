---
title: Wrapping CMake Find Modules
description: Guidelines for wrapping Find Modules using vcpkg-cmake-wrapper.cmake
ms.date: 2/10/2023
---
# Wrapping CMake Find Modules

The vcpkg CMake toolchain modifies CMake's `find_package()` behavior by allowing ports to wrap CMake Find Modules. Wrappers are not intended to replace the original behavior of Find Modules but instead to help initializing variables and targets with vcpkg port configuration details such as installation paths, file names, and transitive usage requirements.

When a CMake project calls `find_package(<Pkg>)`, vcpkg will first look for `share/<pkg>/vcpkg-cmake-wrapper.cmake`. The name of the package will be converted to lowercase in the search path. The original parameters passed to `find_package()` are made available to the wrapper in `ARGS`. The original Find Module can be called via `_find_package(${ARGS})`.

## Example `vcpkg-cmake-wrapper.cmake` structure
```cmake
# Early setup: cache variables for library directories etc.
# (Goal: Make original module succeed with consistent configuration.)
...
# Call the original CMake Find Module.
_find_package(${ARGS})
# Late setup: complementary configuration for transitive usage requirements etc.
# (Goal: Improve usability and maintainability for vcpkg.)
...
```

## CMake version and policies

Wrappers should not rely on having the most recent version of CMake. Unlike portfiles, wrappers are run in the context of user projects where the version of CMake is determined by the user. In order to not limit the availability of vcpkg, wrappers should avoid language features introduced after CMake version 3.7 unless necessary.

Note that some language features depend not only the actual CMake version but also on the activated policies. The default configuration is defined by the user project's `cmake_minimum_required()` statement. To use some features, such as `if ("word" IN_LIST <list>)`, some policies must be activated locally:

```cmake
# vcpkg-cmake-wrapper.cmake
cmake_policy(PUSH)
cmake_policy(SET CMP0057 NEW)
# file contents here, usually including a call to _find_package()
cmake_policy(POP)
```

## Scoping of variables and targets

Find Modules operate in the scope of the subdirectory where they are used (like macros). Therefore, care must be taken to avoid interference with user projects and CMake behavior.

- If a wrapper needs to create variables, these variables must be prefixed with `Z_VCPKG_`. They must not be used without initialization.
- `Z_VCPKG_` variables cannot be assumed to be unmodified through inner calls to `find_package()`.
- If a wrapper creates novel CMake targets, these targets must be namespaced by using prefix `unofficial::`.
- The CMake macro `find_dependency()` from [`CMakeFindDependencyMacro`](https://cmake.org/cmake/help/latest/module/CMakeFindDependencyMacro.html) cannot be used. Additional dependencies must be found using `find_package()`.

## Library locations and selection

CMake Find Modules often use `find_library(<Pkg>_LIBRARY_RELEASE ...)` and
`find_library(<Pkg>_LIBRARY_DEBUG ...)` to locate the link libraries for a
given `<Pkg>`. If the module fails to locate the proper release and debug
variants on its own, the usual wrapper pattern for early setup is:

```cmake
find_library(<Pkg>_LIBRARY_DEBUG
    NAMES name1d name2_d
    NAMES_PER_DIR
    PATHS "${_VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/debug/lib"
    NO_DEFAULT_PATH
)
find_library(<Pkg>_LIBRARY_RELEASE
    NAMES name1 name2
    NAMES_PER_DIR
    PATHS "${_VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/lib"
    NO_DEFAULT_PATH
)
```

If not implemented in the Find Module, the wrapper must take care of selecting
the right configuration for the `<Pkg>_LIBRARIES` variable. Remember that this
depends on the behavior of the Find Module in the lowest supported version of
CMake.

```cmake
include(SelectLibraryConfigurations)
select_library_configurations(<Pkg>)
unset(<Pkg>_FOUND) # https://gitlab.kitware.com/cmake/cmake/-/issues/22509
```

## Setting up targets

Wrappers may add extra targets which are not provided by a particular version of the Find Module. Because Find Modules may be called multiple times from different subdirectories in a user project, there are additional requirements compared to normal CMake Find Module authoring.

- Before adding an imported target, always check if it already exists: `if(NOT TARGET my::target)`
- Calling `target_link_libraries()` on an imported target is only allowed in the subdirectory which created the target, for CMake versions lower than 3.21 (CMP0079/CMake 3.13 lifts the restriction for normal targets only). For simplicity, set/append the properties directly, e.g.
  
  ```cmake
  set_properties(TARGET <Pkg>::Tgt APPEND PROPERTIES INTERFACE_LINK_LIBRARIES some::lib)
  ```

## Handling port features

vcpkg port features often have transitive usage requirements which must be added to imported variables and targets. The transfer of the active features from port build time (portfile variable `FEATURES`) to wrapper usage time can be done by a configuration step in `portfile.cmake`.

```cmake
# portfile.cmake
vcpkg_check_features(OUT_FEATURE_OPTIONS FEATURE_OPTIONS
    PREFIX feature
    FEATURES
        ZLIB WITH_ZLIB
        TIFF WITH_TIFF
)
configure_file(
    "${CMAKE_CURRENT_LIST_DIR}/vcpkg-cmake-wrapper.cmake.in"
    "${CURRENT_PACKAGES_DIR}/share/<pkg>/vcpkg-cmake-wrapper.cmake"
    @ONLY
)
```

Then `vcpkg-cmake-wrapper.cmake.in` can consume the individual feature variables defined by [`vcpkg_check_features()`](../maintainers/functions/vcpkg_check_features.md).
```cmake
# vcpkg-cmake-wrapper.cmake.in
if("@feature_WITH_ZLIB@" STREQUAL "ON") # feature_WITH_ZLIB
    find_package(ZLIB)
    list(APPEND <Pkg>_LIBRARIES ${ZLIB_LIBRARIES})
    if(TARGET <Pkg>::Tgt)
        set_property(TARGET <Pkg>::Tgt APPEND PROPERTY INTERFACE_LINK_LIBRARIES ZLIB::ZLIB)
    endif()
endif()
```

## Enforce loading of exported config files

If a port exports config files which provide accurate information as expected
by users of the Find Module, wrappers may enforce the loading of the config
files by adjusting the argument list.

```cmake
list(REMOVE_ITEM ARGS "NO_MODULE" "CONFIG" "MODULE")
_find_package(${ARGS} CONFIG)
```

This approach is discouraged because it can interact poorly with user-provided Find Modules.

## Handling the `REQUIRED` keyword

When `find_package()` is called with the `REQUIRED` keyword (i.e. `REQUIRED`
occurs in list `ARGS`), any error in setting up the configuration and transitive
usage requirements may immediately raise a fatal error.

However, a normal call to `find_package()` without passing the `REQUIRED` keyword
must not cause fatal CMake errors. It can indicate failure only by setting
`<Pkg>_FOUND` to `FALSE`. This implies that a wrapper must not use the
`REQUIRED` keyword when looking for transitive usage requirements via additional
calls to `find_package()`.

(Normally, failures to find transitive usage requirements indicate serious
issues with the vcpkg setup. However, users may explicitly disable finding and
using some modules by setting `CMAKE_DISABLE_FIND_PACKAGE_<Pkg>` to `ON`.)
