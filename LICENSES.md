<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Licensing

GPUl uses a **split-license model** to keep protocol-critical software in the commons while allowing broad sharing of documentation and specifications.

## Code — AGPL-3.0-or-later

All source code in this repository — including protocol implementations, the node agent, orchestrator logic, ledger service, CLI client, shared libraries, and tests — is licensed under the [GNU Affero General Public License v3.0 or later](LICENSE-AGPL).

**What this means:**

- You may use, study, modify, and redistribute covered software under the terms of the AGPL.
- If you modify the software and deploy it over a network, you must make the corresponding source code available under the same AGPL terms.
- This ensures that protocol-critical software remains open and available to the community.

## Documentation — CC BY-SA 4.0

All non-code materials — including design documents, specifications, white papers, diagrams, and other written documentation — are licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](LICENSE-CC-BY-SA).

**What this means:**

- You may copy and redistribute the material in any medium or format.
- You may remix, transform, and build upon the material for any purpose, including commercially.
- You must give appropriate credit and distribute any adaptations under the same license.

## Which License Applies?

| Material | License | File |
|----------|---------|------|
| Source code (`.rs`, `.go`, `.py`, `.ts`, `.js`, etc.) | AGPL-3.0-or-later | [LICENSE-AGPL](LICENSE-AGPL) |
| Documentation (`.md`, specs, design docs) | CC BY-SA 4.0 | [LICENSE-CC-BY-SA](LICENSE-CC-BY-SA) |
| Configuration files (`.toml`, `.yaml`, `.json`, `Dockerfile`) | AGPL-3.0-or-later | [LICENSE-AGPL](LICENSE-AGPL) |
| Protocol definitions (`.proto`) | AGPL-3.0-or-later | [LICENSE-AGPL](LICENSE-AGPL) |
| Third-party code | Original upstream license | See individual files |

**Default rule:** Code defaults to AGPL-3.0-or-later. Documentation defaults to CC BY-SA 4.0. If a file contains an explicit license header, that header takes precedence.

## Third-Party Code

Any third-party code included in this repository retains its original license. Copyright notices and attribution for third-party dependencies are preserved in their respective files.

## Contributors

By contributing to this repository, you agree that your contributions will be licensed under the applicable default license (AGPL-3.0-or-later for code, CC BY-SA 4.0 for documentation) unless you explicitly state otherwise.
