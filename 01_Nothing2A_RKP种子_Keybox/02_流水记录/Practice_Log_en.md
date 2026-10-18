# Nothing 2a: Practice record

[中文](流水记录_zh-CN.md) | English

This record follows the chronology and causal decisions in the source project. Each stage states the observation, result, and next decision. AI organization is not independent human re-execution. Raw commands, key values, and device-specific components are excluded.

## 01. Wrong initial Widevine target

- Observation or action: DRM and certificate clues did not explain Android KeyMint device-attestation output.
- Result and decision: The investigation returned to the KeyMint Factory Keybox, retaining this failed path as a target-selection control.

## 02. Unmodified baseline

- Observation or action: The build, HAL/Trustlet versions, KeyMint response, and RKP state were recorded.
- Result and decision: Native attestation could fall back to RKP; this did not prove that a Factory Keybox was absent.

## 03. HAL–Trustlet call mapping

- Observation or action: Normal-world requests, TCI commands, and shared-buffer bounds were matched.
- Result and decision: The relevant data was not an ordinary file. It appeared along a transient post-decryption secure-object path.

## 04. Unmodified client control

- Observation or action: The same call class was observed without a changed Trustlet.
- Result and decision: The control did not yield directly parseable Factory Keybox plaintext.

## 05. Diagnostic candidate and bounds

- Observation or action: A controlled candidate, returned length, and mapped buffer were observed in the authorized environment.
- Result and decision: Both length and content had to satisfy bounds before host parsing; a success code alone was rejected.

## 06. Keybox container parsing

- Observation or action: Length prefixes separated EC/RSA private-key DER and certificate DER; buffer tails were not accepted as data.
- Result and decision: Separate algorithm entries and their original boundaries were retained for cross-checks.

## 07. Factory Keybox acceptance

- Observation or action: Private-key SPKI was compared with leaf-certificate SPKI, followed by chain and Android attestation-extension checks.
- Result and decision: Keys, leaves, and issuer chains formed a relationship closure separate from RKP pool certificates.

## 08. RKP pool and direct request baseline

- Observation or action: Native RKP pool state and direct TEE-client behavior were compared.
- Result and decision: An existing certificate pool was not proof of recovered private keys, and direct-CSR failures remained visible.

## 09. Derived-material layer

- Observation or action: The level and call context of a 32-byte value associated with rkp_bcc_km were distinguished.
- Result and decision: It was not labeled a universal UDS root. The raw value is absent from the current `main` branch.

## 10. Direct 12-key batch

- Observation or action: New-key generation and direct CSR behavior were compared across a batch.
- Result and decision: Key generation succeeded while the direct CSR still failed, separating key material from full request construction.

## 11. Outer request variants

- Observation or action: Several signCertificates outer formats and failures were compared.
- Result and decision: Changing outer fields did not replace the native device request shape.

## 12. Native Binder shape

- Observation or action: A valid CSR structure and identity relationships were observed through the stock HAL/RKPD path.
- Result and decision: Same-run challenge, device signature, and Binder request shape became mandatory acceptance conditions.

## 13. Combined validation

- Observation or action: Research material and native CSR structure were reviewed together against the resulting Google certificates.
- Result and decision: Transport, signatures, public-key pairing, and validity period were accepted independently.

## 14. Closure and recovery

- Observation or action: Stock Trustlet/HAL state, temporary overrides, and RKP service recovery were checked.
- Result and decision: The public record preserves conclusions and relationships without the original replayable components.
