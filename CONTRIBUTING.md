# Contributing

> **Production Software — ExecutionProof Pre-Execution Governance Layer**

Thank you for your interest in ExecutionProof. This repository is the **public product surface** for the
ExecutionProof pre-execution governance layer: the API contract, architecture, worked examples, and
documentation. Contributions that improve the clarity, correctness, and verifiability of this surface are
welcome.

Please read this alongside our [SECURITY.md](SECURITY.md) (report vulnerabilities privately) and
[PRODUCTION-STATUS.md](PRODUCTION-STATUS.md) (what is and isn't built).

---

## Ways to contribute

- **Documentation fixes** — clarity, accuracy, broken links, diagram corrections.
- **API contract issues** — inconsistencies in [docs/openapi.yaml](docs/openapi.yaml), missing error cases,
  schema errors.
- **Example improvements** — additional realistic scenarios, corrections to request/response payloads.
- **Reproduction reports** — results from reproducing the published experiments (see below).

Because this repository documents a hosted governance service, the core engine implementation is not published
here. Contributions are focused on the public contract, examples, and docs.

## Development setup

You only need Python 3.11+ (or Docker) to work with the contract, schemas, and examples.

```bash
git clone https://github.com/derekhone/executionproof.git
cd executionproof

python3 -m venv .venv && source .venv/bin/activate
pip install pyyaml jsonschema openapi-spec-validator
```

## Run the verification harness

This is exactly what CI runs (see [.github/workflows/ci.yml](.github/workflows/ci.yml)). Run it locally before
opening a PR:

```bash
# 1. OpenAPI contract is valid OpenAPI 3.0
python -m openapi_spec_validator docs/openapi.yaml

# 2. YAML files parse (contract + CI workflow)
python -c "import yaml; yaml.safe_load(open('docs/openapi.yaml')); yaml.safe_load(open('.github/workflows/ci.yml')); print('yaml: OK')"

# 3. Every example scenarios.json is valid JSON
python -c "import json, glob; [json.load(open(f)) for f in glob.glob('examples/**/scenarios.json', recursive=True)]; print('examples: OK')"
```

With Docker (no local Python needed):

```bash
docker run --rm -v "$PWD":/spec -w /spec python:3.12-slim bash -c "\
  pip install -q pyyaml jsonschema openapi-spec-validator && \
  python -m openapi_spec_validator docs/openapi.yaml && \
  python -c 'import json,glob; [json.load(open(f)) for f in glob.glob(\"examples/**/scenarios.json\", recursive=True)]; print(\"examples: OK\")'"
```

All checks must pass before a change is merged.

## Reproducing the experiments

ExecutionProof's public claims are backed by preregistered experiments and case records:
**253 case records: 250 PASS, 2 FAIL, 1 GATE-STOP** across **75 preregistered experiments**.

- Experiment harnesses and testbeds:
  **[executionproof-testbeds](https://github.com/derekhone/executionproof-testbeds)**.
- Archived datasets and preregistrations: the
  **[Remnant Fieldworks Zenodo community](https://zenodo.org/communities/remnant-fieldworks)**.
- Governing standard: **[RF-100 v1.0](https://github.com/derekhone/rf-100)** (public review).

If your reproduction disagrees with a published result, please open an issue with your environment, the exact
case IDs, and your raw outputs. Disagreements backed by reproducible evidence are the most valuable
contributions we receive.

## Evidence standards

We hold ourselves — and contributions — to a simple standard: **claims must be narrower than the evidence.**

- Every factual claim in docs must be traceable to a published artifact (a case record, an experiment, the
  standard, or the contract). If it cannot be traced, it is softened or removed.
- Do not describe capabilities that are not built. If something is roadmap or pilot-only, label it as such and
  link to [PRODUCTION-STATUS.md](PRODUCTION-STATUS.md).
- Use the exact verdict tokens **ALLOW / HOLD / DENY**.
- Cryptographic claims must match reality: "hardware-backed ML-DSA-65 post-quantum signing via AWS KMS", and
  "formal RF-100 §8.4 conformance remains pending independent external review."
- Numbers must match the published figures verbatim (case counts, experiment counts). Do not round, scale, or
  estimate.

## Pull request process

1. **Fork** and create a topic branch from `main` (e.g. `docs/fix-error-table`, `contract/add-example`).
2. Make focused changes. Keep unrelated edits in separate PRs.
3. **Run the verification harness** locally; ensure all checks pass.
4. Update relevant docs (README, ARCHITECTURE, PRODUCTION-STATUS) if your change affects them, and keep
   error-code tables consistent across files.
5. Open a PR with a clear description: what changed, why, and how you verified it. Reference any issue.
6. Sign-off your commits (`git commit -s`) to certify the [Developer Certificate of Origin](https://developercertificate.org/).
7. A maintainer will review. CI must be green before merge.

### Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `docs:`, `chore:`,
`refactor:`, `test:`. Keep the subject imperative and under ~72 characters.

## Code of conduct

Be respectful, precise, and honest. We are building a trust layer; the same standard applies to how we work
together. Harassment or bad-faith conduct is not tolerated.

## Questions

For non-security questions, open a GitHub issue. For security reports, email
**derek@ownerremnantfieldworks.com** (see [SECURITY.md](SECURITY.md)).

---

*Built in faith. Tested in public. Claims kept narrower than the evidence.*

**Remnant Fieldworks Inc. — Derek Hone, CEO**
