- Feature Name: `mergable_rustdoc_cross_crate_info`
- Start Date: 2024-06-18
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

Mergeable cross-crate information in rustdoc. Facilitates the generation of documentation indexes in workspaces with many crates by allowing each crate to write to an independent output directory. Final documentation is rendered with a lightweight merge step. Configurable with command-line flags, this proposal writes a `crate-info.json` file to hold pre-merge cross-crate information. Currently, rustdoc requires global mutable access to a single output directory to generate cross-crate information, which is an obstacle to integrating rustdoc in build systems that enforce the independence of build actions.

# Motivation

The main goal of this proposal is to facilitate users producing a documentation bundle of every crate in a large environment. When a crate needs to be re-documented, only a relatively lightweight merge step will be needed to produce an updated documentation bundle. This proposal is to facilitate the creation and updating of these bundles.

This proposal also targets documenting individual crates and their dependencies in non-cargo build systems. As will be explained, doc targets in non-cargo build systems often do not support cross-crate information.

There are some files in the rustdoc output directory that are read and overwritten during every invocation of rustdoc. This proposal refers to these files as **cross-crate information**, or **CCI**, as in <https://rustc-dev-guide.rust-lang.org/rustdoc.html#multiple-runs-same-output-directory>.

Build systems may run build actions in a distributed environment across separate logical filesystems. It might also be desirable to run rustdoc in a lock-free parallel mode, where every rustdoc process writes to a disjoint set of files.

Cargo fully supports cross-crate information, at the cost of requiring global read-write access to the doc root (`target/doc`). There are significant scalability issues with this approach.

Rustdoc needing global mutable access to the files that encode this cross-crate information has implications for caching, reproducible builds, and content hashing. By adding an option to avoid this mutation, rustdoc will serve as a first-class citizen in non-cargo build systems.

These considerations motivate adding an option for outputting partial CCI (parts), which are merged (linked) with a later step.

# Guide-level explanation

In this example, there is a crate `t` which defines a trait `T`, and a crate `s` which defines a struct `S` that implements `T`. Our goal in this demo is for `S` to appear as in implementer in `T`'s docs, even if `s` and `t` are documented independently. This guide will be assuming that we want a crate `i` that serves as our documentation index. See the Unresolved questions section for ideas that do not require an index crate.

```shell
mkdir -p t/src s/src i/src merged/doc
echo "pub trait T {}" > t/src/lib.rs
echo "pub struct S; impl t::T for S {}" > s/src/lib.rs
MERGED=file://$(realpath merged/doc)
```

[Actively use](https://doc.rust-lang.org/rustc/command-line-arguments.html#--extern-specify-where-an-external-library-is-located) `t` and `s` in `i`. The `extern crate` declarations are not needed if the crates are otherwise referenced in the index; intra-doc links are enough.

```shell
echo "extern crate t; extern crate s;" > i/src/lib.rs
```

Compile the crates.

```shell
rustc --crate-name=t --crate-type=lib --edition=2021 --emit=metadata --out-dir=t/target t/src/lib.rs
rustc --crate-name=s --crate-type=lib --edition=2021 --emit=metadata --out-dir=s/target --extern t=t/target/libt.rmeta s/src/lib.rs
rustc --crate-name=i --crate-type=lib --edition=2021 --emit=metadata --out-dir=i/target --extern s=s/target/libs.rmeta --extern t=t/target/libt.rmeta -L t/target i/src/lib.rs
```

Document `s` and `t` independently, providing `--merge=none` and `--write-info-json`

```shell
rustdoc -Z unstable-options --crate-name=t --crate-type=lib --edition=2021 --out-dir=t/target/doc --extern-html-root-url t=$(MERGED) --merge=none --write-info-json=t/target/doc.parts/t/crate-info.json t/src/lib.rs
rustdoc -Z unstable-options --crate-name=s --crate-type=lib --edition=2021 --out-dir=s/target/doc --extern-html-root-url s=$(MERGED) --extern-html-root-url t=$(MERGED) --merge=none --write-info-json=s/target/doc.parts/s/crate-info.json --extern t=t/target/libt.rmeta s/src/lib.rs
```

Link everything with a final invocation of rustdoc on `i`. We will provide `--merge=write-only` and `--include-info-json=<path>`. See the Reference-level explanation about these flags.

```shell
rustdoc -Z unstable-options --crate-name=i --crate-type=lib --edition=2021 --enable-index-page --out-dir=i/target/doc --extern-html-root-url s=$(MERGED) --extern-html-root-url t=$(MERGED) --extern-html-root-url i=$(MERGED) --merge=write-only --include-info-json=t/target/doc.parts/t/crate-info.json --include-info-json=s/target/doc.parts/s/crate-info.json --extern t=t/target/libt.rmeta --extern s=s/target/libs.rmeta -L t/target i/src/lib.rs
```

Merge the docs with `cp`. This can be avoided if `--out-dir=$(MERGED)` is used for all of the rustdoc calls. We copy here to illustrate that documenting `s` is independent of documenting `t`, and could happen on separate machines.

```shell
cp -r s/target/doc/* t/target/doc/* i/target/doc/* merged/doc
```

Browse `merged/doc/index.html` with cross-crate information.

In general, instead of two crates in the environment (`s` and `t`) you could have thousands. Upon any changes, only the index and the crates that are changed have to be re-documented.

<details>
<summary>Click here for a directory listing after running the example above.</summary>

<pre>
$ tree . -a
.
├── i
│   ├── src
│   │   └── lib.rs
│   └── target
│       ├── doc
│       │   ├── crates.js
│       │   ├── help.html
│       │   ├── i
│       │   │   ├── all.html
│       │   │   ├── fn.add.html
│       │   │   ├── index.html
│       │   │   └── sidebar-items.js
│       │   ├── index.html
│       │   ├── .lock
│       │   ├── search.desc
│       │   │   ├── i
│       │   │   │   └── i-desc-0-.js
│       │   │   ├── s
│       │   │   │   └── s-desc-0-.js
│       │   │   └── t
│       │   │       └── t-desc-0-.js
│       │   ├── settings.html
│       │   ├── src
│       │   │   └── i
│       │   │       └── lib.rs.html
│       │   ├── src-files.js
│       │   ├── static.files
│       │   │   ├── COPYRIGHT-23e9bde6c69aea69.txt
│       │   │   ├── favicon-2c020d218678b618.svg
│       │   │   ├── favicon-32x32-422f7d1d52889060.png
│       │   │   ├── FiraSans-LICENSE-db4b642586e02d97.txt
│       │   │   ├── FiraSans-Medium-8f9a781e4970d388.woff2
│       │   │   ├── FiraSans-Regular-018c141bf0843ffd.woff2
│       │   │   ├── LICENSE-APACHE-b91fa81cba47b86a.txt
│       │   │   ├── LICENSE-MIT-65090b722b3f6c56.txt
│       │   │   ├── main-20a3ad099b048cf2.js
│       │   │   ├── NanumBarunGothic-0f09457c7a19b7c6.ttf.woff2
│       │   │   ├── NanumBarunGothic-LICENSE-18c5adf4b52b4041.txt
│       │   │   ├── normalize-76eba96aa4d2e634.css
│       │   │   ├── noscript-df360f571f6edeae.css
│       │   │   ├── rustdoc-dd39b87e5fcfba68.css
│       │   │   ├── rust-logo-151179464ae7ed46.svg
│       │   │   ├── scrape-examples-ef1e698c1d417c0c.js
│       │   │   ├── search-0fe7219eb170c82e.js
│       │   │   ├── settings-4313503d2e1961c2.js
│       │   │   ├── SourceCodePro-It-1cc31594bf4f1f79.ttf.woff2
│       │   │   ├── SourceCodePro-LICENSE-d180d465a756484a.txt
│       │   │   ├── SourceCodePro-Regular-562dcc5011b6de7d.ttf.woff2
│       │   │   ├── SourceCodePro-Semibold-d899c5a5c4aeb14a.ttf.woff2
│       │   │   ├── SourceSerif4-Bold-a2c9cd1067f8b328.ttf.woff2
│       │   │   ├── SourceSerif4-It-acdfaf1a8af734b1.ttf.woff2
│       │   │   ├── SourceSerif4-LICENSE-3bb119e13b1258b7.md
│       │   │   ├── SourceSerif4-Regular-46f98efaafac5295.ttf.woff2
│       │   │   ├── src-script-e66d777a5a92e9b2.js
│       │   │   └── storage-118b08c4c78b968e.js
│       │   └── trait.impl
│       │       ├── core
│       │       │   ├── marker
│       │       │   │   ├── trait.Freeze.js
│       │       │   │   ├── trait.Send.js
│       │       │   │   ├── trait.Sync.js
│       │       │   │   └── trait.Unpin.js
│       │       │   └── panic
│       │       │       └── unwind_safe
│       │       │           ├── trait.RefUnwindSafe.js
│       │       │           └── trait.UnwindSafe.js
│       │       └── t
│       │           └── trait.T.js
│       └── libi.rmeta
├── merged
│   └── doc
│       ├── crates.js
│       ├── help.html
│       ├── i
│       │   ├── all.html
│       │   ├── fn.add.html
│       │   ├── index.html
│       │   └── sidebar-items.js
│       ├── index.html
│       ├── s
│       │   ├── all.html
│       │   ├── index.html
│       │   ├── sidebar-items.js
│       │   └── struct.S.html
│       ├── search.desc
│       │   └── i
│       │       └── i-desc-0-.js
│       ├── settings.html
│       ├── src
│       │   ├── i
│       │   │   └── lib.rs.html
│       │   ├── s
│       │   │   └── lib.rs.html
│       │   └── t
│       │       └── lib.rs.html
│       ├── src-files.js
│       ├── static.files
│       │   ├── COPYRIGHT-23e9bde6c69aea69.txt
│       │   ├── favicon-2c020d218678b618.svg
│       │   ├── favicon-32x32-422f7d1d52889060.png
│       │   ├── FiraSans-LICENSE-db4b642586e02d97.txt
│       │   ├── FiraSans-Medium-8f9a781e4970d388.woff2
│       │   ├── FiraSans-Regular-018c141bf0843ffd.woff2
│       │   ├── LICENSE-APACHE-b91fa81cba47b86a.txt
│       │   ├── LICENSE-MIT-65090b722b3f6c56.txt
│       │   ├── main-20a3ad099b048cf2.js
│       │   ├── NanumBarunGothic-0f09457c7a19b7c6.ttf.woff2
│       │   ├── NanumBarunGothic-LICENSE-18c5adf4b52b4041.txt
│       │   ├── normalize-76eba96aa4d2e634.css
│       │   ├── noscript-df360f571f6edeae.css
│       │   ├── rustdoc-dd39b87e5fcfba68.css
│       │   ├── rust-logo-151179464ae7ed46.svg
│       │   ├── scrape-examples-ef1e698c1d417c0c.js
│       │   ├── search-0fe7219eb170c82e.js
│       │   ├── settings-4313503d2e1961c2.js
│       │   ├── SourceCodePro-It-1cc31594bf4f1f79.ttf.woff2
│       │   ├── SourceCodePro-LICENSE-d180d465a756484a.txt
│       │   ├── SourceCodePro-Regular-562dcc5011b6de7d.ttf.woff2
│       │   ├── SourceCodePro-Semibold-d899c5a5c4aeb14a.ttf.woff2
│       │   ├── SourceSerif4-Bold-a2c9cd1067f8b328.ttf.woff2
│       │   ├── SourceSerif4-It-acdfaf1a8af734b1.ttf.woff2
│       │   ├── SourceSerif4-LICENSE-3bb119e13b1258b7.md
│       │   ├── SourceSerif4-Regular-46f98efaafac5295.ttf.woff2
│       │   ├── src-script-e66d777a5a92e9b2.js
│       │   └── storage-118b08c4c78b968e.js
│       └── t
│           ├── all.html
│           ├── index.html
│           ├── sidebar-items.js
│           └── trait.T.html
├── s
│   ├── src
│   │   └── lib.rs
│   └── target
│       ├── doc
│       │   ├── help.html
│       │   ├── .lock
│       │   ├── s
│       │   │   ├── all.html
│       │   │   ├── index.html
│       │   │   ├── sidebar-items.js
│       │   │   └── struct.S.html
│       │   ├── settings.html
│       │   └── src
│       │       └── s
│       │           └── lib.rs.html
│       ├── doc.parts
│       │   └── s
│       │       └── crate-info.json
│       └── libs.rmeta
└── t
    ├── src
    │   └── lib.rs
    └── target
        ├── doc
        │   ├── help.html
        │   ├── .lock
        │   ├── t
        │   │   ├── all.html
        │   │   ├── index.html
        │   │   ├── sidebar-items.js
        │   │   └── trait.T.html
        │   ├── settings.html
        │   └── src
        │       └── t
        │           └── lib.rs.html
        ├── doc.parts
        │   └── t
│       │       └── crate-info.json
        └── libs.rmeta
</pre>

</details>


# Reference-level explanation

Currently, cross-crate information is written during the invocation of the `write_shared` function in [write_shared.rs](https://github.com/rust-lang/rust/blob/04ab7b2be0db3e6787f5303285c6b2ee6279868d/src/librustdoc/html/render/write_shared.rs#L47). This proposal does not add any new CCI or change their contents (modulo sorting order, whitespace).

The existing cross-crate information files, like `search-index.js`, all are lists of elements, rendered in an specified way (e.g. as a JavaScript file with a JSON array or an HTML index page containing an unordered list). The current rustdoc (in `write_shared`) pushes the current crate's version of the CCI into the one that is already found in `doc`, and renders a new version. The rest of the proposal uses the term **part** to refer to the pre-merged, pre-rendered element of the CCI.

## New file: `crate-info.json`

The `crate-info.json` file contains the unmerged contents of a single crates' version of their corresponding CCI. This file is written if the flag `--write-info-json=path/to/crate-info.json` is provided.

`crate-info.json` is a JSON object with several key-value pairs. Every value is a JSON array. Every element of the JSON array is a two-element array: a destination filename relative to the doc root, and the representation of the part. The representation of the part depends on the type of CCI that it describes.

* key: `src-files-js`, for `doc/src-files.js`

This part is the JSON representation of the source index that is later stored in a `srcIndex` global variable.

* key: `search-index-js`, for `doc/search-index.js`

This part is the JSON encoded search index, before it has been installed in `search-index.js`.

* key: `all-crates`, for `doc/crates.js`

This part is the crate name.

* key: `crates-index`, for `doc/index.html`

This part is also the crate name. It represents a different kind of CCI because it is written to a `doc/index.html`, and rendered as an HTML document instead as JSON. In principal, we could use this part to add more information to the crates index `doc/index.html` (the first line of the top level crate documentation, for example).

* key: `type-impl`, for `doc/type.impl/**/*.js`

This part is a two element array with the crate name and the JSON representation of a type implementation.

* key: `trait-impl`, for `doc/trait.impl/**/*.js`

This part is a two element array with the crate name and the JSON representation of a trait implementation.

## New flag: `--write-info-json=path/to/crate-info.json`

When this flag is provided, the unmerged parts for the current crate will be written to `path/to/crate-info.json`. A typical `<path to crate-info.json>` is `./target/doc.parts/<crate name>/crate-info.json`.

Crates `--include-info-json`ed will not appear in `crate-info.json`, as `crate-info.json` only includes the CCI parts for the current crate.

If this flag is not provided, no `crate-info.json` will be written.

## New flag: `--include-info-json=path/to/crate-info.json`

If this flag is provided, rustdoc will expect `path/to/crate-info.json` to be the `crate-info.json` file containing the parts for a crate. It will append these parts to the ones it will render in the doc root (`--out-dir`). The info that's included is not written to `crate-info.json`, as `crate-info.json` only holds the CCI parts for the current crate.

This flag is similar to `--extern-html-root-url` in that it only needs to be provided for externally documented crates. The flag `--extern-html-root-url` controls hyperlink generation. The hyperlink provided in `--extern-html-root-url` never accessed by rustdoc, and represents the final destination of the documentation. The new flag `--include-info-json` tells rustdoc where to search for the `crate-info.json` directory at documentation-time. It must not be a URL.

In the Guide-level explanation, for example, crate `i` needs to identify the location of `s`'s parts. Since they could be located in an arbitrary directory, `i` must be instructed on where to fetch them. In this example, `s`'s parts happen to be in `./s/target/doc.parts/s`, so rustdoc is called with `--include-info-json=s/target/doc.parts/s/crate-info.json`.

## New flag: `--merge=auto|none|write-only`

This flag controls two internal paramaters: `read_rendered_cci`, and `write_rendered_cci`.

When `write_rendered_cci` is active, rustdoc will output the rendered parts to the doc root (`--out-dir`). Rustdoc will generate files like `doc/search-index.js`, `doc/search.desc`, `doc/index.html`, etc if and only if this parameter is true.

When `read_rendered_cci` is active, rustdoc will look in the `--out-dir` for rendered cross-crate info files. These files will be used as the base. Any new parts that rustdoc generates with its current invocation and any parts fetched with `include-info-json` will be appended to these base files. When it is disabled, the cross-crate info files start empty and are populated with the current crate's info and any crates fetched with `--include-info-json`.

* `--merge=auto` (`read_rendered_cci && write_rendered_cci`) is the default, and reflects the current behavior of rustdoc. 
* `--merge=none` (`!read_rendered_cci && !write_rendered_cci`) means that rustdoc will ignore the cross-crate files in the doc root. Only generate item docs. 
* `--merge=write-only` (`!read_rendered_cci && write_rendered_cci`) outputs crate info based only on the current crate and `--include-info-json`'ed crates.
* A (`read_rendered_cci && !write_rendered_cci`) mode would be useless, since the data that is read would be ignored and not written.

## Merge step

This proposal is capable of addressing two primary use cases. It allows developers to enable CCI in these scenarios:
* Documenting a crate and its transitive dependencies in parallel in build systems that require build actions to be independent
* Producing a documentation index of a large number of crates, in such a way that if one crate is updated, only the updated crates and an index have to be redocumented. This scenario is demonstrated in the Guide-level explanation.

CCI is not automatically enabled in either situation. A combination of the `--include-info-json`, `--merge`, and `--write-info-json` flags are needed to produce this behavior. This RFC provides a minimal set of tools that allow developers of build systems, like Bazel and Buck2, to create rules for these scenarios.

Discussion of whether additional features should be included to facilitate this merge step can be found in Unresolved questions (Index crate).

## Compatibility

This RFC does not alter previous compatibility guarantees made about the output of rustdoc. In particular it does not stabilize the presence of the rendered cross-crate information files, their content, or the HTML generated by rustdoc.

In the same way that the [rustdoc HTML output is unstable](https://rust-lang.github.io/rfcs/2963-rustdoc-json.html#:~:text=The%20HTML%20output%20of%20rustdoc,into%20a%20different%20format%20impractical), the contents of the `crate-info.json` will be considered unstable. Between versions of rustdoc, breaking changes to the content of `crate-info.json` should be expected. Only the presence of a `crate-info.json` file is promised, under `--write-info-json`. Merging cross-crate information generated by disparate versions of rustdoc is not supported.

The implementation of the RFC itself is designed to produce only minimal changes to cross-crate info files and the HTML output of rustdoc. Exhaustively, the implementation is allowed to 
* Change the sorting order of trait implementations, type implementations, and other cross-crate info in the HTML output of rustdoc
* Add a comment on the last line of generated HTML pages, to store metadata relevant to appending items to them
* Refactor the JavaScript contents of cross-crate information files, in ways that do not change their overall behavior. If the JavaScript fragment declared an array called `ALL_CRATES` with certain contents, it will continue to do so.

Changes this minimal are intended to avoid breaking tools that use the output of rustdoc, like Cargo, docs.rs, and rustdoc's JavaScript frontend, in the near-term. Going forward, rustdoc will not make formal guarantees about the content of cross-crate info files.

# Drawbacks

The WIP may change the sorting order of the elements in the CCI. It does not change the content of the documentation, and is intended to work without modifying Cargo and docs.rs.

# Rationale and alternatives

Running rustdoc in parallel is essential in enabling the tool to scale to large projects. The approach implemented by Cargo is to run rustdoc in parallel by locking the CCI files. There are some workspaces where having synchronized access to the CCI is impossible. This proposal implements a reasonable approach to shared rustdoc, because it cleanly enables the addition of new kinds of CCI without changing existing documentation.

# Prior art

Prior art for linking and merging independently generated documentation was **not** identified in Javadoc, Godoc, Doxygen, Sphinx (intersphinx), nor any documentation system for other languages. Analogs of cross-crate information were not found, but a more thorough investigation or experience with other systems may be needed.

However, the issues presented here have been encountered in multiple build systems that interact with rustdoc. They limit the usefulness of rustdoc in large workspaces.

## Bazel

Bazel has `rules_rust` for building Rust targets and rustdoc documentation.

* <https://bazelbuild.github.io/rules_rust/rust_doc.html>
* <https://github.com/bazelbuild/rules_rust/blob/67b3571d7e5e341de337317d84a6bec6b9d02ed7/rust/private/rustdoc.bzl#L174>

It does not document crates' dependencies. `search-index.js`, for example, is both a dependency and an output file for rustdoc in multi-crate documentation workspaces. If it is declared as a dependency in this way, Bazel could not build docs for the members of an environment in parallel with a single output directory, as it strictly enforces hermiticity. For a recursive, parallel rustdoc to ever serve as a first-class citizen in Bazel, changes similar to the ones described in this proposal would be needed.

There is an [open issue](https://github.com/bazelbuild/rules_rust/issues/1837) raised about the fact that Bazel does not document crates dependencies. The comments in the issue discuss a pull request on Bazel that documents each crates dependencies in a separate output directory. It is noted in the discussion that this solution, being implemented under the current rustdoc, "doesn't scale well and it should be implemented in a different manner long term." In order to get CCI in a mode like this, rustdoc would need to adopt changes, like the ones in this proposal, for merging cross-crate information.

## Buck2

The Buck2 build system has rules for building and testing rust binaries and libraries. <https://buck2.build/docs/api/rules/#rust_library>

It has a subtarget, `[doc]`, for generating rustdoc for a crate.

You can provide `extern-html-root-url`. You can document all crates independently and manually merge them but no cross-crate information would be shared.

buck2 does not natively merge rustdoc from separate targets. The buck2 maintainers have a [proprietary search backend](https://rust-lang.zulipchat.com/#narrow/stream/266220-t-rustdoc/topic/mergable.20rustdoc.20proposal/near/445952204) that merges and parses `search-index.js` files from separately documented crates. Their proprietary tooling does not handle cross-crate trait implementations from upstream crates. By implementing this merging directly in rustdoc, we could avoid fragmentation and bring cross-crate information to more consumers.

<https://github.com/facebook/buck2/blob/d390e31632d3b4a726aaa2aaf3c9f1f63935e069/prelude/rust/build.bzl#L213>


## Ninja [(GN)](https://fuchsia.dev/fuchsia-src/development/build/build_system/intro) + Fuchsia

Currently, the Fuchsia project runs rustdoc on all of their crates to generate a [documentation index](https://fuchsia-docs.firebaseapp.com/rust/rustdoc_index/). This index is effectively generated as an [atomic step](https://cs.opensource.google/fuchsia/fuchsia/+/main:tools/devshell/contrib/lib/rust/rustdoc.py) in the build system. It takes [3 hours](https://ci.chromium.org/ui/p/fuchsia/builders/global.ci/firebase-docs/b8744777376580022225/overview) to document the ~2700 crates in the environment. With this proposal, building each crate's documentation could be done as separate build actions, which would have a number of benefits. These include parallelism, caching (avoid rebuilding docs unnecessarily), and robustness (automatically reject pull requests that break documentation).

# Unresolved questions

## Index crate?

Require users to generate documentation bundles via an index crate (current) vs. creating a new mode to allow rustdoc to run without a target crate (proposed)

If one would like to merge the documentation of several crates, we could continue to require users to provide an index crate, like [the fuchsia index](https://fuchsia-docs.firebaseapp.com/rust/rustdoc_index/). This serves as the target of the rustdoc invocation, and the landing page for the collected documentation. Supporting only this style of index would require the fewest changes. This is the mode described in the Guide-level explanation.

The proposition, to allow users of rustdoc the flexibility of not having to produce an index, is to allow rustdoc to be run in a mode where no target crate is provided. It would generate rendered cross-crate information based only on what is provided through `--include-info-json`. The source crates are provided to rustdoc, through a mechanism like `--extern`, rustdoc merges and writes the CCI, and copies the item and module links to the doc root. This would require more extensive changes, as rustdoc assumes that it is invoked with a target crate. This mode is somewhat analogous to the [example scraping mode](https://github.com/rust-lang/rfcs/blob/master/text/3123-rustdoc-scrape-examples.md). Having to create an index crate, that actively uses all of the crates in the environment, might prohibit the use of this feature in settings where users do not intend to produce an index, or where exhaustively listing all dependencies (to `--extern` them) is difficult.

## Unconditionally generating the `doc.parts` files?

Generate no extra files (current) vs. unconditionally creating `doc.parts` to enable more complex future CCI (should consider)

The current version of rustdoc performs merging by [collecting JSON](https://github.com/rust-lang/rust/blob/c25ac9d6cc285e57e1176dc2da6848b9d0163810/src/librustdoc/html/render/write_shared.rs#L166) blobs from the contents of the already-rendered CCI.
This proposal proposes to continue reading from the rendered cross-crate information under the default `--merge=auto`. It can also read `crate-info.json` files, under `--include-info-json`. However, there are several issues with reading from the rendered CCI that must be stated:
* Every rustdoc process outputs the CCI to the same doc root by default
* It is difficult to extract the items in a diverse set of rendered HTML files. This is anticipating of the CCI to include HTML files that, for example, statically include type+trait implementations directly
* Reading exclusively from `crate-info.json` is simpler than the existing `serde_json` dependency for extracting the blobs, as opposed to handwritten CCI-type specific parsing (current)
* With this proposal, there will be duplicate logic to read from both `doc.parts` files and rendered CCI.

[@jsha proposes](https://github.com/rust-lang/rfcs/pull/3662#issuecomment-2184077829) unconditionally generating and reading from `crate-info.json`, with no appending to the rendered crate info.

## Item links?

Require users to pass `--extern-html-root-url` on all external dependencies (current), vs. add a new flag to facilitate missing docs links being generated across all user crates (should consider)

Cross-crate links are an important consideration when merging documentation. rustdoc provides a flag, `--extern-html-root-url`, which can provide fine-grain control over the generated URL for documentation. We believe this is already sufficient for generating cross-crate links, and we will make no changes to the generation of item links. Example usage of this flag is in the Guide-level explanation.

A proposal is to provide more options to facilitate cross-crate links. For example, we could add a flag that implies `--extern-html-root-url` and `--include-info-json` on all crates passed through `--extern`. It could be something like `--extern-html-root-url-all-user-crates`. User crates would be taken to mean the transitive dependencies of the current crate, excluding the standard library. This is complicated by things like <https://doc.rust-lang.org/cargo/reference/unstable.html#build-std>.

## Reuse existing option?

Create a new flag, `--merge` (proposed), vs. use existing option `no_emit_shared`

There is a render option, `no_emit_shared`, which is used to conditionally generate the cross-crate information. It also controls the generation of static files, user CSS, etc.

This option is not configurable from the command line, and appears to be enabled unless rustdoc is run in its example scraping mode.

We could make it configurable from the command line, unconditionally generate `doc.parts`, and use it to gate the merging of CCI.

We could also make it configurable from the command line, and use it to gate the generation of `doc.parts` and the generation of all of the shared files.

We could also leave it as-is: always false unless we're scraping examples, and gate the generation of `doc.parts` and the generation of all of the shared files.

# Future possibilities

This change could begin to facilitate type aliases and the trait aliases being
statically compiled as part of the .html documentation, instead of being loaded
as separate JavaScript files. Each type and trait alias could be stored as an
HTML part, which are then merged into the regular documentation.

Another possibility is for `doc.parts` to be distributed on `docs.rs` along with the regular documentation. This would facilitate a mode where documentation of the dependencies could be downloaded externally, instead of being rebuilt locally.

A future possibility related to the index crate idea is to have an option for embedding user-specified HTML into the `--enable-index-page`'s HTML.

The changes in this proposal are intended to work with no changes to Cargo and docs.rs. However, there may be benefits to using `--merge=write-only` with Cargo, as it would remove the need for locking the output directory. More of the documentation process could happen in parallel, which may speed up execution time.
