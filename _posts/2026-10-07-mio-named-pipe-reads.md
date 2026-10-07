---
layout: post
title: "Why Windows named pipe reads in mio stopped at 4 KiB"
date: 2026-10-07
---

In December 2022, LeonarddeR [reported](https://github.com/tokio-rs/tokio/issues/5307) that a Tokio named pipe server on Windows never read more than 4096 bytes at a time. His buffer had 64 KiB of capacity. A client sent 8192 bytes, and the server got two chunks of 4096. In October 2026 a fix landed in mio, the I/O layer under Tokio, as [mio#2022](https://github.com/tokio-rs/mio/pull/2022). This post covers why the cap existed, why earlier attempts stalled, how the fix works, and how it was checked without a local Windows machine.

## The symptom, measured again

A 2025 rework of the pipe's state handling ([mio#1903](https://github.com/tokio-rs/mio/pull/1903)) might have changed things, and Darksonn [asked exactly that](https://github.com/tokio-rs/tokio/issues/5307#issuecomment-3421048262). Nobody had checked. So the first step was fresh numbers.

The [check posted on the issue](https://github.com/tokio-rs/tokio/issues/5307#issuecomment-5928512550) used Windows 11, Tokio 1.53.1 and mio 1.2.3, and mio master. A client sent 64 KiB in one `write_all`, and the server read with a 64 KiB buffer. Every `read`, `read_buf` and `try_read` call returned exactly 4096 bytes, so the server needed 16 calls. A blocking `std::fs::File` client on the same kind of pipe got all 65536 bytes in one read. So the limit was in mio, not in Windows.

## Why mio owns the read buffer

On Windows, mio drives named pipes with overlapped I/O on an I/O completion port (IOCP). You hand the kernel a buffer and the read finishes later. Until then, the kernel may write into that buffer at any time.

Rust's `Read::read` only borrows the caller's buffer for one call. So mio can't give that buffer to the kernel. Instead it keeps one read pending on a buffer it owns. When that read completes, mio reports the pipe as readable, and the next `read` call copies data out. Thomasdezeeuw [explained this](https://github.com/tokio-rs/tokio/issues/5307#issuecomment-1361394729) in the original thread.

The result: the size of each read from the pipe is whatever mio asks for, not what the caller asks for.

## Why it was 4 KiB

mio takes that buffer from a small pool per pipe. New buffers were created with `Vec::with_capacity(4 * 1024)`, and the read asked the kernel for exactly the buffer's capacity. So on a pipe that only reads, no single read could return more than 4096 bytes, whatever the caller passed in.

(The pool is shared with writes. A buffer that grew during a large write could come back as a read buffer, so the cap could change with the pool's history. The fix makes the read size explicit, which also removes that quirk.)

A small size keeps one cost low: a read is always pending on every open pipe, so its buffer is memory held even by idle connections.

## Earlier attempts

The cap was known before this issue, from the other direction. In 2022, [mio#1582](https://github.com/tokio-rs/mio/issues/1582) asked for *smaller* reads. mio-serial and tokio-serial wrap serial ports in mio's `NamedPipe`, and on a serial port a read can wait until its buffer is full.

Thomasdezeeuw wrote [mio#1608](https://github.com/tokio-rs/mio/pull/1608), a `NamedPipe::set_buffer_size` method. Two users confirmed it helped. He closed it in 2023 because "it's likely not the API we want". In the Tokio thread he gave the reason: it exposes an implementation detail, since no other mio type uses a buffer pool. He [suggested](https://github.com/tokio-rs/tokio/issues/5307#issuecomment-1361281847) using "the length/capacity of the buffer passed in" instead. The reporter tried a proof of concept, but it still returned 4096 byte chunks. In 2024 the serial port issue was [closed for inactivity](https://github.com/tokio-rs/mio/issues/1582#issuecomment-2156493847): "It seems people want this, but no one is willing to put in the work."

So the direction had been agreed since 2022. What was missing was the work: the edge cases, the tests, and a way to run them on Windows.

## The fix

[mio#2013](https://github.com/tokio-rs/mio/pull/2013), merged shortly before, added vectored reads and writes to `NamedPipe`. It moved `read` and `read_vectored` onto one shared function, `read_with`. That made this fix a change in one place.

Every read from the pipe after the first is submitted inside a `read` call, right after the caller has taken the last of the previous data. At that moment mio knows how big the caller's buffer is. `read_with` now records that length (the sum of all buffers for `read_vectored`), and the next read from the pipe uses it:

```rust
// Size the next read from the pipe like the caller's buffer, so it can
// be filled in one call.
state.read_size = len.clamp(DEFAULT_BUF_SIZE, self.inner.max_read_size);
```

The bounds have two reasons. The minimum stays at 4 KiB, so a caller that reads a 4 byte length prefix doesn't shrink every read from the pipe to 4 bytes. The maximum is 64 KiB, the pipe buffer size that both mio's `NamedPipe::new` and Tokio's `ServerOptions` use. It also limits the memory held by the read that is always pending. Writes are unchanged.

## Not making serial ports worse

Larger reads would have made mio#1582 worse. mio-serial sets the port's timeouts to [a 1 ms gap between bytes and no total timeout](https://github.com/berkowski/mio-serial/blob/2e2af528994e082a7246ee42b268fa5a536bf898/src/lib.rs#L820-L825). With those settings, a read on a continuous stream finishes only when its buffer is full. At 115200 baud a 4 KiB read takes about 0.36 seconds to fill. A 64 KiB read would take about 5.7 seconds.

So the change is limited to real pipes. Only handles that `GetFileType` reports as `FILE_TYPE_PIPE` get larger reads. Every other handle keeps the old 4 KiB:

```rust
let max_read_size = if GetFileType(handle as HANDLE) == FILE_TYPE_PIPE {
    MAX_READ_SIZE
} else {
    DEFAULT_BUF_SIZE
};
```

This gate was not tested with a real serial device.

## Proving it without a Windows machine

The fix was written on macOS, where these tests can't run. Locally, only compiling and linting for the Windows target was possible. The proof had to come from CI.

There are three new tests in `tests/win_named_pipe.rs`: a 64 KiB buffer (plain and vectored), a buffer size that changes between reads, and a message over 4 KiB in message mode. Each uses a blocking client whose `write_all` returns once all the data is read or sits in the pipe's buffer, so the tests don't depend on timing.

The important step was showing that the tests fail for the right reason before showing that they pass. GitHub Actions also runs on forks, so the branch first went to two PRs inside the fork:

- [waltuov/mio#3](https://github.com/waltuov/mio/pull/3): only the new tests, on unchanged master. The Windows, MinimalVersions (Windows) and Wine jobs failed on exactly those three tests, with the numbers the old code predicts: `got 8192 of 65536 bytes`, `3072` instead of `16384`, and `4096` instead of `10000`.
- [waltuov/mio#4](https://github.com/waltuov/mio/pull/4): the tests plus the fix. Every job passed, including Windows, Wine and the MSRV build on Windows.

Then the PR went upstream. GlenDC, who reviews Windows changes for mio, ran the tests on Windows and [approved](https://github.com/tokio-rs/mio/pull/2022#pullrequestreview-5433302697): "Tested locally on Windows: the new tests fail on master and pass consistently here." Thomasdezeeuw merged it eight minutes later, about 25 hours after it was opened.

## What's still open

The fix has a known gap, stated in the PR. The first read of a new pipe is submitted when the pipe is registered or connects, before any `read` call. So it stays at 4 KiB. The issue's exact case, 8192 bytes right after connecting, still arrives in two reads. Later reads are sized like the caller's buffer. That is why the PR says "For tokio-rs/tokio#5307" and not "Closes", and the issue is still open. The change is also not in a mio release yet. The latest tag, v1.2.4, predates both PRs.

GlenDC has a better long term plan, described in his review. First he is landing a rework that gives each named pipe operation its own `OVERLAPPED` (the fix for [mio#1983](https://github.com/tokio-rs/mio/issues/1983)). Then he wants to try what libuv does: a zero byte read for readiness, then a read straight into the caller's buffer. That sizes every read by the current call, drops the internal copy, and covers the first read too. It would replace this sizing, which he called "a clear improvement until then". That order makes sense. The clamp is a bridge, and the zero byte read removes the problem it works around.

## Thanks

Thanks to [LeonarddeR](https://github.com/tokio-rs/tokio/issues/5307) for the report and the first proof of concept, [Darksonn](https://github.com/tokio-rs/tokio/issues/5307#issuecomment-1361274555) for finding the line in 2022, [Thomasdezeeuw](https://github.com/tokio-rs/tokio/issues/5307#issuecomment-1361281847) for the design and for merging both PRs, [mlsvrts and vadixidav](https://github.com/tokio-rs/mio/issues/1582) for the serial port reports that shaped the gate, and [GlenDC](https://github.com/tokio-rs/mio/pull/2022#pullrequestreview-5433302697) for the review and the Windows test run.
