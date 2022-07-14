# nocargo: Cargo in Nix

🚧 *This project is under development and is not ready for production yet. APIs are subjects to change.*

Build Rust crates with *Nix Build System*.
- No `cargo` dependency during building. Only `rustc`.
- No need for hash prefetching or code generation[^no-code-gen].
- Crate level caching, globally shared.
- [nixpkgs] integration for non-Rust dependencies.

[^no-code-gen]: Initial template generation and `Cargo.lock` updatin don't count for "code generation". The former is optional, and the latter is indeed not "code".

## Feature checklist

- Nix library
  - [ ] Non-flake support.
  - [x] `[workspace]`
    - [x] `members`
    - [ ] Auto-`members`
    - [x] `excludes`
      FIXME: Buggy.
  - [ ] `resolver`
        Currently has custom resolution algorithm, more like v2.
  - [x] `links`
  - [x] `[profile]`
  - [x] `[{,dev-,build-}dependencies]`
  - [x] `[features]`
    - [ ] Overriding API
  - [x] `[target.<cfg>.dependencies]`
  - [ ] `[patch]`
  - [ ] Cross-compilation.
        FIXME: Buggy with proc-macros.
- `noc` helper
  - [x] `noc init`: Initial template `flake.nix` generation
    - Dependency kinds
      - [ ] `registry`
      - [x] `registry-index`
      - [x] `git`
      - [x] `path` inside workspace
      - [ ] `path` outside workspace
    - Target detection
      - [ ] Library
            FIXME: Assume to always exist.
      - [x] Binary
      - [ ] Test
      - [ ] Bench
      - [ ] Example
  - [ ] `Cargo.lock` generation and updating

## Start with Nix flake

1. Enter the root directory of your rust workspace or package. Currently, you should have `Cargo.lock` already created by `cargo`.
2. Run `nix run github:oxalica/nocargo init` to generate `flake.nix`. Or write it by hand by following [the next section](#example-flake.nix-structure).
3. Check flake outputs with `nix flake show`. Typically, the layout would be like,
   ```
   └───packages
       └───x86_64-linux
           ├───default: package 'rust_mypkg1-0.1.0'           # The "default" package. For workspace, it's the top-level one if exists.
           ├───mypkg1: package 'rust_mypkg1-0.1.0'            # Crate `mypkg1` with `release` profile.
           ├───mypkg1-dev: package 'rust_mypkg1-debug-0.1.0'  # Crate `mypkg1` with `dev` profile.
           ├───mypkg2: package 'rust_mypkg2-0.1.0'            # etc.
           └───mypkg2-dev: package 'rust_mypkg2-debug-0.1.0'
   ```
4. Run `nix build .#<pkgname>` to build your package. Built binaries (if any) will be placed in `./result/bin`, and the library will be in `./result/lib`.
5. Have fun!

## Example `flake.nix` structure

A template `flake.nix` with common setup are below. It's mostly the same as the generated one, except that the helper `noc` will scan the workspace and discover all external registries and git dependencies for you.

```nix
{
  description = "My Rust packages";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs";
    flake-utils.url = "github:numtide/flake-utils";
    rust-overlay = {
      url = "github:oxalica/rust-overlay";
      inputs.nixpkgs.follows = "nixpkgs";
      inputs.flake-utils.follows = "flake-utils";
    };
    nocargo = {
      url = "github:oxalica/nocargo";
      inputs.nixpkgs.follows = "nixpkgs";

      # See below.
      # inputs.registry-crates-io.follows = "registry-crates-io";
    };

    # Optionally, you can explicitly import crates.io-index here.
    # So you can `nix flake update` at any time to get cutting edge version of crates,
    # instead of waiting `nocargo` to dump its dependency.
    # Otherwise, you can simply omit this to use the locked registry from `nocargo`,
    # which is updated periodically.
    # registry-crates-io = { url = "github:rust-lang/crates.io-index"; flake = false; };
  };

  outputs = { nixpkgs, flake-utils, rust-overlay, nocargo, ... }@inputs:
    flake-utils.lib.eachSystem [ "x86_64-linux" ] (system:
      let
        # The entry API to make Nix derivations from your Rust workspace or package.
        # The output of it consists of profile names, like `release` or `dev`, each of which is
        # a attrset of all member package derivations keyed by their package names.
        ws = nocargo.lib.${system}.mkRustPackageOrWorkspace {
          # The root directory, which contains `Cargo.lock` and top-level `Cargo.toml`
          # (the one containing `[workspace]` for workspace).
          src = ./.;

          # If you use registries other than crates.io, they should be imported in flake inputs,
          # and specified here. Note that registry should be initialized via `mkIndex`,
          # with an optional override.
          # extraRegistries = {
          #   "https://example-registry.org" = nocargo.lib.${system}.mkIndex inputs.example-registry {};
          # };

          # If you use crates from git URLs, they should be imported in flake inputs,
          # and specified here.
          # gitSrcs = {
          #   "https://github.com/some/repo" = inputs.example-git-source;
          # };

          # Use the latest stable release of rustc. Fallback to nixpkgs' rustc if omitted.
          rustc = rust-overlay.packages.${system}.rust;
        };
      in rec {
        # For convenience, we hoist derivations of `release` and `dev` profile for easy access,
        # with `dev` packages postfixed by `-dev`.
        # You can export different packages of your choice.
        packages = ws.release
          // nixpkgs.lib.mapAttrs' (name: value: { name = "${name}-dev"; inherit value; }) ws.dev;
      });
}
```

## FAQ

### Comparison with [cargo2nix] and [naersk]?

Main differences are already clarified [on the top](#nocargo%3A-cargo-in-nix).

`nocargo` is inspired by `cargo2nix` and `buildRustCrate` in `nixpkgs`. We are more like `cargo2nix` while the generation part is implemented by pure Nix, but less like `naersk` which is a wrapper to call `cargo` to build the package inside derivations.

In other words, we and `cargo2nix` use Nix as a *Build System*, while `nearsk` use Nix as a *Package Manager* or *Packager*.

<details>
<summary>
Detail comparison of nocargo, cargo2nix/buildRustCrate, naersk and buildRustPackage

</summary>

| | nocargo | [cargo2nix]/`buildRustCrate` | [naersk] | `buildRustPackage` |
|-|-|-|-|-|
| Depend on `cargo` | Updating `Cargo.lock` | Updating & generating & building | Updating & vendoring & building | Building |
| Derivation granularity | Per crate | Per crate | Per package + one dependency closure | All in one |
| Crate level sharing | ✔️ | ✔️ | ✖ | ✖ |
| Binary substitution per crate | Planned | Not implemented | ✖ | ✖ |
| Code generation | ✖ | ✔️ | ✖ | ✖ |
| Edit workspace & rebuild | Rebuild leaf crates | Rebuild leaf crates | Rebuild leaf crates | Refetch and rebuild all crates |
| Edit dependencies & rebuild | Rebuild changed crates (refetch if needed) | Refetch, regenerate and rebuild changed crates | Refetch and rebuild all crates | Refetch and rebuild all crates |
| Offline rebuild as long as | Not adding unfetched crate dependency | Not adding unfetched crate dependency | Not changing any dependencies | ✖ |

</details>

### But why pure Nix build system?

- Sharing through fine-grained derivations between all projects, not just in one workspace.
- Binary substitution per crate (planned).
  No need for global `target_dir`/`CARGO_TARGET_DIR` or [sccache].
- Easy `nixpkgs` integration for non-Rust package dependencies, cross-compilation (planned) and package overriding.
- More customizability: per-crate `rustc` flags tweaking, arbitrary crate patching, force dynamic linking and more.

### Can I really throw away `cargo`?

Sorry, currently no. :crying_cat_face: Updating of `Cargo.lock` still relies on `cargo`.
This can happen when creating a new project or changing dependencies.
We are mainly using `cargo`'s SAT solver to pin down the dependency graph.

It's *possible* to implement it ourselves, but not yet, due to the complexity.

## License

MIT Licensed.

[nixpkgs]: https://github.com/NixOS/nixpkgs
[naersk]: https://github.com/nix-community/naersk
[cargo2nix]: https://github.com/cargo2nix/cargo2nix
[sccache]: https://github.com/mozilla/sccache
