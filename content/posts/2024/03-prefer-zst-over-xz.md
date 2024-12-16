+++ 
draft = false
date = 2024-12-16
title = "Prefer ZST compression over XZ for use with Bazel"
description = ""
slug = "prefer-zst-over-xz"
authors = []
tags = ["zst", "xz", "compression"]
categories = ["bazel"]
externalLink = ""
series = []
+++

Earlier this year, I was profiling our Bazel builds looking for any easy wins to speed up CI. One bottleneck I found was the decompression of fetched archives that were compressed with xz. 

Before starting this work I already knew that decompressing our [pre-built llvm toolchain]({{< relref "02-building-macos-llvm-toolchain.md" >}}) and [sysroot package]({{< relref "01-sysroot-package-generation.md" >}}) took a non-trivial amount of time. But with the full profiling data in hand I was able to see exactly how much time we were spending in CI on fetch vs decompression, how many tasks were waiting, and if there were any other decompression tasks taking a non-trivial amount of time.

After doing some research online, I decided to experiment with ZST compression instead of XZ. I already knew that XZ had poor decompression speeds, especially at higher compression levels, so it was worth taking a look.

## Repacking llvm toolchain with ZST compression

Using my Macbook Pro equipped with an M3-Pro, I started with fetching my macos-arm llvm toolchain, decompressed it, and then switched it to ZST as follows:

```bash
# Install pre-reqs, using gnu-tar to prevent any warnings caused by gnu-tar 
# decompressing archives built with macOS bundled bsd-tar. This is optional, 
# you can use `tar` if you like. 
brew install gnu-tar zstd

# Time decompression for XZ archive
time gtar xf clang-17.0.6-aarch64-apple-darwin.tar.xz

# Create new ZST compressed archive with maximum compression
# This will take a while!
gtar cf clang-17.0.6-aarch64-apple-darwin.tar bin include lib
zstd --ultra -T0 -22 clang-17.0.6-aarch64-apple-darwin.tar 

# Remove old folders
rm -r bin include lib

# Time decompression of ZST archive
time gtar xf clang-17.0.6-aarch64-apple-darwin.tar.zst 

# Check filesize of both archives
du -h clang-17.0.6-aarch64-apple-darwin.tar.{zst,xz} 
```

The decompression speeds were much faster than I had hoped. The XZ package took 5.45s while the ZST package was only 0.66s!

Now the part that surprised me was the filesize. I expected that the ZST archives would be larger. However, this resulted in a filesize of 122MB for XZ and 101MB for ZST. So smaller and faster to decompress! A win-win for my purposes here.

## Results of re-bundling all 4 platform toolchains

With these great local results, I went ahead and re-bundled all 4 llvm packages, with the following results. All data was taken as a single shot on my Macbook Pro with M3-Pro processor. You are welcome to complain about significant figures.

| Platform  | File size XZ (MB) | File size ZST (MB) | Decompression time XZ (s) | Decompression time ZST (s) |
|-----------|-------------------|--------------------|---------------------------|----------------------------|
| Linux x86 | 98                | 66                 | 4.51                      | 0.42                       |
| Linux arm | 87                | 61                 | 3.80                      | 0.40                       |
| macOS x86 | 117               | 85                 | 5.29                      | 0.61                       |
| macOS arm | 122               | 101                | 5.45                      | 0.66                       |

All 4 packages are smaller and have 9-10x faster decompression speeds!

A couple of seconds might not seem like much, but:
- My CI workers are slower than my Macbook
- This happens on every single CI job because I am restricted to ephemeral workers
- Setting up the C++ toolchain is a blocker for most of the build

After profiling this in CI, I was able to verify an improvement in the extraction time from 18s down to 2s, which is inline with the speedup that I saw locally.

Coupling this with migrating the sysroot packages to ZST compression, together they helped save approximately 30sec at the start of every single build.

# Feedback

I can be reached on the [Bazel slack community](https://bazelbuild.slack.com) under my full name.
