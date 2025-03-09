+++ 
draft = false
date = 2025-03-09
title = "Introducing bazel-examples"
description = "Introducing scasagrande/bazel-examples, a git repository hosting a variety of Bazel related examples for public reference."
slug = "introducing-bazel-examples"
authors = []
tags = ["examples"]
categories = ["bazel"]
externalLink = ""
series = []
+++

In an effort to provide a consolidated location for all my current and future bazel-related examples, I have created a new centralized location for that on GitHub: [scasagrande/bazel-examples](https://github.com/scasagrande/bazel-examples)

The intention here is to provide something similar to the [Aspect Build bazel-examples repository](https://github.com/aspect-build/bazel-examples), where others can come and find reference implementations for various common tasks. Some examples might be fully runnable bazel modules, while some might be bazel-adjacent code. As I include more I will aim to make a short corresponding post here.

As of today, there are two included examples:

## [rustls-openssl-fips](https://github.com/scasagrande/bazel-examples/tree/main/rustls-openssl-fips)

The goal for this example was to answer the common question of "how do I get openssl-sys to build under rules_rust?", but to make a little more advanced by using this as a way to create an application that can use openssl with FIPS mode enabled. 

This example shows how to:

- Build a basic rustls application
- Use openssl as our crypto provider
- Bundle our application into an OCI image
- Start our application with FIPS mode enabled

To get there, we use the following:

- [rustls-openssl](https://github.com/tofay/rustls-openssl) to use openssl as our crypto provider, instead of [aws-ls-rs](https://github.com/aws/aws-lc-rs). [The latter can be FIPS-140-3 compliant](https://github.com/aws/aws-lc/blob/main/crypto/fipsmodule/FIPS.md), but in this case we explicitly want to use openssl.
- rules_rust crate annotations in order to correctly build crate [openssl-sys](https://github.com/sfackler/rust-openssl) under Bazel. This can be useful for anyone looking to depend on the openssl rust crate. See [third_party/BUILD.bazel](./third_party/BUILD.bazel) for details.
- Add our binary to a [Red Hat Universal Base Image 8 Minimal](https://catalog.redhat.com/software/containers/ubi8/ubi-minimal) image, which will contain a FIPS-140-3 compliant copy of openssl.
- Uses a pre-built copy of openssl 1.1.1 that I provide at [scasagrande/openssl-prebuilt](https://github.com/scasagrande/openssl-prebuilt). 
- container_structure_test to run our image and make sure that all tests report back as passing from inside the container.

## [rdkafka-sys](https://github.com/scasagrande/bazel-examples/tree/main/rdkafka-sys)

Building the rdkafka-sys crate is a little commonly requested by the community in the Bazel slack, but can be tricky to implement. So I have provided an example on how to:

- Set up a basic rust project using crate [rdkafka](https://crates.io/crates/rdkafka)
- Define a bazel-native build of [librdkafka](https://github.com/confluentinc/librdkafka)
- Use crate annotations to correctly build crate [rdkafka-sys](https://crates.io/crates/rdkafka-sys)

With that all defined, we can then run tests against a running kafka + zookeeper service.
