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

## Verification

- `bats tests/python-trust.bats`: automatic static ownership/order/postcondition
  contracts; does not need `/home` or a device.
- `python3 tests/test-python-trust-update.py JOINMARKET_CHECKOUT SCRATCH`: executes
  the real owner installer function for install/update/testPR/commit with the real
  backport and mocked OS/package/native-build operations. Verifies installer and
  postcondition failures stop success reporting, operator policy is forwarded,
  and unknown source variants reject the backport.
- The companion local JoinMarket source patch includes
  `test/python-trust/bootstrap_regression.py`: actual pip 23.3.2 and 26.2.1, local HTTPS
  positives/negatives, additive-trust accepted-before baseline, actual pip and
  certifi upgrades, stale constraints, and executed fresh/reused venv functions.
  Wheels are checked against PyPI hashes. Its one unknown-future-capability test
  is explicitly mocked; its other package/TLS operations are real.

No full JoinMarket native build, Raspberry Pi run, Linux container run, live
incident-host probe, node update, or SSH operation was performed. The local Docker
daemon was unavailable. This is a public CA-incident remediation, not a claim of
a new private JoininBox vulnerability. Repository security-reporting policy still
applies to any genuinely new findings.
