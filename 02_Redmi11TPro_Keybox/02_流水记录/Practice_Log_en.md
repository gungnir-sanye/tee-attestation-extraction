# Redmi 11T Pro: Practice record

[中文](流水记录_zh-CN.md) | English

This record follows the chronology and causal decisions in the source project. Each stage states the observation, result, and next decision. AI organization is not independent human re-execution. Raw commands, key values, and device-specific components are excluded.

## 01. Target and read-only boundary

- Observation or action: The investigation was limited to the existing Keybox read path, with build, slot, TEE, and TA hashes recorded.
- Result and decision: An identical model was not treated as an identical binary; a version mismatch stopped reuse.

## 02. Stock-call baseline

- Observation or action: Unmodified return state, KeyMint behavior, and recovery material were observed.
- Result and decision: Later changes had to be assigned to a specific security boundary rather than an old log or parser fixture.

## 03. Temporary research privilege

- Observation or action: Success and failure conditions for temporary privilege and boot-chain research were compared.
- Result and decision: Normal-world privilege alone did not establish access through the TEE caller gate.

## 04. TA container analysis

- Observation or action: The encrypted ELF, wrapped material, and OEM signature checks were identified.
- Result and decision: No OEM signing key was obtained; repacking the whole container was not a viable signing path.

## 05. Caller-gate location

- Observation or action: The Keybox TA read-only branch was related to its normal caller-identity check.
- Result and decision: The research kept the target request distinct from removal of the whole gate.

## 06. Loader behavior

- Observation or action: The exact-version uTsecManager candidate and error behavior were compared.
- Result and decision: Successful loading and actual Keybox return were separate evidence questions.

## 07. Temporary override observation

- Observation or action: A candidate TA was observed under a recoverable experimental state.
- Result and decision: The override was neither persistent installation nor vendor signing.

## 08. Client return

- Observation or action: Return state, descriptor length, payload length, and errors were retained.
- Result and decision: One sample had a 168-byte descriptor and an 8,578-byte payload; these are not cross-build constants.

## 09. EC/RSA parsing

- Observation or action: Entries were split and parsed without accepting out-of-bounds data or stale tails.
- Result and decision: EC and RSA each formed structurally valid entries.

## 10. Certificate acceptance

- Observation or action: Each private key was paired with its leaf certificate, issuer signatures, and Android attestation fields.
- Result and decision: Valid format was only the first gate; key and certificate relationships had to agree.

## 11. Recovery review

- Observation or action: Original slot, boot chain, and TEE state were checked, with failed attempts retained.
- Result and decision: The final result and recovery were recorded separately. No flashing artifacts are in the public copy.
