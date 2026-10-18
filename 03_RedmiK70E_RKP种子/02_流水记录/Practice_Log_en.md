# Redmi K70E: Practice record

[中文](流水记录_zh-CN.md) | English

This record follows the chronology and causal decisions in the source project. Each stage states the observation, result, and next decision. AI organization is not independent human re-execution. Raw commands, key values, and device-specific components are excluded.

## 01. August 28: native baseline

- Observation or action: Stock RKP/Google flow, CSRv3, and certificate acceptance fields were reviewed.
- Result and decision: The native request became the control for later diagnostics.

## 02. August 29: DICE and BCC

- Observation or action: Raw CDI, BCC, repeated CSR, and current identity signatures were compared.
- Result and decision: Similarly named 32-byte values were not treated as interchangeable.

## 03. August 30–September 1: KeyMint path

- Observation or action: The native per-key scalar was compared with device DICE identity.
- Result and decision: A per-attestation-key private scalar was neither final CDI_Leaf nor the UDS root.

## 04. September 3: recovery basis

- Observation or action: Exact inputs, recovery material, and failed branches were recorded.
- Result and decision: Recoverability was addressed before another dynamic observation.

## 05. September 5: exact-frame TEE

- Observation or action: A build-limited loader error path and candidate runtime were investigated.
- Result and decision: No OEM signature was forged. Candidate execution and subsequent stock restoration were evidenced separately.

## 06. One-shot early binding

- Observation or action: The active window, identity observation, and automatic recovery were correlated.
- Result and decision: A later stock state did not erase the earlier observation.

## 07. Same-run zero-key CSR

- Observation or action: Challenge, DeviceInfo, DICE chain, and identity signature were retained in one run.
- Result and decision: Evidence from an older batch was not spliced into the new CSR.

## 08. Final CDI_Leaf capture

- Observation or action: The DICE-chain leaf public key and signature were used to identify the captured material.
- Result and decision: Raw CDI and native per-key scalar remained distinct.

## 09. Fresh software key

- Observation or action: A separate P-256 software private key was generated for certification.
- Result and decision: The device identity key authenticated the request; the new key matched the eventual leaf certificate.

## 10. AuthenticatedRequest shape

- Observation or action: Complete request, CSRv3 fields, Provisioning Info, and protocol version were reviewed.
- Result and decision: A bare DICE chain was insufficient, and unknown fields were preserved.

## 11. Old NOT_APPROVED inference

- Observation or action: An early run changed both DICE and DeviceInfo and attributed rejection to DICE.
- Result and decision: This was correlation, not proof of one-field causation.

## 12. Causal correction

- Observation or action: A later response with modified DICE and clean DeviceInfo was retained.
- Result and decision: Its approval disproved the claim that every modified DICE request must be rejected.

## 13. Certificate closure

- Observation or action: Server status, software-key/leaf pairing, chain signatures, and validity were checked separately.
- Result and decision: HTTP success, cryptographic validity, and provisioning approval were distinct outcomes.

## 14. Device restoration

- Observation or action: Boot chain, TEE state, and result archive were reviewed without another write.
- Result and decision: Only same-run results with complete evidence were counted. Raw identity material is not public.
