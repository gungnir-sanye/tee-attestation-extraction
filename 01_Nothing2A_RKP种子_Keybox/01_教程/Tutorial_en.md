# Nothing 2a: Review guide for Keybox and RKP material

[中文](教程_zh-CN.md) | English

This guide reviews the existing research and its evidence relationships. The public archive contains no device-specific patches, firmware, real keys, or executable extraction commands. Read the [device principles](../03_原理/Principles_en.md) and [practice record](../02_流水记录/Practice_Log_en.md) for the mechanism and chronology.

## 1. Fix the research target

Record the model, system build, KeyMint HAL and Trustonic Trustlet versions, and boot-chain state. Build, slot, and input hashes must belong to the same experiment. Offsets, TA layouts, and container lengths are not portable across firmware builds.

Treat Factory Keybox EC/RSA keys and certificate chains separately from material derived along the rkp_bcc_km path. Their origin, purpose, and acceptance tests differ. Evidence for one does not establish the other.

## 2. Keybox review sequence

1. Preserve the unmodified KeyMint call, device-attestation output, errors, and logs as a negative control.
2. Map the normal-world HAL request to the Trustlet command, TCI layout, and mapped buffer.
3. Distinguish an encrypted secure object, its transient plaintext, and a client-readable shared buffer. Only the last can demonstrate exposure.
4. Compare the control and diagnostic observations, including return length, buffer bounds, and content type. A successful return code alone proves little.
5. Parse the length-prefixed DER entries on the host and identify EC/RSA private-key and certificate components separately.
6. Compare the public key derived from each private key with its leaf certificate, then verify chain signatures, root, usage, and Android attestation extension.
7. Cross-check against same-run KeyMint evidence to reject random keys, self-signed chains, and old RKP pool entries.

## 3. RKP-derived material

The recorded 32-byte value belongs to a build-specific rkp_bcc_km derivation path. A review must retain the derivation label, source context, call context, length, and same-run identity relationship. Length alone does not identify a universal UDS root, and this material is not a Factory Keybox private key.

Keep a native RKP request as the control. Check whether the research material participates at the same protocol layer in identity construction. The host request, device signature, certificate response, and new key must form one consistent run. The failed direct-CSR attempts and the later use of the native Binder CSR shape are separate branches.

## 4. Acceptance matrix

| Object | Required relationship |
|---|---|
| Keybox format | Lengths, DER parsing, and EC/RSA types agree |
| Key pairing | Derived public key equals the corresponding leaf-certificate public key |
| Chain | Signatures, usage, root, and attestation extension check out |
| RKP identity | Derivation layer, same-run request, and signature agree |
| Provenance | Every conclusion traces to a build, control, log, and failure branch |

If any relationship is missing, mark the outcome unverified. A certificate or one successful return is not complete acceptance.
