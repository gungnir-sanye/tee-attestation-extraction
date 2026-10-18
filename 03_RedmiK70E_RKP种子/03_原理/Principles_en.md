# Chapter 3 — Redmi K70E: proper DICE, CDI_Leaf, and Google RKP certificates

## 3.1 How the research target changed

The first K70E goal was to find an earlier RKP seed or UDS root.

The records then showed that:

- raw CDI, CDI_Leaf, and native per-key scalar are different objects;
- final CDI_Leaf is affected by boot measurements;
- modifying TEE or boot components can change the DICE chain;
- a 32-byte value may be a valid scalar without being UDS;
- an earlier-stage value can be contaminated by the modified measurement state;
- one identity value does not automatically produce a Google-accepted request.

The route therefore changed to:

1. establish a stock RKP baseline;
2. distinguish raw CDI from final CDI_Leaf;
3. capture final CDI_Leaf during one secure-world boot;
4. reuse the same-run DICE and DeviceInfo source;
5. generate a fresh software P-256 private key;
6. obtain the Google certificate chain for that public key;
7. restore the device to the stock baseline.

---

## 3.2 K70E DICE shape

The K70E records are consistent with a proper measured-DICE chain:

~~~text
UDS root
    ↓
Stage-one measurement
    ↓
Stage-two measurement
    ↓
TEE / KeyMint measurement
    ↓
Final CDI_Leaf
    ↓
RKP CSRv3
~~~

The abstract derivation is:

~~~text
CDI_next = KDF(CDI_current, measurement, code_id, configuration)
~~~

Final CDI_Leaf is therefore not a fixed model-wide seed.

Changing any of the following can change the leaf:

- preloader;
- LK;
- TEE;
- init_boot;
- active slot;
- measurement configuration;
- secure-world loading state.

A historical leaf cannot be hardcoded into a new run, and one K70E cannot provide a portable identity for another K70E.

---

## 3.3 Raw CDI, final CDI_Leaf, and native per-key scalar

The K70E records distinguish at least three 32-byte classes.

### Raw CDI

Raw CDI is an input to later DICE or KeyMint derivation. It is not automatically:

- the final P-256 private key;
- the identity used directly by the CSR;
- UDS;
- or a seed accepted directly by the host-side software.

### Final CDI_Leaf

Final CDI_Leaf is the leaf private scalar after the current DICE and KeyMint derivation. It is the identity used to sign the device-side request in the recorded software route.

### Native per-key scalar

A native per-key scalar belongs to one concrete RKP attestation key. It can match its MacedPublicKey and certificate, but it is not the CDI_Leaf identity.

The correct relationship is:

~~~text
raw CDI
    ↓
DICE / KeyMint derivation
    ↓
final CDI_Leaf identity scalar
    ↓
device identity CSR signature

Separate path:
KeyMint generates a new P-256 attestation key
    ↓
public key enters KeysToSign
    ↓
Google signs that public key
~~~

The two private keys should not be equal. They serve different roles.

---

## 3.4 TEE exact-frame research

The main TEE lesson was:

> A modified TEE cannot be treated as an arbitrarily repacked TEE.

It is also important to describe the signature boundary correctly. The OEM RSA signature bytes were not forged. The verifier still ran, and body/header digest checks remained. The research changed the SProvider error branch so that a non-zero verification result no longer immediately blocked later loading, while container parsing, decompression, and length contracts still had to succeed.

This is a loader error-handling boundary, not a recovered OEM signing private key:

~~~text
RSA verification result = failure
    ↓
Original logic: enter rejection branch
Research logic: continue into later parsing
~~~

An earlier full recompression candidate damaged the LZ4 frame tail, certificate material, DTB, or block boundaries and caused a logo loop. The later exact-frame design was:

~~~text
Preserve the original TEE frame layout
    ↓
Change only the minimum bytes near the target instruction
    ↓
Keep decompressed content length unchanged
    ↓
Keep unrelated compressed blocks unchanged
    ↓
Keep frame tail, certificates, and DTB unchanged
    ↓
Recheck image structure and hashes
~~~

The objective was not to solve the OEM TA signature. It was to preserve the digest and loading contract while changing only the targeted error branch.

---

## 3.5 One-shot early bind and automatic recovery

The K70E experiment was not designed to leave a modified TEE permanently installed.

The recorded safety design was:

~~~text
Stage a stock TEE recovery image
    ↓
Arm only when device, slot, size, and hashes match
    ↓
Boot once with the candidate TEE
    ↓
Capture CDI_Leaf and the same-run CSR
    ↓
Restore the disk TEE to stock after boot completion
    ↓
Do not reboot the current secure-world instance
    ↓
Next boot returns to stock TEE
~~~

This produces an important observation:

> Seeing a stock hash on disk after boot does not prove that the candidate TEE never ran.

Automatic restoration protects the next boot while preserving the current secure-world result long enough to finish the read-only capture.

---

## 3.6 Same-run identity capture

The final capture requires a strict order:

~~~text
Stock/approved research environment
    ↓
Native zero-key CSR
    ↓
Check DICE chain, outer signature, and DeviceInfo
    ↓
Capture final CDI_Leaf scalar
    ↓
Derive SPKI
    ↓
Constant-time compare with the same-run CSR leaf SPKI
    ↓
Persist the identity only if they match
~~~

The following sequence is invalid:

~~~text
Capture a 32-byte value today
    ↓
Find a CSR from another run later
    ↓
Combine the files
    ↓
Hope that Google accepts it
~~~

Because CDI_Leaf is measurement-bound, the scalar, DICE chain, DeviceInfo, challenge, and CSR must belong to one run.

---

## 3.7 The two keys used by google-rkp-sw

The software-provisioning phase contains two different private keys.

### Device identity private key

Used to:

- sign the fresh challenge;
- sign the device identity data;
- prove that the request belongs to the device DICE/CDI identity.

### Software-generated P-256 private key

Used to:

- contribute its public key to KeysToSign;
- receive the RKP leaf certificate;
- support later Android Key Attestation use.

The complete relationship is:

~~~text
CDI_Leaf identity private key
    ↓
Signs the device identity CSR
    ↓
Google verifies DICE and DeviceInfo

Software P-256 private key
    ↓
Public key enters KeysToSign
    ↓
Google signs that public key
    ↓
Leaf certificate public key == software private-key public key
~~~

Requiring these two keys to be equal would incorrectly reject a valid result.

---

## 3.8 RKP request objects

The K70E route links the following objects:

~~~text
AuthenticatedRequest
  ├─ outer signed data
  ├─ DICE certificate chain
  ├─ DeviceInfo
  ├─ challenge
  └─ CSR payload

CSR payload
  ├─ request version
  ├─ KeyMint profile
  ├─ KeysToSign
  └─ canonical encoded fields
~~~

The required relationships are:

~~~text
Same-run CDI_Leaf public key
    ==
AuthenticatedRequest signing identity

Software private-key public key
    ==
KeysToSign public key

Software private-key public key
    ==
Google-returned leaf certificate public key

Leaf certificate
    ↓
Intermediate CA
    ↓
Root CA
~~~

The parser must also check canonical CBOR, CSR-version fields, DeviceInfo profile, fresh EEK/challenge, complete certificate consumption, duplicate leaves, broken chains, cycles, and unattached certificates.

---

## 3.9 K70E failure points

### Mistake one: treating native per-key scalar as CDI_Leaf

A native scalar can close the loop for one attestation key while still being unrelated to device identity.

### Mistake two: passing raw CDI directly to the host tool

Raw CDI has not necessarily undergone the current device's DICE/KeyMint derivation and cannot automatically be used as the final P-256 identity scalar.

### Mistake three: requiring the two private keys to be equal

The identity key and the software-generated attestation key have different roles.

### Mistake four: mixing CSRv3 and older fields

A field that is valid in one CSR version can invalidate another if placed in the wrong outer structure or DeviceInfo map.

### Mistake five: replacing the complete AuthenticatedRequest with a bare DICE chain

Removing the signed outer context can destroy the relationship between DeviceInfo, the identity signature, and the request payload.

### Mistake six: assuming modified DICE alone caused rejection

An early rejected run changed modified DICE and DeviceInfo together. It established correlation, not single-field causality.

### Mistake seven: dropping unknown Provisioning Info fields

An early validator discarded unknown service fields and classified a certificate containing NOT_APPROVED as PASS. Unknown fields must be retained, and provisioning status must be an explicit acceptance gate.

### Mistake eight: misunderstanding automatic TEE restoration

The disk may be stock while the current secure-world boot still came from the candidate. Boot markers, runtime namespace, TA identity, recovery records, and boot ID must be checked together.

### Mistake nine: full TEE recompression

Changing the LZ4 frame, certificate tail, DTB, or fixed-size contract can cause a logo loop. The exact-frame approach exists because those structures are part of the loader contract.

---

## 3.10 NOT_APPROVED causality correction

The most important K70E interpretation correction concerns NOT_APPROVED.

An early rejected request changed several variables at once:

~~~text
Modified DICE
    +
Historical or mismatched DeviceInfo
    +
New challenge, request, and server-side state
~~~

That combination can be rejected, but it does not prove that modified DICE is the sole cause.

A later comparison retained the exact modified DICE and replaced DeviceInfo with the clean tuple associated with the target profile. The source record obtained APPROVED in that comparison.

The defensible conclusions are:

- “DICE changed” is not by itself a sufficient rejection explanation;
- the DeviceInfo tuple and the server-side profile may be important gates;
- the result is evidence about a combined profile, not a single-field A/B experiment;
- a single-field root cause cannot be claimed without isolating the other variables.

This also shows why certificate parsing is part of security validation. X.509 and DICE signatures can be mathematically valid while provisioning status is unacceptable:

~~~text
X.509/DICE signatures valid
    ≠
Service-side provisioning status accepted
~~~

Both layers must pass before a final result is published.

---

## 3.11 K70E certificate closure

The final logical sequence is:

~~~text
Same-run stock/approved environment
    ↓
Capture final CDI_Leaf
    ↓
Derive identity SPKI and compare with native zero-key CSR
    ↓
Generate a fresh software P-256 private key
    ↓
Reuse the same-run DICE/DeviceInfo source
    ↓
Obtain a fresh challenge/EEK
    ↓
Construct canonical CSRv3
    ↓
Perform remote fetch/sign
    ↓
Receive one complete certificate chain
    ↓
Leaf certificate public key == software private-key public key
    ↓
Validate every certificate to one self-signed root
    ↓
Persist CSR, response, private key, and chain
    ↓
Restore the device to the stock baseline
~~~

The final product is not a native KeyMint private-key dump. It is:

- same-run device identity evidence;
- one newly generated software P-256 private key;
- the Google certificate chain for that public key;
- an auditable CSR, response, certificate, and recovery set.

---

---

[Project home](../../README_en.md) · [Complete detailed principles report](../../Detailed_Principles_Report_en.md)
