---
title: Running neural networks in a custom kernel
slug: running-neural-networks-in-a-custom-kernel
type: experiment
status: draft
date: undated
updated: 2026-04-17
tags:
  - ml
  - kernel
  - c++
  - nn
  - niros
summary: Neural network code was ported into the kernel environment by replacing userspace and STL dependencies with kernel-safe build, math, allocation, and file access paths.
verification_status: unverified
verified_on:
---

# Running neural networks in a custom kernel

## Context

I copied my neural network code into the `nirOS` directory.

## Goal

- integrate the NN code into the kernel build
- remove userspace and STL dependencies that do not belong in a freestanding kernel context
- make math, allocation, randomness, and data loading work with kernel-safe paths

## Environment

- `nirOS` source tree
- `GNUmakefile` based build
- freestanding C / C++ kernel build rules
- kernel libc support files and headers
- TarFS-backed file access for dataset loading

## The Final State

The neural network sources now have kernel-oriented build and runtime shims in place instead of depending on ordinary userspace facilities.

## What Changed

- copied NN code into the `nirOS` directory
- added `CPPFILES` discovery and `.cpp` objects / deps in `GNUmakefile`
- added NN include path to `CPPFLAGS` in `GNUmakefile`
- added or kept a kernel-safe `.cpp` compile rule with `-fno-exceptions -fno-rtti` plus freestanding-safe extras
- added a `math.c` special compile path to avoid `-mgeneral-regs-only` for floating-point code
- added kernel libc support files
- added C++-safe `extern "C"` guards in libc headers
- updated the NN shim to use kernel libc math and allocation
- replaced STL RNG with `xorshift32` plus Box-Muller
- updated dense-layer init to use kernel-safe RNG and math with no `<random>` or `<cmath>`
- rewrote the CSV loader to use TarFS plus raw buffer parsing with no `std::string`, `ifstream`, or STL
- added `tarfs_open` / `tarfs_read` API and used it in the CSV path

## Verification

Current proof captured in this note is structural only:

- the build system changes are listed
- kernel-safe math, allocation, RNG, and file access replacements were added
- this note does not yet include a successful build log, boot log, or runtime inference output

## Failure Modes Worth Caring About

- userspace or STL dependencies getting reintroduced into freestanding kernel code
- compile flags that silently break floating-point or C++ integration in kernel space
- TarFS parsing behaviour diverging from the original userspace CSV loader
- allocator or math shim behaviour differing enough to invalidate model assumptions
