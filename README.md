# eMudhra CA Compromise: Securing Python Virtual Environments on Ubuntu and RHEL

Oct 21, 2026 - Version 1.0

## Why this matters

Python virtual environments on Ubuntu, Red Hat and other Linux variants must be updated to stop trusting the compromised eMudhra certificates. That means patching the OS trust store and upgrading `pip` and `certifi` inside every virtual environment.

On October 17, 2026, Mozilla [received a report](https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/) that the private keys of eight eMudhra intermediate certificate authorities were compromised. An attacker holding them can issue a trusted certificate for any website, including Qualys API endpoints, and intercept TLS traffic. Python applications trust two sources of root certificates: the operating system store and the `certifi` bundle inside each virtual environment (pip carries its own copy too). Both must be fixed.

Four steps:

1. Follow your Corporate Patch Instructions for Red Hat or Ubuntu.  Ensure you are at Python 3.10 or greater before proceeding.
2. Check each Python virtual environment and upgrade pip and certifi.
3. QualysETL users: optionally reinstall QualysETL.
4. Validate with `pip list` that `pip` and `certifi` are at the latest versions.

## Step 1: Infrastructure team patches the OS

Ask your infrastructure team to apply the operating system vendor's `ca-certificates` update, following the vendor's own instructions, **before** you reinstall QualysETL. The QualysETL installer starts from the operating system's Python and pip, so the OS trust store must be fixed first.

| Release | What to apply | Vendor instructions |
| --- | --- | --- |
| Ubuntu 22.04 LTS | `ca-certificates` 20260816ubuntu0.22.04.1 | [USN-8944-1](https://ubuntu.com/security/notices/USN-8944-1) |
| Ubuntu 24.04 LTS | `ca-certificates` 20260816ubuntu0.24.04.1 | [USN-8944-1](https://ubuntu.com/security/notices/USN-8944-1) |
| Ubuntu 26.04 LTS | `ca-certificates` 20260816ubuntu0.26.04.1 | [USN-8944-1](https://ubuntu.com/security/notices/USN-8944-1) |
| RHEL 8, 9, 10 | Red Hat's published mitigation now; updated `ca-certificates` when released | [RHSB-2026-012](https://access.redhat.com/security/vulnerabilities/RHSB-2026-012) |

Application owners can confirm the patch is in place with `dpkg -l ca-certificates` (Ubuntu) or `rpm -q ca-certificates` (RHEL) and compare against the vendor notice.

## Step 2: Check the Python virtual environment

A virtual environment is vulnerable if its `certifi` is older than **2026.10.19** or its `pip` is older than **26.2.2**, the releases that drop the compromised root ([certifi advisory](https://github.com/certifi/python-certifi/security/advisories/GHSA-6q48-6c7w-9gw2), [pip 26.2.2](https://github.com/pypa/pip/releases/tag/26.2.2)). Patching the OS does not fix these, because each venv carries its own copy of the root certificates.

Activate the environment and check both bundles:

```bash
python3 --version  # Python versions less than 3.10 are EOL and no longer supported for pip.  Upgrade to python 3.10 before proceeding.
source /path/to/venv/bin/activate

python3 - <<'EOF'
import certifi
from pip._vendor import certifi as pip_certifi
for name, path in (("certifi", certifi.where()), ("pip vendored certifi", pip_certifi.where())):
    bad = "emSign ECC Root CA - G3" in open(path).read()
    print(f"{name:22} {'VULNERABLE' if bad else 'ok'}  {path}")
EOF
```

Any line that prints `VULNERABLE` needs fixing. Upgrade both packages in place:

```bash
python -m pip install "pip @ https://files.pythonhosted.org/packages/3f/06/3e4420302363fbb71be3c7f35ce042f54bfca330e3e66b3a2e3c4cca579f/pip-26.2.2-py3-none-any.whl#sha256=fd7f4dfa8190e014013b9f08f3e973f4e4c26ffd179a204895533a7254813a70"
python3 -m pip install --upgrade "pip>=26.2.2" "certifi>=2026.10.19"
```

Repeat for every virtual environment on the host, including those used by cron jobs and services. Then rerun the check. QualysETL users should follow Step 3 instead of upgrading in place.

## Step 3: Optional Step - QualysETL users — reinstall QualysETL

Once the OS is patched, if the commands from Step 2 executed in your python virtual environment did not result in any issues, then you can go onto next step.  To be cautious, the official QualysETL install-or-upgrade instructions have been updated below to bring the QualysETL virtual environment up to current package versions. QualysETL calls the Qualys API through `requests`, which trusts the `certifi` bundle in the virtual environment, so this step protects QualysETL's API traffic by upgrading `certifi` and `pip`.

The installer works in two stages:

- It creates a temporary virtual environment (`~/qetl_venv_tmp`) and installs the latest `qualysetl` from PyPI.
- It then runs `qetl_setup_python_venv`, which creates `/opt/qetl/qetl_venv` and upgrades `pip`, `setuptools`, `wheel`, `requests`, `boto3` and QualysETL's other dependencies from PyPI before installing QualysETL. On a new virtual environment, `certifi` is installed at its current version as a dependency of `requests`.

Follow these instructions to execute standard [QualysETL install-or-upgrade instructions on PyPI](https://pypi.org/project/qualysetl/):

1. Log in as the non-root user that runs QualysETL and ensure no QualysETL applications are running.
2. Exit any active virtual environment, then move the current /opt/qetl/qetl_venv to backup /opt/qetl/qetl_venv_backup_01 for reference.

   ```bash
   deactivate
   mv /opt/qetl/qetl_venv /opt/qetl/qetl_venv_backup_01
   ```
3. Exit any active virtual environment, then run the official script:
   ```bash
   #!/usr/bin/env bash
   # Login as user that will execute qetl_manage_user
   # Install Application in Python Virtual Environment /opt/qetl/qetl_venv
   # Exit if in a Python virtual environment
   [ -n "$VIRTUAL_ENV" ] && { echo "Please deactivate the virtual environment and rerun this script."; exit 1; }
   python3 -m venv ~/qetl_venv_tmp || { echo "Could not create virtual environment. Determine issue with python3 -m venv ~/qetl_venv_tmp and rerun this script."; exit 1; }
   source ~/qetl_venv_tmp/bin/activate
   python -m pip install "pip @ https://files.pythonhosted.org/packages/3f/06/3e4420302363fbb71be3c7f35ce042f54bfca330e3e66b3a2e3c4cca579f/pip-26.2.2-py3-none-any.whl#sha256=fd7f4dfa8190e014013b9f08f3e973f4e4c26ffd179a204895533a7254813a70"
   python3 -m pip install --upgrade "pip>=26.2.2" "certifi>=2026.10.19"
   python3 -m pip install --upgrade qualysetl
   deactivate
   ~/qetl_venv_tmp/bin/qetl_setup_python_venv /opt/qetl
   source /opt/qetl/qetl_venv/bin/activate
   python -m pip install "pip @ https://files.pythonhosted.org/packages/3f/06/3e4420302363fbb71be3c7f35ce042f54bfca330e3e66b3a2e3c4cca579f/pip-26.2.2-py3-none-any.whl#sha256=fd7f4dfa8190e014013b9f08f3e973f4e4c26ffd179a204895533a7254813a70"
   python3 -m pip install --upgrade "pip>=26.2.2" "certifi>=2026.10.19"
   echo "Follow instructions output from qetl_setup_python_venv"
   ```

4. Answer `yes` when `qetl_setup_python_venv` asks to create the Python virtual environment. Your data and credentials in `/opt/qetl/users` are not touched.


**Existing installations:** We've removed `/opt/qetl/qetl_venv` in step 2 above to ensure a fresh upgrade of requests and certifi and pip.

If you run QualysETL in a container, rebuild the image after Step 1 so it picks up both the patched OS trust store and the new virtual environment.

## Step 4: Validate with pip list

The environment is fixed when `pip list` shows `pip` at 26.2.2 or later and `certifi` at 2026.10.19 or later.

Activate the virtual environment and list both packages (for QualysETL, use `/opt/qetl/qetl_venv`):

```bash
source /opt/qetl/qetl_venv/bin/activate
pip list | grep -Ei '^(pip|certifi) '
```

Expected output (your versions may be newer):

```
certifi    2026.10.19
pip        26.2.2
```

| Package | Minimum safe version |
| --- | --- |
| pip | 26.2.2 |
| certifi | 2026.10.19 |

## References

- [Red Hat: RHSB-2026-012 Certificate compromise - eMudhra](https://access.redhat.com/security/vulnerabilities/RHSB-2026-012)
- [Ubuntu: USN-8944-1 ca-certificates update](https://ubuntu.com/security/notices/USN-8944-1)
- [Ubuntu blog: eMudhra CA compromise fixes available](https://ubuntu.com/blog/emudhra-ca-compromise-fixes-available)
- [Python Software Foundation: Information for Python users regarding the eMudhra / emSign CA incident](https://discuss.python.org/t/information-for-python-users-regarding-emudhra-emsign-ca-certificate-security-incident/109576)
- [certifi advisory GHSA-6q48-6c7w-9gw2](https://github.com/certifi/python-certifi/security/advisories/GHSA-6q48-6c7w-9gw2)
- [pip 26.2.2 release](https://github.com/pypa/pip/releases/tag/26.2.2)
- [QualysETL on PyPI: install and upgrade instructions](https://pypi.org/project/qualysetl/)
- [Mozilla Security Blog: Protecting Firefox users following the compromise of eMudhra intermediate certificates](https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/)
