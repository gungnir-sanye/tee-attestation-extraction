# Detailed Principles Report: From Android TEE Keybox Research to RKP Certificate Validation

> Devices: Nothing 2a, Redmi 11T Pro, and Redmi K70E  
> Keywords: Android TEE, KeyMint, Trustlet, TA, Keybox, RKP, DICE, CDI, CDI_Leaf, UDS, CSR, certificate chains, reverse engineering

---

## 0. Scope and provenance

This is the English counterpart of the Chinese detailed principles report. It is a technical post-mortem of three real device research tracks, not a copy-and-run flashing guide.

Each chapter explains:

1. where the vulnerable or weak security boundary was located;
2. how a normal-world request reached the TEE;
3. which data was observed or exported inside the TEE flow;
4. how the host side converted the result into certificate material;
5. which mistakes occurred and why earlier conclusions were wrong.

The public archive excludes real private keys, RKP seeds, unique device identifiers, patch bytes, and directly executable deployment commands. The material was organized by AI after reading the existing project tree, source notes, ledgers, scripts, patch registers, and manifests. It is not a claim that every historical command has been independently executed again by a human.

This report is for research and defensive review on devices you own or are expressly authorized to examine. Publication grants no permission to access third-party devices, services, or identity material. Before acting, users must independently confirm authorization, applicable law and service terms, build conditions, and recovery plans. Technical conclusions may be wrong or outdated. Material is provided as-is; liability limitations in [LICENSE](LICENSE) apply only to the extent permitted by law.

---

## 1. Executive conclusions

The three devices did not reduce to one generic task called “extract the seed.”

The actual research questions were different:

~~~text
Nothing 2a
  Controlled observation of decrypted local Factory Keybox material
  plus an independent RKP-derived-material path

Redmi 11T Pro
  MediaTek TEE Keybox caller gate and TA-container boundary
  plus descriptor/payload/certificate-chain validation

Redmi K70E
  Final CDI_Leaf identity confirmation under proper DICE
  plus same-run CSRv3, DeviceInfo, and Google certificate validation
~~~

The most dangerous interpretation errors were:

- treating Keybox, RKP seed, CDI, UDS, and CDI_Leaf as one secret;
- treating a successful boot, a 32-byte value, or one HTTP 200 as final proof;
- transferring a TA, TEE, slot, or DICE conclusion from one device to another.

---

# Chapter 1 — Nothing 2a: Trustonic KeyMint and Keybox plaintext exposure

## 1.1 Research target

The Nothing 2a records contain two related but separate tracks:

1. a local Factory Keybox track;
2. an RKP-derived-material track.

The Factory Keybox goal was to obtain a parseable EC/RSA attestation keybox and its certificate chains.

The RKP goal was to determine whether a 32-byte value produced by a KeyMint TA KDF could serve as part of the current device's software RKP identity.

Both tracks use the same general KeyMint and Trustonic TEE foundation, but their result semantics are different.

---

## 1.2 The vulnerability was a chain, not one bug

The Keybox research chain can be summarized as:

~~~text
Bootloader unlocked state
    ↓
Boot-chain verification-policy weakness
    ↓
Controlled loading of modified LK/TEE components
    ↓
Trustonic MCLF/Trustlet loading boundary relaxed
    ↓
Modified KeyMint Trustlet executes in the secure world
    ↓
TA reads and decrypts the Factory Keybox secure object
    ↓
Plaintext briefly exists in a TA stack/heap object
    ↓
Plaintext is observed before the parser call
    ↓
Plaintext is copied to a mapped buffer before handler return
    ↓
A normal-world direct client reads the mapped buffer
    ↓
The host parses DER, private keys, and certificate chains
~~~

The important point is that the encrypted Keybox file was not broken in the normal world.

The security failure was the combination of:

- the ability to load a modified Trustlet in the same TEE environment;
- the normal TA decrypting the secure object;
- a new copy-out path added while plaintext was alive;
- a direct client able to retrieve the mapped buffer.

The secret became observable because it was already legitimately decrypted inside the secure world.

---

## 1.3 KeyMint to Trustlet data flow

The normal call path is approximately:

~~~text
KeyMint HAL
    ↓
Trustonic client
    ↓
Trustlet session
    ↓
Mapped normal-world buffer
    ↓
TCI request
    ↓
Secure-world notification
    ↓
TA response
~~~

The TCI request carries the command class, return status, mapped-buffer address, mapped-buffer length, and request context.

The original HAL only expects the TA to process the encrypted object. The stock TA does not return the decrypted Keybox plaintext through the input buffer.

The research direct client reproduced the communication contract rather than using the Java Keystore API. Its role was to:

1. establish the Trustlet session;
2. map the encrypted object;
3. send the same class of request used by the HAL;
4. wait for the TA response;
5. compare the buffer before and after the call.

With the unmodified TA, the mapped buffer remained unchanged. That negative result was important because it demonstrated that the stock path did not already expose plaintext.

---

## 1.4 What happened inside the TA

The relevant handler can be represented as:

~~~text
handler
  ├─ read mapped-buffer address and length
  ├─ check the input range
  ├─ copy the encrypted object to a TA heap buffer
  ├─ call the secure-object decrypt routine
  ├─ obtain a plaintext buffer
  ├─ call the Keybox parser
  ├─ release the plaintext buffer
  └─ return status
~~~

The research patch added two observation points:

~~~text
After decryption, before parsing
    ↓
Save plaintext pointer and length

After parsing, before handler return
    ↓
Copy the saved plaintext into the mapped buffer
~~~

The original decrypt routine, parser, return code, and read-only request semantics were retained.

This matters because it separates the actual weakness from a much stronger claim. The research did not implement a new cryptographic decryption algorithm and did not guess the Keybox contents from an XML file. It changed the data-flow boundary around a plaintext object that already existed in the TA.

---

## 1.5 Plaintext Keybox layout

The plaintext is a length-prefixed binary sequence, not an XML document.

The logical layout is:

~~~text
Version or Keybox number
EC private DER length
EC private DER
EC certificate count
EC certificate 0
EC certificate 1
EC certificate 2
RSA private DER length
RSA private DER
RSA certificate count
RSA certificate 0
RSA certificate 1
RSA certificate 2
~~~

The parser must not split the data using fixed offsets alone. Certificate lengths may change between builds, and a mapped buffer may contain an unmodified tail after the copied plaintext.

The safe parsing order is:

1. read each length prefix;
2. verify that the remaining buffer is large enough;
3. verify DER structure and internal length;
4. compare descriptor and payload information;
5. derive SPKI from each private key;
6. compare it with the matching leaf certificate;
7. validate the complete certificate chain.

---

## 1.6 Why the result is a Factory Keybox

Producing EC and RSA private-key files is not sufficient.

The evidence must close at three levels.

### Level one: private key and leaf certificate

~~~text
PublicKey(EC private key)  ==  PublicKey(EC leaf certificate)
PublicKey(RSA private key) ==  PublicKey(RSA leaf certificate)
~~~

### Level two: certificate chain

~~~text
leaf signed by intermediate
intermediate signed by root
root self-signed with valid constraints
~~~

### Level three: Android KeyMint cross-check

A new Android KeyStore attestation request is used to compare the resulting leaf chain with the exported Factory Keybox chain.

This excludes:

- random private keys;
- a different device's certificate chain;
- offset mistakes;
- a local test CA;
- data that is DER-shaped but not an attestation Keybox.

---

## 1.7 The Nothing 2a RKP-derived material

The RKP track is not obtained by parsing the Factory Keybox plaintext.

The records show a KeyMint TA path related to an rkp_bcc_km KDF:

~~~text
Device-bound secret
    ↓
Labelled KDF
    ↓
32-byte derived material
    ↓
RKP/CDI-related identity handling
    ↓
CSR and challenge signing
~~~

On this device and with the recorded software implementation, the 32-byte value can serve as part of the software RKP identity after same-run public-key, CSR, and signature checks.

It should not be described as a universal:

- UDS;
- raw CDI;
- CDI_Leaf format for every platform;
- cross-device root seed.

The precise claim is narrower:

> The rkp_bcc_km-derived 32-byte value observed in the Nothing 2a KeyMint path was usable as the current software RKP identity input after the recorded public-key, CSR, and signature relationships were validated.

---

## 1.8 Nothing 2a failure points

### Mistake one: treating Keybox as a normal file

A filename or input path proves only that an input object exists. It does not prove that plaintext can be read from the normal world.

### Mistake two: checking only the client return code

A successful TA return proves that the decrypt and parser path did not report an error. It does not prove that the output buffer contains the correct Keybox.

### Mistake three: treating the buffer tail as Keybox plaintext

If the copy length is shorter than the mapped buffer, the tail may still contain original input bytes. The parser must use length prefixes, DER structure, and certificate relationships.

### Mistake four: treating RKP fallback as proof that Factory Keybox is absent

A device may prefer RKP and still fall back to local Factory Keybox when the RKP service or network is unavailable.

### Mistake five: using fixed offsets across builds

Certificate and container lengths can change. Version and input hashes must be hard gates, and parsing must follow length prefixes and DER lengths.

---

## 1.9 Nothing 2a evidence chain

~~~text
Controlled Trustlet loading
    ↓
TA decrypts the Keybox in the secure world
    ↓
Plaintext observed before parsing
    ↓
Mapped buffer receives a controlled copy
    ↓
Host parses length-prefixed DER
    ↓
Private-key SPKI matches leaf-certificate SPKI
    ↓
Certificate chain validates
    ↓
Android KeyMint cross-check succeeds
~~~

If any link is missing, the result is only an observed byte sequence, not a usable Factory Keybox.

---

# Chapter 2 — Redmi 11T Pro: MediaTek TEE TA container and caller gate

## 2.1 Research target

The Redmi 11T Pro goal was to determine whether the existing MediaTek TEE Keybox read path could be invoked by an authorized normal-world client on one exact software build, and whether its output could be parsed into EC/RSA private keys and certificate chains.

This route differs from Nothing 2a:

- Nothing 2a follows a Trustonic TCI/Trustlet model;
- Redmi 11T Pro follows MediaTek TEE/GenieZone/uTsecManager;
- the TA container format is different;
- the caller checks are different;
- the Keybox payload layout is different.

---

## 2.2 Full path from boot chain to TA

~~~text
Unlocked-state boot-chain research environment
    ↓
Matched TEE image is loaded
    ↓
TEE starts its security-management modules
    ↓
uTsecManager loads the Keybox TA
    ↓
Normal-world TEE client opens a session
    ↓
A read-only Keybox request is sent
    ↓
TA checks command, parameter types, and caller mode
    ↓
TA reads the Keybox secure object
    ↓
Descriptor and payload are returned
    ↓
Host parses DER and certificate chains
~~~

Two independent questions must be answered:

1. Can the modified TEE/TA be loaded?
2. Once loaded, does the Keybox TA allow the current caller to reach the read handler?

The first is a boot-chain and TA-container problem. The second is an in-TA caller-authorization problem.

---

## 2.3 Why the TA container cannot simply be repacked

The Redmi 11T Pro TA is not a bare ELF. It contains an encrypted ELF payload and a verification envelope:

~~~text
TA file
  ├─ encrypted ELF payload
  ├─ container metadata
  ├─ wrapped symmetric key and IV
  └─ signature or digest material
~~~

uTsecManager performs the equivalent of:

1. reading the TA container;
2. processing the envelope with embedded RSA material;
3. recovering the AES-CTR key and IV;
4. decrypting the ELF;
5. checking payload integrity;
6. placing the TA into the secure-world runtime.

The research did not possess the original TA signing private key. The modified TA therefore could not acquire a genuine OEM signature.

The relevant boundary was narrower:

- retain container parsing;
- retain symmetric-material recovery;
- retain AES-CTR decryption;
- alter only the rejection branch after the verification result;
- keep the payload tied to the exact target build;
- reject unknown versions.

The source record pins the work to one exact system build, the corresponding LK and TEE, the Keybox TA, and uTsecManager hashes. An OTA, downgrade, slot change, or second device requires a new static analysis.

---

## 2.4 The Keybox TA caller gate

The read path has multiple checks:

~~~text
Command type is correct
    ↓
TEE parameter types are correct
    ↓
Caller mode matches the TA expectation
    ↓
Read-only Keybox handler executes
~~~

A normal Linux TEE client can satisfy the first two layers and still fail the caller-mode check.

The research boundary therefore was not “remove every check.” It was:

- retain command checking;
- retain parameter-type checking;
- relax only the caller gate for the read-only path;
- leave write-related paths untouched;
- use a temporary runtime override;
- recover the stock TA after reboot.

Removing the complete dispatcher checks would turn a narrowly scoped read experiment into an unbounded TA call surface.

---

## 2.5 Runtime override instead of permanent vendor modification

The recorded deployment model is a temporary runtime override:

~~~text
Stock vendor TA
    ↓
Temporary runtime replacement after boot
    ↓
Research client opens the modified TA
    ↓
Read-only request completes
    ↓
Reboot removes the override
~~~

The advantages are:

- vendor storage is not rewritten;
- the stock TA remains available;
- reboot provides a simple cleanup boundary;
- pre- and post-override hashes can be compared;
- failure recovery is easier to audit.

The limitations are equally important:

- a service may already hold a session to the old TA;
- restarting the service can change the active session;
- a visible path does not prove which code a process is using;
- the runtime state before reboot is not the same as the state after reboot.

---

## 2.6 Descriptor and payload format

The successful sample has a descriptor and a payload:

~~~text
descriptor
  ├─ magic
  ├─ payload length
  ├─ two key/certificate entries
  ├─ offsets, sizes, and counts
  └─ payload digest

payload
  ├─ length prefix
  ├─ EC private DER
  ├─ EC certificate chain
  ├─ length prefix
  ├─ RSA private DER
  └─ RSA certificate chain
~~~

The recorded sample used a 168-byte descriptor and an 8578-byte payload. These values describe that sample; they are not universal constants for every build.

The parser must validate both the descriptor and the payload:

1. check the magic and reserved fields;
2. check the payload length;
3. check the descriptor/payload digest relation;
4. read the payload length prefixes;
5. check DER internal lengths;
6. reject out-of-range offsets;
7. validate certificate counts and chain ordering;
8. compare private-key and leaf-certificate SPKIs.

---

## 2.7 Certificate acceptance

The result is considered valid only if:

~~~text
Descriptor structure is valid
    ↓
Descriptor and payload integrity checks pass
    ↓
All entries are within bounds
    ↓
EC/RSA private DER parses strictly
    ↓
Private-key SPKI == leaf-certificate SPKI
    ↓
Leaf -> intermediate -> root signatures validate
~~~

A client return code, a payload length, or a generated PEM file is not enough.

A public-key mismatch can indicate:

- a wrong offset;
- a wrong descriptor version;
- a truncated payload;
- files from different runs mixed together;
- incorrect certificate ordering;
- test data;
- or the modified TA not being the code that actually handled the request.

---

## 2.8 Redmi 11T Pro failure points

### Mistake one: assuming same model means same binary

All patches are tied to one exact build, LK, TEE, Keybox TA, and uTsecManager. A build change requires new disassembly and new hashes.

### Mistake two: modifying only the TA

If the container signature does not pass, uTsecManager can reject the TA before its payload is decrypted.

### Mistake three: changing verification and breaking the container path

Even if the rejection branch is changed, the TA still needs valid container parsing, key/IV recovery, AES-CTR decryption, and payload layout.

### Mistake four: removing the whole caller gate

The research target is a read-only caller boundary, not a universal TA dispatcher bypass.

### Mistake five: treating a bind mount as a permanent installation

A bind mount is runtime state. Reboot removes it, while a running service may retain an earlier session. Both states must be recorded.

### Mistake six: writing one slot without a restore verifier

The candidate slot, the untouched stock slot, the input hash, the readback hash, and the post-recovery service state all need independent checks.

---

## 2.9 Redmi 11T Pro evidence chain

~~~text
Exact-build baseline
    ↓
Static TA-container analysis
    ↓
Container parsing and decryption retained
    ↓
Only the read-path caller gate is relaxed
    ↓
Temporary runtime TA override
    ↓
Real TEE session opened
    ↓
Descriptor and payload returned
    ↓
EC/RSA DER parsed
    ↓
Private-key and leaf-certificate SPKIs match
    ↓
Certificate chain validates
    ↓
Stock TA restored after reboot
    ↓
Partition and service health verified
~~~

The research object is therefore not a single Keybox file. It is the combined boundary of boot loading, TA container verification, caller authorization, payload format, and recovery.

---

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

# 4. Cross-device lessons

## 4.1 The vulnerability entry and result exit are separate questions

A vulnerability entry may be:

- a boot-chain loading boundary;
- a TA-container error path;
- a caller-mode authorization gap;
- an observable plaintext lifetime;
- or a CSR path that fails to bind key origin.

The result exit may be:

- a mapped buffer;
- descriptor plus payload;
- CSR and SignedData;
- or a Google certificate chain.

Both sides need independent proof.

## 4.2 Static analysis does not replace dynamic evidence

Finding a KDF, parser, copy routine, or branch in a binary does not prove that the live device executed that path.

Dynamic evidence still needs:

- exact build and hashes;
- active slot;
- active TA/TEE;
- service state;
- request input;
- returned status;
- output material;
- recovery state.

## 4.3 Certificate validation must be relationship validation

A valid certificate chain proves signatures, but it does not automatically prove device identity.

The complete relation is:

~~~text
Private key == leaf certificate public key
CSR identity == same-run CDI_Leaf
DeviceInfo == current device profile
Request == current challenge
Result == current run
~~~

## 4.4 Failed evidence must remain visible

All three devices produced intermediate states that looked successful:

- output existed but was not the target material;
- a private key existed but was not device identity;
- HTTP 200 existed but provisioning status was unacceptable;
- the device booted but measurement state was wrong;
- a certificate chain existed but did not match the private key.

Those states must remain in the ledger and be marked as evidence, failure, or pending review.

---

# 5. Conclusion

The durable result of these three tracks is not a collection of files. It is a method that connects:

~~~text
Boot-chain loading boundary
    ↓
TEE/TA call path
    ↓
Secure-object decryption or identity derivation
    ↓
Plaintext, CSR, or payload observation point
    ↓
Structured parsing
    ↓
Private-key, public-key, and certificate relationships
    ↓
Remote issuance status
    ↓
Rollback and recovery evidence
~~~

Nothing 2a shows how Trustonic KeyMint plaintext can briefly exist in the secure world and become observable through a controlled research data-flow.

Redmi 11T Pro shows how the MediaTek TA container, uTsecManager, caller mode, descriptor, payload, and recovery boundary jointly decide whether a Keybox result is valid.

Redmi K70E shows that raw CDI, CDI_Leaf, native per-key scalar, and software-issued private keys must remain separate, and that the final certificate must be tied to the same-run identity, DeviceInfo, and challenge.

A byte sequence is not an extraction result until its structure, cryptographic relationships, device context, and recovery state all close.

This archive is intended for authorized research, CTF work, security analysis, and defensive review. Do not apply it to unauthorized devices, production systems, or third-party identity material.

