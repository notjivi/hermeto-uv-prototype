# Hermeto: `uv` Backend Prototype

This repository contains the functional proof-of-concept referenced in the **GSoC 2026 Native `uv` Backend Proposal**. 

It implements the proposed "Export Bridge" architecture, demonstrating static lockfile parsing, integration with Hermeto's async fetcher, and a verified network-isolated installation (path B).

---

## Quick Start & Verification

To verify the offline installation bridge locally:

**Clone and Install**
```bash
git clone https://github.com/notjivi/hermeto-uv-prototype.git
cd hermeto-uv-prototype
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```
**Create a Minimal uv.lock Dummy Project**
```bash
mkdir demo-project && cd demo-project

cat << 'EOF' > uv.lock
version = 1
revision = 1

[[package]]
name = "colorama"
version = "0.4.6"
source = { registry = "https://pypi.org/simple" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d1/d6/3965ed04c63042e047cb6a3e6ed1a63a35087b6a609aa3a15ed8ac56c221/colorama-0.4.6-py2.py3-none-any.whl", hash = "sha256:4f1d9991f5acc0ca119f9d443620b77f9d6b33703e51011c16baf57afb285fc6" }
]
EOF
```

**Execute the Secure Prefetch**
Run the Hermeto fetcher against a directory containing a `uv.lock`.
```bash
hermeto fetch-deps --source . --output ./out x-uv
```

* **Verify the Air-Gapped Installation**
```bash
UV_FIND_LINKS="$(pwd)/out/deps/uv" UV_OFFLINE="true" UV_NO_INDEX="true" uv pip install -r ./out/hermeto-uv-export.txt
```

# Architecture: The Static Approach
Hermeto needs to stay SLSA compliant, which means prefetching has to be completely isolated. If we just run `uv sync` or `uv pip install` to grab source distributions (`sdists`), uv will try to trigger build backends like `setup.py`. That instantly breaks the sandbox by allowing Arbitrary Code Execution (ACE).

To get around this, the prototype treats everything statically:

* **Parser**: We use tomlkit to pull the exact URLs and `SHA-256` hashes straight out of uv.lock.

* **Fetcher**: Feeds those URLs directly into Hermeto's existing async downloader (`async_download_files`).

* **The Export Bridge**: Instead of fighting uv's internal cache, we generate a `hermeto-uv-export.txt` manifest and pass it through `RequestOutput`. The downstream build container can then use this for a clean, offline install.

# Security & Integrity
Security-wise, the main priority here is avoiding any accidental code execution (ACE) during the prefetch.

Instead of letting uv build `sdists`, we just fetch them as raw tarballs or zips. No Python code actually runs during the download.

Every single file is checked against the `uv.lock` hash using Hermeto's `must_match_any_checksum()`. If a hash fails, the file is immediately deleted.

To validate `sdist` metadata, I reused the existing `_check_metadata_in_sdist` logic. It just reads the `PKG-INFO` directly from the archive without extracting or executing anything.

For the actual build phase, the backend sets `UV_OFFLINE`, `UV_NO_INDEX`, and `UV_FIND_LINKS`. This physically blocks `uv` from trying to hit the network later on.
___
If you're reviewing the code and want to skip straight to the actual logic, check out: [uv/main.py](https://github.com/notjivi/hermeto-uv-prototype/blob/main/hermeto/core/package_managers/uv/main.py)

`_parse_uv_lock()`: Handles the TOML parsing and figures out whether to grab a wheel or an sdist.

`fetch_uv_source()`: The main entry point. This is where the checksum validation happens and where the Export Bridge manifest gets generated.
