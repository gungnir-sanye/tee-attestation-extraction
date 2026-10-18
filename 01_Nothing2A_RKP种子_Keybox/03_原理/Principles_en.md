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

---

[Project home](../../README_en.md) · [Complete detailed principles report](../../Detailed_Principles_Report_en.md)
