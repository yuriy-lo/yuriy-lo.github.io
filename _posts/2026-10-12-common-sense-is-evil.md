---
layout: post
title: "Common sense is evil"
date: 2026-10-12 00:00:00 +0000
---

Sometimes, when I brainstorm a task, an AI agent tries really hard to talk me out of my ideas.

For example, it says Bazel is for huge projects, that it is too complicated, and that CMake
should be the default. But if I choose Bazel for a tiny project, I only need this to start
writing unit tests:

```bazel
# MODULE.bazel
bazel_dep(name = "rules_cc", version = "0.2.26")
bazel_dep(name = "googletest", version = "1.18.0")

# BUILD
cc_test(
    name = "my_test",
    srcs = ["my_test.cc"],
    deps = [
        "@googletest//:gtest_main",
    ],
)
```

One command will download Bazel, fetch dependencies, locate the compiler, build everything,
and run the tests:
```bash
bazelisk test //...
```
The build and test results are cached, so the next run is very fast. The xUnit report
for my QA dashboard is already there. If I need integration tests in Python,
I just add a `py_test` target.

If it looks like a duck, swims like a duck, and quacks like a duck, it probably is a duck.
If something looks simple and easy to use, it is probably a good choice. But AI relies
on common sense which could be just wrong or out of date.

TL;DR: Don’t trust AI agents blindly. Give your ideas a chance.
