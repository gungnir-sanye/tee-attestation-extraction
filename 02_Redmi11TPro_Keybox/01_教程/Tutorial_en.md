# Redmi 11T Pro: Keybox research review guide

[中文](教程_zh-CN.md) | English

This guide preserves the mechanism and acceptance order without distributing vendor firmware, modified TAs, boot images, or executable extraction commands. See the [principles](../03_原理/Principles_en.md) and [practice record](../02_流水记录/Practice_Log_en.md).

## 1. Establish a version baseline

Fix the exact build, boot slot, TEE image, uTsecManager, and Keybox TA versions and hashes. The same model does not imply identical binaries. A mismatch in any component prevents direct reuse of historical diagnostics. Preserve the unmodified call as a control.

## 2. Understand the TA and caller gate

This device follows a MediaTek TEE path. The Keybox TA is not a plain ELF file: its container involves encryption and vendor signing. The research did not obtain the OEM signing private key. The investigation concerns the existing read-only Keybox path, its caller gate, and diagnostic loading behavior in a controlled environment.

Separate evidence into four layers: reaching the research environment, loading the TA, satisfying the caller gate, and actually returning a payload. Failure at one layer cannot be hidden by changing the parser.

## 3. Parse the payload

Retain the raw return length and descriptor, then parse EC and RSA entries separately. One recorded run observed a 168-byte descriptor and an 8,578-byte payload. These are sample observations, not constants for other builds. Lengths, entry boundaries, private-key encodings, and certificate DER must agree.

## 4. Cryptographic acceptance

1. Derive public keys from the EC and RSA private keys and compare each with its leaf certificate SPKI.
2. Verify each certificate-chain signature, intended usage, and root relationship.
3. Inspect Android attestation extensions and same-run output rather than trusting a filename or one certificate common name.
4. Establish that the result came from the target read path, not built-in test keys or stale files.

## 5. Failure and closure

Repacking the whole TA container, mixing boot-chain and TA versions, changing only the TA without a viable call path, and treating temporary replacement as permanent installation produce different failures. Record loader, call, return, and parser evidence separately.

A complete review also checks that the original boot and TEE state can be restored. The public record does not contain flashing steps or deployable research components.
