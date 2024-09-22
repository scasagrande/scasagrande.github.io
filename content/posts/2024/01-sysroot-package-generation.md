+++ 
title = "Sysroot package generation for use with toolchains_llvm"
draft = false
date = 2024-03-23
lastmod = 2024-09-08
description = "How to generate a sysroot package for use with the Bazel ruleset [`toolchains_llvm`][toolchains_llvm] in order to utilize a hermetic llvm-clang C/C++ compiler toolchain."
slug = "sysroot-generation-toolchains-llvm"
aliases = ['/articles/sysroot-generation-toolchains-llvm']
authors = []
categories = ["bazel"]
externalLink = ""
series = []
tags = ["bazel", "sysroot", "toolchains_llvm"]
+++

[Bazel](https://bazel.build/) aims to provide a hermetic environment for your build & test operations ("actions") to run in. However by default Bazel does not provide a hermetic C / C++ (CC) toolchain. If you would like to learn more about this and why it is important, then I suggest you read [this article by Thulio Ferraz Assis at Aspect](https://blog.aspect.dev/hermetic-c-toolchain).

Alongside their article, [they developed a solution](https://github.com/f0rmiga/gcc-toolchain) to enable a hermetic GCC compiler and associated Linux sysroot package. Their solution is great if you want to use GCC. But if you instead want to use a hermetic `llvm-clang` compiler toolchain, then we are going to have to make some modifications.

By the end of this article you will be able to generate your own Linux sysroot package and combine it together with `llvm-clang` in order to provide a hermetic CC toolchain capable of cross-compilation.

Link to source code: https://github.com/scasagrande/toolchains_llvm_sysroot

Please leave your questions and comments [under the discussions tab on the repository](https://github.com/scasagrande/toolchains_llvm_sysroot/discussions/1).

{{< notice info >}}
2024-09-07: Updated to emphasize the `bzlmod` workflow over the legacy `WORKSPACE` one.
{{< /notice >}}

# toolchains_llvm ruleset

To start with, we are going to use the [`toolchains_llvm`][toolchains_llvm] ruleset in order to fetch and setup the actual toolchain. This ruleset allows you to [specify a Bazel target that contains your sysroot package files](https://github.com/bazel-contrib/toolchains_llvm?tab=readme-ov-file#sysroots).

Before we set it up, you'll need to choose if you're going to use `llvm-libc++` or if you are going to use `libstdc++`. The former is the default for the ruleset (when doing native compilation builds), it is the common libc++ implementation for macOS builds, and is included as part of the [standard llvm packages](https://github.com/llvm/llvm-project/releases). The latter, `libstdc++`, is more common for Linux builds. In both cases we will still require a separate sysroot package in order to provide a hermetic CC toolchain environment. Here I will be using `libstdc++` for our Linux target platform builds, and `libc++` for our macOS target platform builds.

Without a sysroot package, you might use this ruleset to define your CC toolchain as follows. This will enable a CC toolchain for Linux and macOS on both x86_64/amd64 and aarch64/arm64.

{{< notice example >}}
`llvm` version `15.0.2` was chosen for this example because [there exists release packages](https://github.com/llvm/llvm-project/releases/tag/llvmorg-15.0.2) for all four of our build platform configurations. Choose whatever versions suit your needs. However, I suggest that you ultimately build your own copies and host the packages yourselves.
{{< /notice >}}

`MODULE.bazel`

```bzl {linenos=inline}
bazel_dep(name = "toolchains_llvm", version = "1.1.2", dev_dependency = True)

# Configure and register the toolchain.
llvm = use_extension("@toolchains_llvm//toolchain/extensions:llvm.bzl", "llvm", dev_dependency = True)
llvm.toolchain(
    name = "llvm_toolchain",
    llvm_version = "15.0.2",
    stdlib = {
        "linux-x86_64": "stdc++",
        "linux-aarch64": "stdc++",
    }
)

use_repo(llvm, "llvm_toolchain")

register_toolchains("@llvm_toolchain//:all", dev_dependency = True)
```

or for projects using the legacy `WORKSPACE` system:

`WORKSPACE.bazel`

```bzl {linenos=inline}
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "toolchains_llvm",
    sha256 = "e91c4361f99011a54814e1afbe5c436e0d329871146a3cd58c23a2b4afb50737",
    strip_prefix = "toolchains_llvm-1.0.0",
    url = "https://github.com/bazel-contrib/toolchains_llvm/releases/download/1.0.0/toolchains_llvm-1.0.0.tar.gz",
)

load("@toolchains_llvm//toolchain:deps.bzl", "bazel_toolchain_dependencies")

bazel_toolchain_dependencies()

load("@toolchains_llvm//toolchain:rules.bzl", "llvm_toolchain")

llvm_toolchain(
    name = "llvm_toolchain",
    llvm_version = "15.0.2",
    stdlib = {
        "linux-x86_64": "stdc++",
        "linux-aarch64": "stdc++",
    }
)

load("@llvm_toolchain//:toolchains.bzl", "llvm_register_toolchains")

llvm_register_toolchains()
```

# Sysroot package generation

Link to source code: https://github.com/scasagrande/toolchains_llvm_sysroot

Now we need to provide a sysroot package in order to ensure that the build is using a consistent version of our system libraries. As a bonus, we will generate a matching package for both `x86_64` and `aarch64` CPU architectures.

To programmatically construct these packages, I started with the [sysroot generation code in the gcc-toolchain repo](https://github.com/f0rmiga/gcc-toolchain/tree/main/sysroot). Although it's designed to build a sysroot package for that specific toolchain ruleset, we can modify it to meet the needs of [`toolchains_llvm`][toolchains_llvm]. It's also already set up to enable the generation of sysroot packages for both `x86_64` and `aarch64`.

With this starting point, I made the following changes:

- Updated the following system libraries to target a Red Hat Enterprise Linux (RHEL) 8 environment
    - Kernel 4.18
    - glibc 2.28
    - libstdc++ 10.3
- Updated toolchains used to versions that support building those above libraries
- Removed a bunch of the files that we aren't going to need, such as binaries
- Moved the files around, renamed some folders, and placed everything under a standard `/usr` layout

With these changes, you'll have the basics that you need to provide a hermetic CC toolchain to Bazel.

In my case, I also decided to expand the sysroot package with `openssl` and `cyrus-sasl`, copied from a [RHEL-UBI 8 container image](https://catalog.redhat.com/software/containers/ubi8/5c647760bed8bd28d0e38f9f). Openssl is a very common build dependency and is useful to have in your sysroot. For those of us targeting a RHEL 8 production environment, it can be very helpful to have this specific copy of openssl for FIPS compliance. But we're not going to get into that today, and instead just focus on the fact that we are including these optional libraries in our sysroot package.

{{< notice note >}}
If your deliverable is an OCI image, you should not be shipping these libraries into your final artifacts. These are just to ensure that they are hermetically available for any Bazel build and test operations, and that we are dynamically linking against these production-like copies. Your base image should already have the exact versions that you need. You may even have regulation compliance requirements where you must use the copies already present in your base image. If you think that this may apply to you, be sure to consult the rest of your team!
{{< /notice >}}

So now lets go ahead and build it. These commands will build our two sysroot packages with these extra ssl libraries bundled in, and output them into our current directory.

```console
$ ./build.sh x86_64 . ssl
$ ./build.sh aarch64 . ssl
```

If you want to omit these ssl related packages, then replace `ssl` with `base`.

After some processing time, you will be left with 2 `tar.xz` files. If you have been following along, the `x86_64` file should be approximately 55MB, and the `aarch64` file approximately 35MB.

# Using the sysroot package

First we need to make the `tar.xz` files available to Bazel. You'll need to make your sysroot package available somewhere for your team to download, such as in [Artifactory][Artifactory]. How you do that is up to you. For testing, you can put in a fake URL and use the `--override_repository=` CLI argument built into Bazel.

`MODULE.bazel`

```bzl {linenos=inline}
http_archive = use_repo_rule("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "sysroot_linux_x86_64_2_28",
    url = "https://example.com/sysroot-x86_64-ssl.tar.xz",
    sha256 = "ABCD1234",
    build_file = "//external:BUILD.sysroot.bazel"
)

http_archive(
    name = "sysroot_linux_aarch64_2_28",
    url = "https://example.com/sysroot-aarch64-ssl.tar.xz",
    sha256 = "1234ABCD",
    build_file = "//external:BUILD.sysroot.bazel"
)
```

`WORKSPACE.bazel`

```bzl {linenos=inline}
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "sysroot_linux_x86_64_2_28",
    url = "https://example.com/sysroot-x86_64-ssl.tar.xz",
    sha256 = "ABCD1234",
    build_file = "//external:BUILD.sysroot.bazel"
)

http_archive(
    name = "sysroot_linux_aarch64_2_28",
    url = "https://example.com/sysroot-aarch64-ssl.tar.xz",
    sha256 = "1234ABCD",
    build_file = "//external:BUILD.sysroot.bazel"
)
```

`external/BUILD.sysroot.bazel`

```bzl {linenos=inline}
filegroup(
    name = "sysroot",
    srcs = glob(["**"]),
    visibility = ["//visibility:public"]
)
```


And then we just need to update our toolchain definition to use these new sysroot package when building and targeting linux `x86_64` and `aarch64`.

`MODULE.bazel`

```bzl {linenos=inline,hl_lines="11-20"}
bazel_dep(name = "toolchains_llvm", version = "1.1.2", dev_dependency = True)

llvm = use_extension("@toolchains_llvm//toolchain/extensions:llvm.bzl", "llvm", dev_dependency = True)
llvm.toolchain(
    llvm_version = "15.0.2",
    stdlib = {
        "linux-x86_64": "stdc++",
        "linux-aarch64": "stdc++",
    },
)
llvm.sysroot(
    name = "llvm_toolchain",
    label = "@@sysroot_linux_x86_64_2_28//:sysroot",
    targets = ["linux-x86_64"],
)
llvm.sysroot(
    name = "llvm_toolchain",
    label = "@@sysroot_linux_aarch64_2_28//:sysroot",
    targets = ["linux-aarch64"],
)
use_repo(llvm, "llvm_toolchain")

register_toolchains("@llvm_toolchain//:all", dev_dependency = True)
```

`WORKSPACE.bazel`

```bzl {linenos=inline,hl_lines="8-11"}
llvm_toolchain(
    name = "llvm_toolchain",
    llvm_version = "15.0.2",
    stdlib = {
        "linux-x86_64": "stdc++",
        "linux-aarch64": "stdc++",
    }
    sysroot = {
        "linux-x86_64": "@sysroot_linux_x86_64_2_28//:sysroot",
        "linux-aarch64": "@sysroot_linux_aarch64_2_28//:sysroot",
    }
)
```

Optionally, we can test our build without having uploaded the `tar.xz`. Start with unpacking the tar file into a folder on your filesystem. Next make a `BUILD.bazel` file located at the root of this extracted folder (that is, beside the extracted `usr/` folder) with the contents from your `BUILD.sysroot.bazel` file.

Finally, to execute your test, run the following if you are building on a Linux x86_64 machine:

```console
$ bazel build --override_repository=sysroot_linux_x86_64_2_28=/path/to/sysroot-x86_64-ssl //...
```

# Result

In the end you should now be able to build and test targeting linux `x86_64` or `aarch64`, both with native builds or cross compilation, including from macOS!

But does that mean that you now have a fully hermetic CC build? Well, that is going to depend on the rest of your project. For example, if you have rust dependencies fetched via [`rules_rust`][rules_rust] , some of them may include `build.rs` files that perform their own CC compilation that breaks out of the Bazel sandbox, such as [`rdkafka-sys`][rdkafka-sys], [`openssl-sys`][openssl-sys], or [`sasl2-sys`][sasl2-sys]. In the future I will cover how best to deal with these cases, while also using our new sysroot packages.

# Feedback

Please leave your questions and comments [in this discussions thread on the repository](https://github.com/scasagrande/toolchains_llvm_sysroot/discussions/1).

I can also be reached on the [Bazel slack community](https://bazelbuild.slack.com) under my full name.

[Artifactory]: https://jfrog.com/artifactory/
[bzlmod]: https://bazel.build/external/overview#bzlmod
[openssl-sys]: https://github.com/sfackler/rust-openssl/blob/master/openssl-sys/build/main.rs
[rdkafka-sys]: https://github.com/onsails/rust-rdkafka/blob/master/rdkafka-sys/build.rs
[rules_rust]: https://github.com/bazelbuild/rules_rust
[sasl2-sys]: https://github.com/MaterializeInc/rust-sasl/blob/master/sasl2-sys/build.rs
[toolchains_llvm]: https://github.com/bazel-contrib/toolchains_llvm
