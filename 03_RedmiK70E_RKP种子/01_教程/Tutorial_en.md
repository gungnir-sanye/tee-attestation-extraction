# Redmi K70E: Review guide for RKP identity and certificate chains

[中文](教程_zh-CN.md) | English

This guide is for offline review of recorded results. The public copy contains no raw CDI, device identity private key, TEE/TA image, firmware, or online provisioning automation. See the [principles](../03_原理/Principles_en.md) and [practice record](../02_流水记录/Practice_Log_en.md).

## 1. Separate four layers

- Raw CDI: an input or intermediate value in DICE derivation, not automatically the final identity.
- CDI_Leaf: identity material at the end of the device DICE chain.
- Native per-key scalar: KeyMint private-key material for one RKP attestation key.
- Software P-256 key: a fresh host-generated key submitted for certification.

The device identity key and the software key have different roles and should not be required to match. Capturing one layer does not establish extraction of the underlying UDS root.

## 2. Offline acceptance order

1. Bind the build, boot measurements, challenge, DeviceInfo, DICE evidence, CSR, and certificate response to the same run.
2. Verify DICE chain signatures and identity public-key relationships; distinguish final CDI_Leaf identity from a native per-key scalar.
3. Confirm that AuthenticatedRequest remains complete. A bare DICE chain is not a substitute for the formal request.
4. Check CSRv3 fields against the protocol version and preserve unknown Provisioning Info fields so the parser does not create false conclusions.
5. Separate transport success, valid signatures, approved provisioning status, and usable final certificates.
6. Match the fresh software P-256 private key to the leaf certificate, then verify the complete chain and validity period.

## 3. Diagnostic evidence boundaries

The recorded investigation observed a diagnostic candidate on one exact TEE loader error path; it did not forge an OEM signature. Restoration to stock TEE after a run does not prove that the candidate never ran. Keep boot, loading, identity capture, and restoration evidence separate.

An early experiment correlated modified DICE data with NOT_APPROVED. Later evidence does not support a single-field causal claim. DeviceInfo, DICE, challenge, and response must be compared within the same run.

## 4. Completion criteria

A complete experiment needs a valid same-run identity signature, correctly shaped CSR, target server status, a software key matching the leaf certificate, a valid certificate chain, and verifiable device restoration. Mark missing relationships as unverified or as offline cryptographic validation only.
