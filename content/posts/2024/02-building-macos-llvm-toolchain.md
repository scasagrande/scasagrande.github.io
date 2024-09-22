+++ 
draft = false
date = 2024-09-22
title = "Building macOS llvm for toolchains_llvm"
description = ""
slug = "building-macos-llvm-package"
authors = []
tags = ["llvm", "toolchains_llvm"]
categories = ["bazel"]
externalLink = ""
series = []
+++

[toolchains_llvm](https://github.com/bazel-contrib/toolchains_llvm) is a great [Bazel](https://bazel.build/) ruleset for providing a hermetic llvm C / C++ (CC) toolchain.

By default, this ruleset will download [official llvm release packages](https://github.com/llvm/llvm-project/releases). This is a good starting point, but you may want to build your own. Some reasons include:

- Not wanting to rely on GitHub hosted artifacts
- Wanting to take ownership over your CC toolchain
- Downloading and extracting these large official releases is significantly impacting your CI startup time
- You want to use a version that doesn't have official releases for your platforms

By the end of this article you will be able to build your own macOS llvm package with libc++ for use with toolchains_llvm. In addition to native macOS builds, this toolchain will be able to cross-compile for both Linux-x86_64 and Linux-aarch64 [with an appropriate Linux sysroot package]({{< relref "01-sysroot-package-generation.md" >}}).

Link to source code: https://github.com/scasagrande/build_macos_llvm

{{< notice note >}}
This example is geared towards building a toolchain package with libc++ for macOS.

For Linux toolchain builds please see [David Zbarsky's static-clang repository](https://github.com/dzbarsky/static-clang) as well as my previous post on [building a Linux sysroot package for use with toolchains_llvm]({{< relref "01-sysroot-package-generation.md" >}}). David's repo can also build a toolchain for macOS, but it does not include libc++, leaving you reliant on your system copy.
{{< /notice >}}

## Prerequisites

You will need to install all the required tools that the LLVM project requires.

- Make sure you have [XCode](https://developer.apple.com/xcode/) installed and operational through the Apple App Store.
- Next, use [`brew`](https://brew.sh/) (or your other preferred method) to install the following:

```console
brew install bzip2 cmake coreutils git lz4 make ninja xz zlib zstd
```

## Build

Now we're ready to invoke the build. For this example, I am building version 17.0.6. This will be using a multi-stage build process, so expect it to take a while. This took approx 40min on my M3 Macbook pro.

The build will match your system architecture. If you need to generate packages for both x86_64 and aarch64/arm64, you will need to execute this build on both platforms.

```console
./build-macos-llvm.sh 17.0.6 ~/clang-17.0.6-aarch64-apple-darwin.tar.xz
```

For me, this resulted in an approximately 125MB package, down from 767MB for the official release. Could this be even smaller? Probably. If you see more files that can be deleted, feel free to submit a PR!

## Test

Before we upload this package to your team's or organization's file storage system (eg, [Artifactory](https://jfrog.com/artifactory/)) we want to test it to make sure it will work for your project.

Let's start by unpacking this archive to a temporary folder on your system. Here we will use `/tmp/llvm`. 

```console
mkdir /tmp/llvm
tar xf ~/clang-17.0.6-aarch64-apple-darwin.tar.xz -C /tmp/llvm
```

Copy this build file onto your unpacked dir:
https://github.com/bazel-contrib/toolchains_llvm/blob/v1.1.2/toolchain/BUILD.llvm_repo

```console
wget https://raw.githubusercontent.com/bazel-contrib/toolchains_llvm/refs/tags/v1.1.2/toolchain/BUILD.llvm_repo -O /tmp/llvm/BUILD.llvm_repo
```

Update your project to point to this unpacked directory:

`MODULE.bazel`

```bzl {linenos=inline,hl_lines="10-13"}
bazel_dep(name = "toolchains_llvm", version = "1.1.2", dev_dependency = True)

llvm = use_extension("@toolchains_llvm//toolchain/extensions:llvm.bzl", "llvm", dev_dependency = True)
llvm.toolchain(
    stdlib = {
        "linux-x86_64": "stdc++",
        "linux-aarch64": "stdc++",
    },
)
llvm.toolchain_root(
    name = "llvm_toolchain",
    path = "/tmp/llvm"
)
```

Now run your bazel build and test!

## Use with toolchains_llvm

Once everything has passed, and you're ready to share this toolchain with your colleagues, upload it to your file store platform and update your toolchains_llvm configuration like so:

`MODULE.bazel`

```bzl {linenos=inline,hl_lines="6-9 14-17"}
bazel_dep(name = "toolchains_llvm", version = "1.1.2", dev_dependency = True)

llvm = use_extension("@toolchains_llvm//toolchain/extensions:llvm.bzl", "llvm", dev_dependency = True)
llvm.toolchain(
    llvm_version = "17.0.6",
    sha256 = {
        "darwin-aarch64": "...",
        "darwin-x86_64": "...",
    },
    stdlib = {
        "linux-x86_64": "stdc++",
        "linux-aarch64": "stdc++",
    },
    urls = {
        "darwin-aarch64": ["..."],
        "darwin-x86_64": ["..."],
    },
)
```

# Feedback

Please leave your questions and comments [in this discussions thread on the repository](https://github.com/scasagrande/build_macos_llvm/discussions/).

I can also be reached on the [Bazel slack community](https://bazelbuild.slack.com) under my full name.
