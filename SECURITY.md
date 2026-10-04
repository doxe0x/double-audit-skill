# Security policy

## What double-audit can do

The skill is instructions and reference text (`SKILL.md`, `references/audit-playbook.md`). It declares no tools and fetches nothing at runtime. During an audit it directs Claude to run local commands; the trust gate keeps a third-party repo's tests, builds, and installs inside a sandbox the user approves, and secret findings are reported masked.

`scripts/build_bundle.py` is a maintainer tool for packaging. The skill never calls it.

## Verifying what you install

- `double-audit.skill` is rebuilt deterministically from the sources. CI fails when the bundle, `SHA256SUMS`, and the sources drift apart, or when zero-width, bidi, Unicode tag characters, or HTML comments appear in the bundled files.
- Check a downloaded bundle against `SHA256SUMS` from the same commit: `shasum -a 256 double-audit.skill` (macOS) or `sha256sum double-audit.skill` (Linux).
- Pin a clone to the commit you reviewed and read the diff before updating (`git log -p HEAD..origin/main`).

## Reporting a problem

Use GitHub's private vulnerability reporting on this repository if it is enabled. Otherwise open an issue titled "security contact" without exploit details, and a private channel will be arranged. Include the affected file and steps to reproduce.
