# How this fork tracks upstream njs

This repository is a fork of [nginx/njs](https://github.com/nginx/njs). It
carries no source changes. Its whole delta over an upstream release is four
files under `.github/workflows/`, and it exists to produce the release tarball
FreeUnit builds against.

## What the fork changes, and why

| file | state | why |
| --- | --- | --- |
| `build-and-release.yml` | added | builds a release tarball on every tag push; upstream has nothing like it |
| `check-pr.yml` | trimmed | the removed jobs build QuickJS; FreeUnit builds `--no-quickjs` |
| `ci.yml` | deleted | upstream's own CI, not run here |
| `f5_cla.yml` | deleted | F5's CLA bot, which does not apply to this fork |

Check that list before vendoring: `git diff --stat <upstream tag> HEAD` must
show those four files and nothing else. If it shows a fifth, either upstream
moved something or a local change crept in, and that has to be explained before
the release is cut.

## Vendoring a new upstream release

The tag never points at an upstream commit. It points at the fork's merge
commit, which is what `git archive` then packages. Both previous releases were
made this way: 0.9.8 through PR #2, 1.0.0 through PR #5.

    git fetch base --tags                       # base = nginx/njs
    git worktree add -b vendor-njs-X.Y.Z ../njs-X.Y.Z origin/master
    cd ../njs-X.Y.Z

    git read-tree -u --reset X.Y.Z              # upstream's tree, exactly
    git checkout origin/master -- \
        .github/workflows/build-and-release.yml \
        .github/workflows/check-pr.yml          # keep the fork's two
    git rm -f .github/workflows/ci.yml .github/workflows/f5_cla.yml

    git diff --cached --stat X.Y.Z              # must be the four files above
    git commit -m "Vendor njs X.Y.Z"

Open a pull request, merge it, and tag the **merge commit**:

    git tag X.Y.Z <merge commit>
    git push origin X.Y.Z

Pushing the tag runs `build-and-release.yml`, which builds njs and attaches
`njs-X.Y.Z.tar.gz` and `.zip` to a GitHub release.

## The tarball FreeUnit consumes

FreeUnit's `pkg/contrib/src/njs/` pins a version and a SHA512, and downloads
from `https://packages.freeunit.org/njs/njs-$(NJS_VERSION).tar.gz`. That file
is this fork's release asset, mirrored.

**It is not GitHub's tag archive, and the two never match.** The release
workflow runs

    git archive --format=tar.gz --prefix=njs-<tag>/ HEAD

over the fork's tree, so the bytes differ from
`github.com/nginx/njs/archive/refs/tags/<tag>.tar.gz` in both content and
prefix. Measured on 1.0.0: the mirrored tarball is 1,007,848 bytes with SHA512
`fde328f7…`, which is what `pkg/contrib/src/njs/SHA512SUMS` records, while
GitHub's archive is 1,011,616 bytes and hashes to `11ed2e1b…`. A digest taken
from the wrong one passes review and fails the build.

The digest can be computed before the release exists, because `git archive` is
reproducible:

    git archive --format=tar.gz --prefix=njs-X.Y.Z/ <merge commit> | sha512sum

That command reproduces 1.0.0's recorded digest exactly.

## Also needed on a FreeUnit bump

FreeUnit's CI checks this fork out by version string:
`.github/workflows/build-test.yml` reads `NJS_VERSION` from
`pkg/contrib/src/njs/version` and clones `freeunitorg/njs` at `ref:` that
value. So the tag has to exist here before the version pin moves there, or
every njs-using leg fails on the checkout.

Order of operations, all four steps:

1. vendor and merge here, tag the merge commit, push the tag;
2. mirror the release asset to `packages.freeunit.org`;
3. compute the SHA512 from that asset;
4. bump `version` and `SHA512SUMS` in FreeUnit.
