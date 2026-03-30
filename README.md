# Hermeto: `uv` Backend Prototype

This repository contains the functional proof-of-concept referenced in the **GSoC 2026 Native `uv` Backend Proposal**. 

It implements the proposed "Export Bridge" architecture, demonstrating static lockfile parsing, integration with Hermeto's async fetcher, and a verified network-isolated installation (path B).

---

## Quick Start & Verification

To verify the offline installation bridge locally:

**1. Clone and Install**
```bash
git clone https://github.com/notjivi/hermeto-uv-prototype.git
cd hermeto-uv-prototype
python3 -m venv .venv
source .venv/bin/activate (.venv\Scripts\activate on windows)
pip install -e .
```

**2. Execute the Secure Prefetch**
Run the Hermeto fetcher against a directory containing a `uv.lock`.
```bash
hermeto fetch-deps --source . --output ./out x-uv
```

**3. Verify Air-Gapped Installation**
Disable the network and route `uv` to the sandboxed artifacts using the generated bridge.
```bash
export UV_FIND_LINKS="$(pwd)/out/deps/uv"
export UV_OFFLINE="true"
uv pip install -r ./out/hermeto-uv-export.txt
```
