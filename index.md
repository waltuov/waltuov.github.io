---
layout: home
title: selected open source work
list_title: writing
---

i work on rust build tooling, mostly cc-rs, plus tokio and mio on windows. these are the six merged fixes i'd point someone to first: what broke, what changed, how it was checked, and what the maintainers said.

## 1. mio: windows named pipe reads stuck at 4 KiB ([mio#2022](https://github.com/tokio-rs/mio/pull/2022))

- **problem:** every read from a windows named pipe returned at most 4096 bytes, even with a 64 KiB buffer and data waiting ([tokio#5307](https://github.com/tokio-rs/tokio/issues/5307), open since december 2022).
- **fix:** each read from the pipe is now sized like the caller's buffer, between 4 and 64 KiB, and only for real pipe handles, so serial ports keep their latency. builds on #2013.
- **evidence:** three new tests fail on master in ci (`got 8192 of 65536 bytes`) and pass with the fix.
- **maintainer:** GlenDC, mio's windows reviewer: "Tested locally on Windows: the new tests fail on master and pass consistently here." ([review](https://github.com/tokio-rs/mio/pull/2022#pullrequestreview-5433302697))

## 2. cc-rs: cross compiling masm files ([cc-rs#1975](https://github.com/rust-lang/cc-rs/pull/1975))

- **problem:** building a crate with masm `.asm` files for `x86_64-pc-windows-msvc` from linux or macos stopped at `failed to find tool "ml64.exe"` ([cc-rs#1022](https://github.com/rust-lang/cc-rs/issues/1022), open since april 2024).
- **fix:** a new `CC_MASM_ASM` env var picks the assembler, and x86 targets fall back to `llvm-ml` with the right `-m64` or `-m32` when `ml64.exe` and `ml.exe` aren't found. design agreed with the maintainer on the issue first.
- **evidence:** 10 new tests. 7 fail on main with the user's exact error, 3 guard behavior that must not change.
- **maintainer:** NobodyXu approved and asked for a follow-up on case-insensitive tool names: "I'd really appreciate that" ([comment](https://github.com/rust-lang/cc-rs/pull/1975#discussion_r4193446747))

## 3. tokio: stdout output lost at runtime shutdown ([tokio#8506](https://github.com/tokio-rs/tokio/pull/8506))

- **problem:** a write to `tokio::io::stdout()` without a flush could be silently dropped when the runtime shut down right after it ([tokio#7174](https://github.com/tokio-rs/tokio/issues/7174), open since february 2025).
- **fix:** stdout and stderr writes now run as mandatory blocking tasks, like `tokio::fs`, and a write after shutdown returns an error instead of doing nothing. stdin stays non-mandatory, so a pending read can't hold up shutdown. continues maminrayej's #7178, keeping those commits.
- **evidence:** a loom test that fails on master. a program writing 3 lines from each of 1000 runtimes lost 21 to 188 of 3000 lines per run on master, and none with the change.
- **maintainer:** ADD-SP: "Overall looks good to me" ([review](https://github.com/tokio-rs/tokio/pull/8506#pullrequestreview-5330140778))

## 4. wasm-bindgen: imports silently bound to the wrong js ([wasm-bindgen#5352](https://github.com/wasm-bindgen/wasm-bindgen/pull/5352))

- **problem:** two `inline_js` snippets exporting the same name shared one binding, so one module silently read the other's value ([wasm-bindgen#3878](https://github.com/wasm-bindgen/wasm-bindgen/issues/3878), open since march 2024).
- **fix:** the generated shim name for an import now hashes the snippet text. the same bug class is closed for `instanceof` checks, `static_string` and call attributes like `variadic` and `catch`.
- **evidence:** 10 runtime tests fail on main and pass with the fix. reference outputs change only in shim names and order.
- **maintainer:** guybedford: "Amazing, thank you for your patience on this one." ([review](https://github.com/wasm-bindgen/wasm-bindgen/pull/5352#pullrequestreview-5422660514))

## 5. mio: vectored i/o for windows named pipes ([mio#2013](https://github.com/tokio-rs/mio/pull/2013))

- **problem:** on windows, tokio's `try_read_vectored` and `try_write_vectored` for named pipes only processed the first buffer, because mio's `NamedPipe` fell back to std's default vectored methods ([tokio#6970](https://github.com/tokio-rs/tokio/issues/6970), open since november 2024).
- **fix:** writes copy every buffer into mio's owned write buffer, reads fill every buffer from the data mio already holds. plain and vectored calls share one path that never builds an `IoSlice`, since `IoSlice::new` panics above 4 GiB on windows.
- **evidence:** 4 tests, including message mode and buffers smaller than the data. the windows reviewer confirmed the first two fail on master.
- **maintainer:** GlenDC: "Your follow-up is even stronger than what I had in mind, so awesome to see." ([review](https://github.com/tokio-rs/mio/pull/2013#pullrequestreview-5407623061))

## 6. cc-rs: split compile, archive and link ([cc-rs#1972](https://github.com/rust-lang/cc-rs/pull/1972))

- **problem:** build scripts that change the compiled objects or build their own archive had to copy cc's archive and link logic by hand. requested by tgross35 in [cc-rs#1818](https://github.com/rust-lang/cc-rs/issues/1818), with rust-lang/rust#157914 as the motivating case.
- **fix:** `Build::create_archive` and a free function `emit_link_directives`, in the shape agreed on the issue. `compile` now calls both, so its commands and link lines stay the same.
- **evidence:** a subprocess test requires `compile` and the three step path to run the same commands and print the same cargo lines on linux-gnu, pauthtest, cuda and msvc.
- **maintainer:** NobodyXu approved after one review round: "Thank you!" ([review](https://github.com/rust-lang/cc-rs/pull/1972#pullrequestreview-5398326881))

[all merged prs](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Awaltuov+-user%3Awaltuov&type=pullrequests)
