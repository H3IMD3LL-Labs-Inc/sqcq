# A low-level io_uring powered file ingestion engine for Linux

## Overview
sqcq (Submission Queue -> Completion Queue) is a Linux-only Rust library that provides a safe, opinionated, and production-ready abstraction over io_uring for high throughput file reading workloads.

This library is designed to be used as a shared I/O engine by higher-level systems such as:
- multi-file tailers
- log ingestion agents
- streaming ETL pipelines
- event collectors
- filesystem-backed queues

sqcq intentionally does not attempt to be async/await-native. Instead, it exposes a kernel-asynchronous, user-controlled polling model that intergrates cleanly with:
- dedicated I/O threads
- actor systems
- custom runtimes
- message-passing pipelines

If you want a Tokio-style API, this is no that library.

## Design Goals
1. One io_uring instance per engine
   - No per-request rings
   - No hidden syscalls
2. Explicit control over backpressure
   - Bounded submission queues
   - No unbounded buffering
3. Zero-copy (where possible)
   - Optional fixed buffers
   - Optional registered files
4. Predictable performance
   - No internal spawning
   - No implicit blocking
5. Safe by default
   - Unsafe code is fully encapsulated
   - All lifetimes are owned and explicit

## Non-Goals
scqc DOES NOT:
- Provide an async/await interface
- Hide io_uring semantics
- Automatically retry or mask kernel errors
- Implement scheduling or fairness
- Replace a full async runtime

These are responsibilities of the consumer

## Supported Platforms
- Linux kernel 5.10+
- x86_64 and aarch64
- `NOTE:` Requires io_uring support enabled in the kernel

This library will not compile on non-Linux platforms.

## Core Concepts
If you understand these, then congratulations, you understand the library

1. The Ring
An io_uring instance is a pair of shared memory queues between user space and the kernel:
- Submission Queue (SQ): requests go in
- Completion Queue (CQ): results come out

sqcq owns exactly one ring per engine instance. The ring is created once, lives for the lifetime of the engine and is polled explicitly by the user.

2. Submissions
A submission represents a request made to the kernel, such as:
- reading from a file descriptor
- reading at a specific offset
- reading into a specific buffer

Submissions are prepared by the library (sqcq), pushed into the submission queue and flushed explicitly to the kernel.

3. Completions
A completion is the kernel's response to a submission. Each completion:
- corresponds to exactly one submission
- includes a result code (bytes read or -errno)
- includes user provided metadata

Completions are retrieved by polling

4. Buffers
The kernel writes directly into user-provided memory.

sqcq supports:
- heap-allocated buffers
- optional fixed (registered) buffers for high-throughput use

Buffers are owned, tracked and reclaimed safely.

5. Files
Files may be used directly via raw file descriptors or optionally registered with the kernel as fixed files.

Fixed files reduce per-request overhead and are recommended for hot paths.

## Structure
### Public API:
- Engine: Owns the io_uring instance and global configuration
- FileHandle: Represents a file registered with the engine
- Buffer: Represents owned memory that the kernel may write into
- Completion: Represents a completed I/O operation

### Internal Modules
- ring: io_uring setup and lifecycle
- submission: SQE construction
- completion: CQE parsing
- buffer: memory safety and ownership
- error: error types and errno mappings

## Basic Usage
1. Create the engine
   - Configure the queue depth
   - Configure polling strategy
   - Optionally enable fixed buffers and files

2. Register files
   - Open files normally
   - Register them with the engine
   - Receive lightweight file handles

3. Submit reads
   - Request reads explicitly
   - Control offsets and buffer sizes
   - Associate user metadata if needed

4. Poll Completions
   - Poll the completion queue
   - Handle results
   - Reuse buffers

Nothing is implicit. Nothing happens in the background.

## Error Handling
sqcq exposes kernel errors directly:
- All kernel error codes are returned as structures Rust errors
- No retries are performed automatically
- No errors are swallowed

This is intentional. If the kernel returns EIO, EINVAL, or ENOMEM, the caller must decide what to do.

## Threading Model
sqcq is thread-safe but not thread-magical:
- The engine may be shared across threads
- Submission and completion may be performed from different threads
- This library DOES NOT spawn threads internally

A common pattern is:
- One dedicated I/O thread
- Many producers sending read requests
- Completions forwarded downstream via channels

## Performance Notes
- Creating an io_uring instance is expensive, but this library does it once.
- Queue depth should match expected in-flight I/O.
- Fixed buffers and files significantly reduce overhead.
- Polling beats blocking in high-throughput scenarios.

This library is optimized for:
- Steady-state throughput
- Predictable latency
- Sustained load

## Safety Guarantees
sqcq guarantees:
- No use-after-free of buffers
- No use of closed file descriptors
- No data races in shared ring memory

All unsafe code in this library is:
- minimal
- localized
- documented

If you only use the public API, you should not need to write any unsafe code.

## Project Status
sqcq is under active development. The API is unstable, experimental and expected to evolve rapidly. Breaking changes may occur until version 1.0.

Currently sqcq is built for use in another project of ours. See [VES](https://github.com/H3IMD3LL-Labs-Inc/VES). Despite the library being intended for general use, most changes may be guided by the development of our `VES` tool.

## Final Notes
sqcq is intentionally low-level. If you are uncomfortable reasoning about the following, this library may not be for you:
- File descriptors
- Buffer ownership
- Kernel queues
- Backpressure
