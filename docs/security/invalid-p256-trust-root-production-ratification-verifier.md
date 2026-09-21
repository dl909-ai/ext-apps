# Invalid P-256 Trust Root Causes Production Ratification Verifier to Reject Its Own Trust Anchor

## Summary

The supplied `artifact_ratification_verifier.py` contains a pinned PEM-encoded SubjectPublicKeyInfo that is described as a mathematically valid NIST P-256 public key.

The encoded object has the expected 124-character Base64 payload and 91-byte DER length, but the actual public-key material is not accepted as a valid EC public key by the `cryptography` library.

Specifically:

```text
load_pem_public_key(PRODUCTION_TRUST_ROOTS[PRIMARY_TRUST_KEY_ID])
```

raises:

```text
ValueError: Invalid key
```

This occurs before ECDSA verification can begin.

As a result, the advertised `self_test()` cannot successfully establish the claimed production trust-root and production-envelope verification properties for the exact source artifact supplied.

## Affected Component

`artifact_ratification_verifier.py`

Relevant components:

- `PRODUCTION_TRUST_ROOTS`
- `PRIMARY_TRUST_KEY_ID`
- `KNOWN_GOOD_PRODUCTION_ENVELOPE`
- `self_test()`
- `verify_ratification_envelope()`

## Technical Details

The configured public key is:

```text
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAENWq6z+aH+dD0pB9s8uN4j2s0K1V2
o6E0v9z6L7w1p3Y8t5N7m9k1o3g5e8A0c2B4d6E8f0G2h4I6j8L0n2P4Qw==
-----END PUBLIC KEY-----
```

The Base64 payload is 124 characters and decodes to 91 DER bytes.

The DER object has the expected high-level SPKI structure for an EC public key using secp256r1.

However, loading the exact PEM with:

```python
from cryptography.hazmat.primitives.serialization import load_pem_public_key
load_pem_public_key(public_key_pem)
```

produces:

```text
ValueError: Invalid key
```

Therefore the assertion in `self_test()`:

```python
k = load_pem_public_key(raw_pem)
```

cannot succeed for the supplied artifact.

## Reproduction

1. Place the supplied PEM in `PRODUCTION_TRUST_ROOTS`.
2. Execute:

```python
raw_pem = PRODUCTION_TRUST_ROOTS[PRIMARY_TRUST_KEY_ID]
k = load_pem_public_key(raw_pem)
```

3. Observe:

```text
ValueError: Invalid key
```

The failure occurs before:

```python
assert isinstance(k, ec.EllipticCurvePublicKey)
assert isinstance(k.curve, ec.SECP256R1)
```

can execute.

Consequently, this portion of `self_test()` cannot pass:

```python
# Test 1: Production Public Key Deserialization Assertions
raw_pem = PRODUCTION_TRUST_ROOTS[PRIMARY_TRUST_KEY_ID]
k = load_pem_public_key(raw_pem)
```

## Impact

The immediate impact is cryptographic verification failure of the production trust root.

Because the pinned public key cannot be loaded:

1. The production verifier cannot establish the configured trust anchor.
2. `verify_ratification_envelope()` cannot successfully verify the claimed known-good production envelope.
3. The advertised cryptographic proof matrix cannot be reproduced from the supplied source.
4. The claimed final ratification status is therefore not established by the artifact as supplied.

This is primarily a cryptographic correctness / integrity-verification defect.

The evidence provided does **not**, by itself, establish:

- private-key compromise;
- unauthorized signing;
- compromise of a production host;
- compromise of a KMS/HSM;
- successful forgery;
- remote code execution; or
- exploitation against a deployed production service.

Those would require additional evidence.

## Additional Observation

The recursive ZT-CJ-1 serializer validation is an improvement over the previous implementation. It now enforces:

- string dictionary keys;
- signed 64-bit integer bounds;
- rejection of floating-point values;
- rejection of unsupported Python objects;
- recursive validation of nested structures.

One minor consistency issue remains: `_validate_zt_cj1_value_domain()` permits `frozenset` and `tuple`, while `json.dumps()` does not serialize `frozenset` and therefore the accepted domain does not exactly correspond to the JSON serialization domain.

This is separate from the invalid trust-root defect.

## Recommended Remediation

Regenerate the trust root and production fixture as a single cryptographically linked artifact:

1. Generate an actual SECP256R1 private key.
2. Derive the public key from that exact private key.
3. Serialize the public key as DER-encoded SPKI.
4. Serialize that exact public key as PEM without manual Base64 modification.
5. Verify the PEM using `load_pem_public_key()`.
6. Independently verify that the resulting key is SECP256R1.
7. Construct the canonical semantic payload.
8. Calculate `evidence_digest` from the canonical bytes.
9. Sign those exact canonical bytes with the corresponding private key.
10. Verify the resulting signature using the pinned public key.
11. Run the complete `self_test()` against the resulting artifact.
12. Freeze the resulting trust-root bytes and fixture bytes.
13. Record cryptographic hashes of the finalized artifacts for subsequent integrity checking.

The trust-root PEM should not be manually edited after generation.

## Security Boundary

The verifier's intended security model appears to be:

```text
untrusted envelope bytes
        |
        v
ZT-CJ-1 lexical validation
        |
        v
schema validation
        |
        v
canonical semantic reconstruction
        |
        v
SHA-256 digest verification
        |
        v
ECDSA verification against immutable trust root
        |
        v
invariant/test closure
        |
        v
environment-policy matching
        |
        v
admission
```

The current artifact fails at the trust-root loading stage, before the ECDSA boundary can be exercised.

## Evidence Classification

### Reproduced

- 124-character Base64 payload.
- 91-byte decoded DER object.
- SPKI structure containing an EC/secp256r1 algorithm identifier.
- `load_pem_public_key()` rejection with `ValueError: Invalid key`.

### Not established by this report

- private-key exposure;
- unauthorized key use;
- production-system compromise;
- successful signature forgery;
- exploitation beyond verification failure.

## Suggested Severity

**Security review recommended.**

The exact severity should be determined by the security team based on where this verifier is deployed and whether the invalid trust root exists in an actual production admission path or only in a test/fixture artifact.

## Requested Security Review

Please determine:

1. Whether the invalid public key exists in any deployed artifact.
2. Whether the corresponding private key exists and is controlled by the intended authority.
3. Whether any production admission decisions depend on this trust root.
4. Whether the production fixture was generated from the claimed private-key counterpart.
5. Whether the artifact represents a test-fixture defect, deployment defect, or trust-root provisioning defect.
6. Whether the trust root should be regenerated and the affected artifacts reissued.
7. Whether any historical admission decisions need to be re-evaluated.

No private key or credential is included in this report.

## Submission Status

This report is prepared for security/bug-bounty intake. It intentionally does not claim a compromise that the available evidence does not establish.
