# Missing unit-test dependencies — Salt 3008.5 on `ycedres-proxy2`

Date: 2026-11-26
Branch: `openSUSE/draft/3008.5`
Python: 3.13.18 (`python313` package flavor)
Host: `ycedres-proxy2.mgr.suse.de` (KVM guest, 3.8 GiB RAM, no swap)

This document lists the tests that still fail **only because the installed
packages or the test environment do not provide required files/modules**.
They are not failures of the Salt code changes in this branch.

## Result summary

| Run | Tests | Errors | Failures | Skipped |
|---|---:|---:|---:|---:|
| Before the fixes (`junit-report-unit.xml.before-loader-fix`, 2026-11-25) | 14,448 | 2,421 | 310 | 1,969 |
| After the fixes (chunked run, 2026-11-26) | 14,355 | 42 | 44 | 2,178 |

The 86 remaining problems are all accounted for by the missing items below:

| Category | Tests |
|---|---:|
| `python313-salt-testsuite` does not ship Salt source-tree files | 82 |
| `python313-salt` does not ship `salt/_pkg.txt` | 2 |
| Missing Python module `etcd3` (`etcd3-py`) | 1 |
| Missing Python module `opentelemetry.exporter.prometheus` | 1 |
| **Total** | **86** |

Validation: the 82 source-tree-artifact tests were re-run against a throwaway
copy of the testsuite with `pkg/`, `tools/`, `changelog/` and
`salt/_pkg.txt` added, and **84/84 affected tests passed** (the 2 remaining
are the optional-module tests listed below).

## A. Files missing from `python313-salt-testsuite` (82 tests)

The tests derive the repository root from
`pathlib.Path(__file__).resolve().parents[N]`. In an installed testsuite this
resolves to `/usr/lib/python3.13/site-packages/salt-testsuite`, which does not
contain the source-tree `pkg/`, `tools/`, or `changelog/` directories.
`rpm -ql python313-salt-testsuite` confirms none of these paths are owned by
the package.

| Missing path(s) | Affected test files | Tests |
|---|---|---:|
| `tools/pkg/salt_build_backend.py` | `tests/pytests/unit/test_salt_build_backend.py` | 14 |
| `pkg/rpm/salt.spec` | `tests/pytests/unit/pkg/test_rpm_minion_scriptlets.py` (9), `test_master_scriptlets.py` (2), `test_minion_scriptlets.py` (2), `tests/pytests/unit/test_pkg_scripts.py` (2) | 15 |
| `pkg/debian/salt-{api,common,master,minion,syndic}.preinst` | `tests/pytests/unit/test_pkg_scripts.py` (26), `tests/pytests/unit/pkg/test_master_scriptlets.py` (3), `test_minion_scriptlets.py` (2), `tests/pytests/unit/pkg/debian/test_preinst_scripts.py` (1) | 32 |
| `pkg/debian/salt-{api,cloud,master,minion,syndic}.postinst` | `tests/pytests/unit/pkg/test_debian_postinst.py` | 5 |
| `changelog/.template.jinja` | `tests/pytests/unit/changelog/test_template.py` | 5 |
| `tools/audit_doc_links.py` | `tests/pytests/unit/doc/test_link_audit.py` | 7 |
| `pkg/common/onedir/_salt_onedir_extras.py` | `tests/pytests/unit/test_salt_onedir_extras.py` | 3 |
| `pkg/common/salt-minion.service` | `tests/pytests/unit/test_pkg_systemd_units.py` | 1 |
| **Total** | | **82** |

### Remediation

Ship these source-tree paths in `python313-salt-testsuite` (preserving their
relative layout under the testsuite root), or add a supported override for the
repository root in the tests and point it at a Salt source checkout.

## B. `salt/_pkg.txt` missing from `python313-salt` (2 tests)

`salt/utils/package.py::pkg_type()` reads `salt/_pkg.txt` to report the package
type. The file exists in the source tree (`salt/_pkg.txt`, content `pip`) but is
not installed by the RPM:

```text
$ rpm -ql python313-salt | grep _pkg.txt
# (no output)
$ ls /usr/lib/python3.13/site-packages/salt/_pkg.txt
ls: cannot access '.../salt/_pkg.txt': No such file or directory
```

Because it is absent, `salt.grains.package.package()["package"]` returns `None`
instead of `pip`, failing:

- `tests/pytests/unit/grains/test_package.py::test_grain_package_type` (1)
- `tests/pytests/unit/utils/test_package.py` (1)

### Remediation

Install `salt/_pkg.txt` as part of `python313-salt` (the source file contains
`pip`, which is what the unit test expects for a non-onedir build).

## C. Missing Python modules (2 tests)

| Module | Required by | Test |
|---|---|---|
| `etcd3` (`etcd3-py==0.1.6` in `requirements/static/ci/common.txt`) | `tests/pytests/unit/modules/test_etcd_mod.py::test_auth` | 1 |
| `opentelemetry.exporter.prometheus` (`opentelemetry-exporter-prometheus>=0.62b1` in `requirements/base.txt` for Python >= 3.10) | `tests/pytests/unit/utils/test_metrics.py::test_prometheus_exporter_binds_listener` | 1 |

Neither `python313-etcd3` nor `python313-opentelemetry-exporter-prometheus` is
installed in the test environment (`rpm -q` reports both as not installed).

### Remediation

Install the distribution packages (or the equivalent pip packages) in the
environment used to run the test suite.

## How the after-fix results were produced

A single pytest process running the complete unit suite cannot finish on this
VM: it is OOM-killed while executing
`tests/pytests/unit/utils/test_minions_resources.py::test_check_minions_grain_target_10000_minions_100_resources_each`
(10,000 minions × 100 resources = 1,000,000 cache entries; the process reached
~2.95 GiB RSS with 3.8 GiB total and no swap).

The suite was therefore run in 30 bounded chunks (the stress test isolated in
its own chunk), each with its own JUnit XML, then merged:

- merged report: `/root/results_junit/junit-report-unit-chunked.xml`
  (also copied to `/root/results_junit/junit-report-unit.xml`; the previous
  report is preserved as `junit-report-unit.xml.before-loader-fix`)
- chunked runner: `/root/chunked_run.py`
- per-chunk logs/XML: `/root/results_junit/chunks-20261126-194526/`
- summary: `/root/results_junit/chunks-20261126-194526/summary.txt`

The stress test passed when run alone (21.8 s, rc=0). If a single-process run is
required, add swap/RAM; otherwise keep the chunked runner.

Note for direct pytest invocations: do **not** pass `-p no:cacheprovider` on
this environment. It makes `pytestsysstats` register `sysstats-processes` twice
and pytest aborts with `ValueError: Plugin already registered under a different
name`. `salt-test` itself does not pass that flag.
