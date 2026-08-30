# .github

Shared defaults for this organisation.

`default.json` is the Renovate preset every repository here extends with
`{"extends": ["github>go-tpm2/.github"]}`.

Minor and patch Go dependency updates merge themselves once the tests pass.
**The Go toolchain does not.** `go1.27.0` miscompiles on loong64
([golang/go#81000](https://github.com/golang/go/issues/81000), bisected on real
hardware to `3fdac6780b`), and a repository without a loong64 lane sees nothing
wrong with the bump. A library that breaks is one library; a toolchain that
breaks is every architecture at once.
