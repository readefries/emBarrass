# Security audit

**Review date:** 2026-10-24  
**Scope:** `.github/workflows/*`, `.goreleaser.yaml`, `Makefile`, `cmd/**`, `pkg/**`, `go.mod`, and `go.sum`.

This is a static review of the checked-out source, not a guarantee that the
repository, its dependencies, or future releases are free of vulnerabilities.

## Findings

### High — GoReleaser was installed before its signature verifier

**Status: Remediated in this change.**

The release workflow originally installed GoReleaser before installing cosign,
and used the mutable `~> v2` version range. The GoReleaser action can verify its
downloaded tool signature when cosign is available, so the original order
allowed the tool to execute without that verification. GoReleaser runs in the
release job with write-capable GitHub release credentials and can also receive
the Homebrew tap token when publishing is enabled. A compromised tool release
could therefore expose those credentials or publish altered artifacts; the
later checksum-signature verification would not undo that exposure.

**References:** [release workflow](../.github/workflows/release.yml#L50-L58)
installs cosign before GoReleaser and pins GoReleaser to `v2.19.0`.

**Remediation:** Install cosign before invoking the GoReleaser action and use
an exact GoReleaser version. Keep the subsequent checksum-signature check as
an independent verification of release outputs. Both changes are applied.

## Areas reviewed; no additional high-confidence findings

- **Other workflow controls:** Actions are pinned to commit SHAs and the build
  and lint workflows use read-oriented permissions ([build/test](../.github/workflows/build_test.yml#L8-L24),
  [lint](../.github/workflows/golangci-lint.yml#L8-L27)). The release workflow
  uses a separate repository-scoped token for Homebrew publishing
  ([release](../.github/workflows/release.yml#L27-L38)).
- **Release artifacts:** GoReleaser creates a checksum file and signs it with
  cosign ([configuration](../.goreleaser.yaml#L48-L61)); the workflow verifies
  that signature after release ([release](../.github/workflows/release.yml#L71-L76)).
- **Build/install commands:** Makefile targets invoke fixed Go commands; install
  paths are quoted ([Makefile](../Makefile#L6-L17)). No shell-injection issue
  was identified.
- **Application execution and repair:** Child commands are executed with
  argument-based process APIs rather than shell interpolation
  ([runner](../pkg/runner/runner.go#L146-L169)). Repair operations constrain
  targets and validate ownership and paths before privileged changes
  ([Linux repair](../pkg/repair/linux.go#L118), [store](../pkg/repair/store.go#L260)).
- **Network and credentials:** The local proxy uses a cryptographically random
  token ([runner](../pkg/runner/runner.go#L329-L335)); the public `gh` probe
  overrides token variables ([command](../pkg/runner/command.go#L79-L102)).
  No additional exploitable download, forwarding, or credential exposure was
  identified.
- **Dependencies:** `go.mod` specifies fixed module versions and `go.sum`
  records checksums ([go.mod](../go.mod#L5-L37), [go.sum](../go.sum#L1-L127)).
  A dependency advisory/database scan was not performed, so this is not a
  claim that every dependency is vulnerability-free.

## Verdict

**Safe to check out and use with the following caveats:** the identified
release-workflow issue is remediated in this change; no other high-confidence
vulnerability was identified in the reviewed source. This static review does
not verify repository/environment settings, secret configuration, or a
dependency advisory database. The `batch` command intentionally executes
installed programs found in `PATH`; use it only when those programs and their
configured environments are trusted. The `fix` command changes supported
system trust stores, so review its preview and recovery information before
applying it.

The existing Go tests were run. Most packages passed, but several `pkg/repair`
tests failed because their temporary fixture directories were rejected as
unsafe native store directories; this audit did not modify those tests or
repair behavior.
