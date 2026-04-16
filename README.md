# Howion's Unique IDentifier (HUID)

## 1. Introduction

This document specifies **Howion's Unique IDentifier (HUID)**.

HUID is a **63-bit, time-sortable integer identifier** composed of:

- a millisecond timestamp component, and
- a random entropy component.

This document is the concise normative specification for HUID.

## 2. Motivation and Design Goals

HUID is designed for systems that require:

- compact identifiers that fit naturally in signed 64-bit integer storage,
- fast generation and parsing, and
- numeric ordering correlated with generation time.

Compared to 128-bit identifier families, HUID intentionally optimizes for storage and index locality in common database systems (for example, PostgreSQL `BIGINT` + B-tree indexes). HUID does not embed machine identifiers, sequence counters, or version marker bits inside the identifier body.

On modern 64-bit CPU architectures and database engines, arithmetic, comparison, and index-key handling for 64-bit integers are a natural fast path. HUID is designed to stay on this path.

Non-goals include global strict monotonicity under concurrency and embedding deployment topology (node/worker identity) into the identifier.

## 3. Notation

### 3.1. Language

The key words **"MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY"**, and **"OPTIONAL"** in this document are to be interpreted as described in BCP 14 [RFC2119](https://www.rfc-editor.org/info/rfc2119) and [RFC8174](https://www.rfc-editor.org/info/rfc8174) when, and only when, they appear in all capitals.

### 3.2. Terms and Abbreviations

- **CSPRNG**: Cryptographically Secure Pseudorandom Number Generator
- **Unix time (ms)**: milliseconds since 1970-01-01T00:00:00Z
- **HUID epoch**: 2026-01-01T00:00:00.000Z (Unix `1767225600000` with ms precision)
- **Canonical value**: the integer value in the closed interval `[0, 2^63 - 1]`

## 4. HUID Format

### 4.1. Value Domain

A HUID value is an integer in the inclusive range: `0` to `2^63 - 1`.

Implementations MUST reject values outside this range as invalid HUIDs.

This range is intentionally chosen so that HUID values can be stored in a signed 64-bit integer while remaining non-negative.

Any input that is not an integer in this interval MUST be treated as invalid.

### 4.2. Bit Layout

HUID is composed of 63 payload bits:

- **42-bit time component** (most significant bits)
- **21-bit random component** (least significant bits)

In bit index form (most significant to least significant):

- `bits[62..21]`: `time_ms_since_2026_epoch`
- `bits[20..0]`: `random_21_bits`

or, equivalently `id = (time << 21) | random`.

### 4.3. Time Window

The 42-bit time field represents milliseconds since the HUID epoch.

- Minimum representable time offset: `0`
- Maximum representable time offset: `2^42 - 1`

Therefore, the valid wall-clock generation interval is: `2026-01-01T00:00:00.000Z` through `2165-05-15T07:35:11.103Z`.

Implementations MUST fail generation when the current clock falls outside this interval.

## 5. Generation

To generate a HUID:

1. Read current Unix time in milliseconds `unix_ms`.
2. Compute `time = unix_ms - HUID_EPOCH_MS` where `HUID_EPOCH_MS = 1767225600000`.
3. If `time < 0`, generation MUST fail.
4. If `time > (2^42 - 1)`, generation MUST fail.
5. Generate `random`, a uniformly distributed 21-bit integer using a CSPRNG.
6. Return `id = (time << 21) | random`.

Generation does not require coordination state (worker ID, sequence counter, or lock service).

## 6. Ordering and Collision Properties

### 6.1. Ordering

- HUID values are numerically sortable by `(time, random)`.
- For differing millisecond timestamps, numeric order matches timestamp order.
- Within the same millisecond, order is determined by random bits and is not monotonic by generation call order.

### 6.2. Collision Model

Within a single millisecond bucket, the random space is `2^21` values.

- Collision probability is non-zero for multiple IDs generated in the same millisecond.
- Implementers SHOULD model expected per-millisecond burst rates and assess collision tolerance with birthday-bound analysis.
- Workloads requiring strict per-millisecond uniqueness guarantees SHOULD use additional coordination or a different scheme.

### 6.3. Database Index Locality Notes (Informational)

In B-tree indexes, key shape influences write distribution.

- A strictly increasing tail sequence can concentrate concurrent inserts at right-most leaf pages.
- HUID keeps a time-ordered high part for range locality, while randomizing low bits within a millisecond bucket.
- This random tail can reduce same-page insert concentration under high concurrent same-millisecond writes, at the cost of less strict in-page locality than a pure counter tail.

Therefore, random tail bits are a deliberate trade-off for stateless generation and better concurrent write dispersion in common B-tree workloads.

## 7. Encoding and Interoperability

### 7.1. Numeric Representation

The canonical logical value is a non-negative integer in `[0, 2^63 - 1]`.

HUID defines **no UUID-like canonical textual format** (no fixed hyphenated layout, no base32/base36 canonical alphabet) by design.

If a string form is required, base10 decimal string representation is RECOMMENDED for textual interchange.

Rationale:

- It maps directly to the canonical integer value.
- It avoids alphabet/case normalization rules.
- It is compact for 63-bit values (up to 19 digits), while preserving straightforward interoperability with SQL and JSON ecosystems.

When ordering in text form is required, implementations SHOULD sort by numeric value (or by fixed-width zero-padded decimal) rather than naive lexicographic decimal string order.

### 7.2. Binary Representation

When serialized to bytes, HUID SHOULD be encoded as:

- exactly 8 octets,
- unsigned big-endian integer,
- with the most significant bit of the first octet equal to `0`.

Decoders MUST reject payloads that decode to values greater than `2^63 - 1`.

## 8. Best Practices

- Use signed 64-bit integer columns (`BIGINT`, equivalent) for storage.
- Index directly on the numeric value for efficient range and time-window queries.
- Prefer integer storage and transport; use decimal string only when the protocol or language cannot safely carry 64-bit integers.
- Use a high-quality CSPRNG provided by the host runtime or operating system.
- Ensure system clocks are synchronized (for example via NTP) to preserve expected temporal ordering semantics.
- Treat HUID as an identifier, not as an authentication secret or capability token.

## 9. Comparison to Related Schemes

The following comparison is informational.

| Scheme | Total bits | Time-sortable | Random bits | Coordination bits |
| --- | --- | --- | --- | --- |
| HUID | 63 | Yes (ms bucket) | 21 | None |
| UUIDv4 ([RFC9562](https://www.rfc-editor.org/rfc/rfc9562)) | 128 | No | 122 (effective) | None |
| UUIDv7 ([RFC9562](https://www.rfc-editor.org/rfc/rfc9562)) | 128 | Yes | 74 random/counter bits | None required by spec |
| Snowflake | 63 | Yes | Typically 0 random bits | Worker + sequence |

Notes:

- Compared to UUIDv4 and UUIDv7, HUID prioritizes compactness (63 bits) over entropy width.
- Compared to Snowflake, HUID avoids embedding node identity and sequence state, trading deterministic per-node uniqueness for simpler stateless generation.
- Compared to ULID/KSUID-class identifiers, HUID is optimized for numeric 64-bit storage and indexing.

## 10. Security Considerations

HUID is not designed to hide generation time.

- The upper 42 bits reveal millisecond time relative to a fixed epoch.
- The random component is only 21 bits; brute-force within a known millisecond bucket is feasible.
- HUID MUST NOT be used as a bearer secret, password reset token, or cryptographic nonce.

If unpredictability is security-critical, systems SHOULD use dedicated high-entropy tokens separate from HUID.

## 11. Implementation Status (Informational)

The official reference implementation (in TypeScript) is available at [github.com/howionlabs/huid-ts](https://github.com/howionlabs/huid-ts). It is informative and does not override this specification.

## 12. References

- [RFC2119 - Key words for use in RFCs](https://www.rfc-editor.org/info/rfc2119)
- [RFC8174 - Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words](https://www.rfc-editor.org/info/rfc8174)
- [RFC9562 - Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562)
- [PostgreSQL Numeric Types (`bigint`)](https://www.postgresql.org/docs/current/datatype-numeric.html)
- [PostgreSQL Indexes and `ORDER BY` (B-tree)](https://www.postgresql.org/docs/current/indexes-ordering.html)
- [Twitter Snowflake (archived source)](https://github.com/twitter-archive/snowflake)
- [ULID canonical specification](https://github.com/ulid/spec)
- [KSUID reference repository](https://github.com/segmentio/ksuid)

## License

This specification is licensed under the [CC-BY-SA 3.0](./LICENSE) © 2026 Howion Inc.

## Appendix A. HUID Example

Given:

- `unix_ms = 2026-01-01T00:00:00.123Z`
- `time = 123`
- `random = 0x15555` (decimal `87381`)

Then:

- `id = (123 << 21) | 0x15555`
- `id = 258037077`

Decoding `258037077`:

- `time = 258037077 >> 21 = 123`
- `random = 258037077 & 0x1fffff = 87381`
- `decoded_unix_ms = 1767225600000 + 123 = 1767225600123`
