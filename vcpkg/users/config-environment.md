---
title: Environment variables
description: Use environment variables to control how vcpkg works and where it looks for files.
ms.date: 11/30/2022
---
# Environment variables

## VCPKG_DOWNLOADS

This environment variable can be set to an existing directory to use for storing downloads instead of the internal
`downloads/` directory. It should always be set to an absolute path.

## VCPKG_FEATURE_FLAGS

This environment variable can be set to a comma-separated list of off-by-default features in vcpkg. These features are
subject to change without notice and should be considered highly unstable.

There are no off-by-default feature flags at this time.

## EDITOR

This environment variable can be set to the full path of an executable to be used for `vcpkg edit`. Please see
`vcpkg help edit` for command-specific help.

## VCPKG_ROOT

This environment variable can be set to a directory to use as the root of the vcpkg instance. This will only be used if
the vcpkg executable is not located within a valid root and the command line switch `--vcpkg-root` is unused.

## VCPKG_VISUAL_STUDIO_PATH

This environment variable can be set to the full path to a Visual Studio instance on the machine. This Visual Studio instance
will be used if the triplet does not override it via the [`VCPKG_VISUAL_STUDIO_PATH`](triplets.md#VCPKG_VISUAL_STUDIO_PATH) triplet setting.

Example: `D:\2017`

## VCPKG_DEFAULT_TRIPLET

This environment variable can be set to a triplet name which will be used for unqualified triplet references in command lines.

## VCPKG_DEFAULT_HOST_TRIPLET

This environment variable can be set to a triplet name which will be used for unqualified host port references in command lines and all host port references in dependency lists. See [the host-dependencies documentation](host-dependencies.md) for more information.

## VCPKG_OVERLAY_PORTS

This environment variable adds additional [overlay ports](registries.md#configuration-overlay-ports) paths considered after those listed on the command line. List paths to overlays using
the platform dependent PATH separator (Windows `;` | others `:`)

Example (Windows): `C:\custom-ports\boost;C:\custom-ports\sqlite3`

## VCPKG_OVERLAY_TRIPLETS

This environment variable allows users to add directories to search for triplets.

List paths to overlays using the platform dependent PATH separator (Windows `;`, others `:`)

See [Example: overlay triplets](examples/overlay-triplets-linux-dynamic.md) for an example.

## VCPKG_FORCE_SYSTEM_BINARIES

This environment variable, if set, suppresses the downloading of CMake and Ninja and forces the use of the system binaries.

## VCPKG_FORCE_DOWNLOADED_BINARIES

This environment variable, if set, ignores the use of the system binaries and will always download and use the version defined by vcpkg.

## VCPKG_KEEP_ENV_VARS

This environment variable can be set to a list of environment variables, separated by `;`, which will be propagated to
the build environment.

The values of the kept variables will not be tracked in package ABIs and will not cause rebuilds when they change. To
pass in environment variables that should cause rebuilds on change, see [`VCPKG_ENV_PASSTHROUGH`](triplets.md#VCPKG_ENV_PASSTHROUGH).

Example: `FOO_SDK_DIR;BAR_SDK_DIR`

## <a name="VCPKG_MAX_CONCURRENCY"></a> VCPKG_MAX_CONCURRENCY

This environment variables limits the amount of concurrency used by underlying buildsystems. If unspecified, this defaults to logical cores + 1.

## VCPKG_DEFAULT_BINARY_CACHE

This environment variable redirects the default location to store binary packages. See [Binary Caching](binarycaching.md#configuration) for more details.

## VCPKG_BINARY_SOURCES

This environment variable adds or removes binary sources. See [Binary Caching](binarycaching.md#configuration) for more details.

## VCPKG_NUGET_REPOSITORY

This environment variable changes the metadata of produced NuGet packages. See [Binary Caching](binarycaching.md#configuration) for more details.

## VCPKG_USE_NUGET_CACHE

This environment variable allows using NuGet's cache for every nuget-based binary source. See [Binary Caching](binarycaching.md#nuget-provider-configuration) for more details.

## X_VCPKG_ASSET_SOURCES

> [!NOTE]
> This feature is experimental and may change or be removed at any time.

This environment variable allows using a private mirror for all SHA512-tagged assets. See [Asset Caching](assetcaching.md) for more details.

## VCPKG_NO_CI

Setting `VCPKG_NO_CI` disables vcpkg's CI environment detection heuristics.
