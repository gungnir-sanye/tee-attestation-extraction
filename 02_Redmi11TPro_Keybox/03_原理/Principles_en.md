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

---

[Project home](../../README_en.md) · [Complete detailed principles report](../../Detailed_Principles_Report_en.md)
