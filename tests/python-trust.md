# JoinMarket certificate bootstrap

JoininBox recreates JoinMarket's source directory and venv during updates.
A post-install pip repair alone cannot protect the first pip/setuptools downloads
inside the newly created environment. The installer now applies a small local
backport after ref selection and the existing source-verification step, before
executing it. Normal pinned install/update paths retain signature verification;
explicit development/PR paths keep their existing trust policy unchanged. It checks
pip and application certifi again before declaring the update successful and on
the already-installed early-return path. The initial system-pip prerequisites use
the same checked, file-based CA bootstrap without upgrading Debian's pip.

The backport targets JoinMarket `v0.9.12` at
`77a426c504cbf6fa56ba0316dfd8da63a76d2bbf`. Its `install.sh` is identical to
archived master `c2dd2534afa59ddcb7f7d9a67e4b57cefcf9e4a4`. Unknown installer
variants fail if the exact bootstrap hunks no longer match; do not relax patch
fuzz to make an unreviewed version install. The helper lives in JoininBox because
[JoinMarket's repository is archived](https://github.com/JoinMarket-Org/joinmarket-clientserver)
and its [final release note](https://github.com/JoinMarket-Org/joinmarket-clientserver/blob/master/docs/release-notes/release-notes-0.9.12.md)
explicitly says no further updates are planned. There is no dependency on an
unpublished JoinMarket merge.

## Compatibility and policy boundary

The venv bootstrap requires Python >=3.10 for the published fixed pip>=26.2.2.
The project's package metadata previously permits Python 3.9, and older docs also
mention 3.8. This patch does not silently change package metadata or auto-upgrade
Python: it fails with a clear explanation on those interpreters. PyPI's complete
release metadata shows the last compatible pip releases are 25.0.1 for Python
3.8 and 26.0.1 for Python 3.9; both downloaded wheels retain affected eMudhra SSL
roots. A maintained pip backport or explicit interpreter upgrade is required.
Maintainers must accept that installer boundary before releasing this patch.

Update the OS CA package first. Operator PIP_* settings and CA overrides survive
the sudo boundary. Conflicting pip/certifi constraints remain fatal rather than
being ignored. The bootstrap rejects affected or malformed roots, TLS bypasses,
HTTP package sources, old pip with opt-in truststore, and unknown modern pip
capabilities. It requires a CA file, not directory-only bootstrap trust. Modern
pip is temporarily switched out of additive truststore only during bootstrap;
normal verified TLS remains in use afterward.

Postconditions inspect pip's vendored certifi, explicit CA files, OpenSSL's loaded
default roots, and application certifi. They are not a native Keychain audit or
an enumeration of lazy OpenSSL CA directories. This patch is scoped to the
JoinMarket updater, not every standalone image-builder or package updater.

## Reproduce from public sources

Run from this JoininBox checkout with Python 3.11 (or Python >=3.10), `git`,
`patch`, `bash`, `openssl`, and bats-core installed. Python must support `venv`
and verified HTTPS to PyPI; the bootstrap regression downloads wheels and binds
an ephemeral loopback HTTPS port. No unpublished JoinMarket patch is required.

```sh
# Use a new directory; on Hermes, TMPDIR points to its scratch workspace.
work=$(mktemp -d "${TMPDIR:-/tmp}/jib-python-trust.XXXXXX")
git clone --depth 1 --branch v0.9.12 \
  https://github.com/JoinMarket-Org/joinmarket-clientserver.git "$work/joinmarket"
test "$(git -C "$work/joinmarket" rev-parse 'v0.9.12^{commit}')" = \
  77a426c504cbf6fa56ba0316dfd8da63a76d2bbf
python3.11 tests/bootstrap_regression.py "$work/joinmarket" "$work/bootstrap"
python3.11 tests/test-python-trust-update.py "$work/joinmarket" "$work/owner"
bats tests/python-trust.bats tests/repository-contracts.bats \
  tests/install.joinmarket-args.bats tests/verify.git.bats
git -C "$work/joinmarket" status --porcelain  # must remain empty
```

Both Python harnesses check that the tag and HEAD resolve to the pinned public
commit and that `install.sh` matches its Git object. This is source-identity
checking, not an independent GPG release-signature verification. Neither writes
to the supplied checkout. Use separate scratch directories for the two harnesses;
the bootstrap harness requires an empty or nonexistent directory. Keys, venvs,
modified test CA bundles, wheel metadata, and `results.json` remain there.

### Coverage and boundaries

- `tests/bootstrap_regression.py`: **20 checks** using the actual owned
  `scripts/joinmarket-python-trust.py`, not a second production implementation.
  Genuine pip 23.3.2 and 26.2.1 upgrade to 26.2.2 over local HTTPS; controls cover
  wrong CA rejection despite an accepted-before injected vendor root, modern
  additive truststore, Requests CA precedence, stale pip/certifi constraints,
  TLS bypass rejection, and affected actual wheel bundles. Wheels are fetched
  from the exact URL in PyPI metadata and checked against its SHA-256.
  Only installer-function cases apply the owned 58-line backport with
  `--batch --forward --fuzz=0` to a scratch copy of public `install.sh`.
  Fresh/reused `venv_setup` and failure propagation run for real; the skip-build
  decision is stubbed, native builds and installer `main` are not run. The helper
  is copied unchanged under the installer's expected filename in test venv
  fixtures. One future-capability case explicitly mocks pip's advertised parser
  capability. Local TLS controls simulate trust paths, not an incident-host probe.
- `tests/test-python-trust-update.py`: **13 checks** execute the real owner
  installer function for install/update/testPR/commit and the real backport.
  OS/sudo, Git source retrieval/authentication, package installation, downstream
  installer execution, and postcondition execution are mocked. This checks
  ordering, policy forwarding, fatal failures, and unknown-source rejection;
  it does not prove a native owner update or real sudo environment preservation.
- The selected **17 Bats checks** cover ownership/order/postconditions, canonical
  repository syntax/config contracts, argument validation, and mocked Git/GPG
  verification behavior. They need neither `/home` nor a device. This selected
  set is not the entire Bats suite.

The self-contained harness adaptation passed 20 bootstrap checks on Python
3.11, 13 owner checks, and all 17 selected Bats checks. Earlier reference-harness
runs covered Python 3.10–3.13; those four versions were not all rerun for this
harness-only adaptation.

### Additional Debian retrieval evidence

The parent validation run independently exercised this owned helper on actual
Debian: a fresh venv with pip 23.0.1 upgraded to 26.2.2 and **18/18 real pip/TLS
controls passed**. Actual distro pip 23.0.1 also completed `safe_pip download`;
all **494 dpkg-owned package files checked remained unchanged**. These are
retrieval/bootstrap checks, not a full JoinMarket build or actual owner update.

The configured PiWheels extra index returned a wheel with ZIP timestamp
bytes different from PyPI's artifact; all **478 archive members were
byte-identical**. That is not whole-wheel PyPI hash identity. A separate retrieval
from the exact official PyPI metadata URL passed its published SHA-256 check.
Do not conflate those two retrieval results.

No full native JoinMarket build, Raspberry Pi deployment, actual owner update,
live incident-host probe, or SSH operation is claimed. Python 3.8/3.9 support
remains an explicit maintainer release gate, not hidden compatibility. Outer
RaspiBlitz sudo policy preservation and approved release/tag linkage are separate
integration gates; this owner test set does not certify them. This is a public
CA-incident remediation, not a claim of a new private JoininBox vulnerability.
Repository security-reporting policy still applies to genuinely new findings.
