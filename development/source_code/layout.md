# Source code layout

## Status

We are currently migrating to this source code layout. Some aspects of this
document refect the current reality, but some aspects are still aspirational.

## Overview

Most first-party, open-source code is in the
["fuchsia.git" repository](https://fuchsia.googlesource.com/fuchsia).  Most code in
this repository is organized into a recursive tree of areas, which have a
regular directory and dependency structure.

Source repositories (whether open- or closed-source) also follow the
conventions for areas and are mapped into subdirectories of `src` in
fuchsia.git. Current, we have small number of petal repositories but will
"promote" areas into separate petal repositories as the system stabilizes.

The products directory contains a list of products that you can build. Some
products are quite small and build quickly (e.g., the [base](/products/base.gni)
product), whereas others are more elaborate (e.g., the
[workstation](/products/workstation.gni) product).

Most third-party dependencies are stored in separate repositories. These
repositories are included in a local checkout only when needed to support one of
the following source tree configurations:

 * Bringup. This source tree configuration contains enough code to build the
   [bringup](/products/bringup.gni) product.
 * Open Source. This source tree configuration contains all the open source code
   in the Fuchsia Source Tree.
 * All Source.  This source tree configuration contains all the open and closed
   source code in the Fuchsia Source Tree.

## Areas

Most code is organized into a recursive tree of areas. Each area has a regular
directory and dependency structure, which helps people understand code structure
across the whole project.

### Directory Structure

Each area is required to have a [OWNERS](owners.md) file as well as
documentation and tests. Areas can also include binaries, libraries, drivers,
and other source code. In addition, areas can have subareas, which repeat the
pattern:

 * `OWNERS`
    * Each area or subarea must have a [list of owners](owners.md)
 * `docs/`
    * This directory should contain docs for people working in this area
    * Docs for end-developers (or people working in other areas of Fuchsia)
      should be in the top-level docs or sdk repository
* `bundles/`
    * This directory contains bundles of package targets in this area. Each area
      should contain at least a `tests` bundle with unit tests for the area, but
      may include other bundles.
 * `bin/` (optional)
 * `lib/` (optional)
 * `drivers/` (optional)
 * `examples/` (optional)
 * `fidl/` (optional)
    * In some cases, an area might have internal FIDL interfaces that are not
      exposed to other areas or to end-developers. Rather than put those
      interfaces in the SDK, an area can put those interfaces in this directory.
 * `third_party/` (optional)
    * Most third_party dependencies should be in separate repositories
    * Include third_party dependencies in an area only if all of the following:
        * The code is required to be in a third_party directory by policy
        * You intend to fork upstream (i.e., make major changes and not plan to
          integrate future changes from upstream)
        * You make a new name for the code that (a) does not match upstream and
          (b) does not appear in any other third_party directory anywhere in the
          Fuchsia Source Tree
 * `[subareas]` (optional)
    * Subareas should follow the generic area template
    * Do not create deeply nested area structures (e.g., three should be enough)

### Dependency Structure

In addition to depending on itself, an area can depend only on the top-level
`build`, `buildtools`, `sdk`, and `third_party` directories, as well as the
`lib` directories of its ancestors:

 * `//build`
 * `//buildtools`
 * `//sdk`
 * `//third_party`
 * `(../)+lib`

## Repository layout

This section depicts the directory layout for the Fuchsia Source Tree. Non-bold
entries are directories or files in the fuchsia.git repository. Bold entries are
separate repositories that are mapped into the directory structure using `jiri`
(except for the prebuilt directory, which is populated from CIPD).

 * `.clang-format`
 * `.dir-locals.el`
 * `.gitattributes`
 * `.gitignore`
 * `AUTHORS`
 * `CODE_OF_CONDUCT.md`
 * `CONTRIBUTING.md`
 * `LICENSE`
 * `OWNERS`
 * `PATENTS`
 * `README.md`
 * `rustfmt.toml`
 * `sdk/banjo/ddk.protocol.gpio/`
 * `sdk/banjo/...`
 * `sdk/fidl/fuchsia.media/`
 * `sdk/fidl/fuchsia.mediacodec/`
 * `sdk/fidl/...`
 * `sdk/lib/ddk/`
 * `sdk/lib/fit/`
 * `sdk/lib/fidl/`
 * `sdk/lib/zircon/`
 * `sdk/lib/...`
 * `.gn`
 * `BUILD.gn`
 * `build/`
 * `bundles/`
 * `configs/`
 * `integration/`
 * `products/`
 * `scripts/`
 * `docs/`
 * `examples/`
 * `third_party/`
    * **`boringssl/`**
    * **`icu/`**
    * **`rust_crates/`**
    * **`...`**
 * `prebuilt/`
    * **`chromium/`**
    * **`dart/`**
    * **`flutter/`**
    * **`llvm/`**
 * `tools/`
    * `banjo/`
    * `fidl/bin/backend/{c,cpp,dart,go,llcpp,rust}`
    * `fidl/bin/frontend/`
    * `fidl/docs/`
    * `fidl/examples/`
    * `fidl/tests/`
 * `src/`
    * `lib/`
    * `cobalt/`
    * `component/`
    * `connectivity/`
    * `developer/`
    * **`experiences/`**
    * `graphics/`
    * `identity/`
    * `ledger/`
    * `media/`
    * `modular/`
    * `storage/`
    * `updater/`
    * `virtualization/`
    * `zircon/docs/`
    * `zircon/kernel/`
    * `zircon/drivers/`
    * `zircon/userspace/`
    * `vendor/`
        * **`[closed-source code from various vendors]`**

## Evolution

As system stabilizes, we can promote areas out of fuchsia.git into separate
repositories. Generally, we should promote an area to a separate repository when
the interface between the area and the rest of the system is sufficiently stable
(requires approval by top-level OWNERS).

New code can be:

 * Added to an existing directory in fuchsia.git
 * Added to a new top-level area or subarea of an existing area
 * Added to an existing repository
 * Added to a new repository (requires approval by top-level OWNERS)
