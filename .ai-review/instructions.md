# Reviewing xrpl.js

xrpl.js is the canonical **TypeScript SDK for the XRP Ledger** — a monorepo of `xrpl`,
`ripple-binary-codec`, `ripple-keypairs`, `ripple-address-codec`, `secret-numbers`, and `isomorphic`.
It's a **financial primitive**: bugs in amount handling, serialization, or signing corrupt transactions
or break consensus compatibility and surface only against a live network. **rippled (the C++ node) is the
protocol source of truth** — verify field names, enum/flag values, and required fields against
rippled/xrpl.org, not intuition. The conventions below are silent-failure-prone; prioritize them.

## Amounts & numbers
- XRP amounts are **integer strings of drops** (1 XRP = 1e6 drops), never JS `number`. Convert with
  `xrpToDrops`/`dropsToXrp`; XRP→drops floors (`ROUND_FLOOR`); reject any decimal in drops. Drops range 0–1e17.
- Use **bignumber.js** for all amount math; never native `number`, and never introduce a second
  big-number library — the repo deliberately consolidated to one (mixing them causes silent precision drift).
- `Amount` is a union: `string` (XRP drops) | `IssuedCurrencyAmount {value,currency,issuer}` (`MPTAmount`
  is accepted by the `isAmount` guard but not yet in the static `Amount` type — gated on MPTv2). Validate
  amounts with the `isAmount`/`isIssuedCurrencyAmount`/`isMPTAmount` guards, not inline shape checks.
  `"XRP"` is never a valid issued currency (the `isCurrency`/`isIssuedCurrency` guards reject it). IOU
  precision ≤16 significant digits, exponent ∈ [-96, 80]; MPT values are non-negative integer strings.

## Binary serialization (ripple-binary-codec) — silent-failure-prone
- **Round-trip is the core invariant:** `encode(decode(bytes))` must be byte-identical, and
  `Type.from(json).toJSON()` must equal `json`. Any serialization change needs a round-trip test.
- Fields are defined in `enums/definitions.json` (`nth`, `type`, `isSerialized`, `isVLEncoded`,
  `isSigningField`); output is ordered by **ordinal**. Adding, reordering, or renumbering fields changes
  the wire format and breaks consensus compatibility. `definitions.json` must be synced from rippled's
  `server_definitions` — verify new fields/flag values against the **rippled implementation**, not just a
  draft XLS spec (specs and implementations diverge). It's **generated** from rippled by
  `ripple-binary-codec/tools/generateDefinitions.js`; flag hand-edits to it.
- `isSigningField` correctness is **signature-critical**: mis-marking a field silently invalidates
  signatures. `undefined`/omitted fields must not be serialized. Trailing zeros in non-XRP amounts are
  stripped before signing (`removeTrailingZeros`) so JSON↔binary round-trips.
- New `SerializedType` subclass: `from()`/`fromParser()` must be exact inverses, and the type must be
  registered (`coreTypes` / `associateTypes`) or it is unreachable at runtime.
- **Intentional, do NOT flag:** the UNLModify `Account`-field omission workaround — it replicates a known
  rippled encoding quirk and must be preserved.

## rippled is the source of truth (defensive client)
- Field names (snake_case), enum values, and optionality mirror rippled's RPC API — not SDK convenience.
- The SDK must **tolerate rippled emitting things it doesn't model**: `BaseResponse.result` and pagination
  `marker`s are typed `unknown`; requests allow extra fields; deprecated fields are retained. Do **not**
  reject, filter, or normalize unknown fields (this is the class of bug behind sibling-SDK failures where
  rippled returned a variant the SDK's model didn't include).
- Response handling: only `status: "success"` proceeds; `"error"` throws `RippledError`. Response types
  are **API-version-aware** (APIv1 vs APIv2 differ structurally) — don't collapse versions.

## Transaction models & validation
- **Adding/modifying a transaction type must update ALL five sites:** (1) the type's interface + `validate`
  fn, (2) its export in `models/transactions/index.ts`, (3) the union in `transaction.ts`, (4) its import
  there, (5) a `case` in the `validate()` switch. TypeScript does **not** enforce switch exhaustiveness, so
  a missing case silently skips validation.
- Flags need three artifacts in the type's file: a `<Type>Flags` enum, a `<Type>FlagsInterface`, **and** an
  entry in `models/utils/flags.ts` `txToFlag` — omitting the last makes valid flag objects fail as "invalid".
- Each `validate<Type>()` must call `validateBaseTransaction()` first. Prefer the
  `validateRequiredField`/`validateOptionalField` helpers for consistent error messages and
  type-narrowing; inline checks exist in the codebase and are acceptable for complex/conditional fields.
- Validate interdependent fields and flag/field dependencies **explicitly, with messages naming both**
  (e.g. `Amount` requires `Amount2`; `DeliverMin` requires `tfPartialPayment`; NFTokenMint `Issuer` ≠
  `Account`). Validate hex fields (`isHex`, non-empty) and array fields via guards (`isMemo`/`isSigner`;
  empty `Memos`/`Signers` are invalid).
- Transaction type codes come from `definitions.json` (`TRANSACTION_TYPES`); never hard-code them — a type
  absent there is unreachable.
- **Intentional, do NOT flag:** pseudo-transactions (`EnableAmendment`/`SetFee`/`UNLModify`) have no
  `validate()` case — they are ledger-created, not user-submitted.

## Cryptography & signing (security-sensitive)
- All randomness via `@xrplf/isomorphic/randomBytes` — never `Math.random`. Seed entropy is exactly 16 bytes.
- **Redact private-key values in error messages** (public keys may be shown).
- Signing is XRPL-specific: SHA-512-half prehash, RFC-6979 deterministic, low-S canonical, DER→uppercase
  hex; ed25519 verifies with `zip215=false`; strip the `"ED"` prefix before crypto ops. Wrong settings
  silently fail on-ledger verification.
- Multisigning: `SigningPubKey=""`, per-signer signatures, `Signers` sorted by numeric account; keep
  single- vs multi-sign encoding paths distinct.

## Cross-environment (browser + Node)
- Binary APIs return `Uint8Array`, not `Buffer` (Buffer is Node-only). Don't add Node-only dependencies for
  crypto/binary — use the `@noble/*` / `@scure/*` isomorphic packages; code must run in both Node and browser.

## Breaking changes & tests
- Public API / model / serialization changes are **breaking**: the package's `HISTORY.md` changelog entry
  is required (it's in the release checklist), and user-facing breaking changes should also get a
  `MIGRATION.md` before/after. Flag a breaking change that has no `HISTORY.md` entry.
- Ledger-touching changes (e.g. a new transaction type) need integration tests against a rippled node, not
  just unit tests (per `CONTRIBUTING.md`); serialization changes also need a round-trip test (see the
  serialization section). Test **correctness** (a test that calls the wrong API or masks a real failure) is
  in scope; test **style** is not.

## Test conventions (do NOT flag these idioms)
- **`: any` in `packages/xrpl/test/models/**` is deliberate** — model tests build intentionally
  malformed objects (`let tx: any`, then mutate fields) to drive negative validation; don't flag it
  as loose typing.
- **Model-validation tests assert the exact `ValidationError` message via `assertTxValidationError` /
  `assertTxIsValid`, which check BOTH the type-specific validator AND generic `validate()`.** Flag an
  ad-hoc `assert.throws` or truthiness-only assertion here; don't flag the dual-validator pattern.
- **Integration-test idioms, all intentional (do NOT flag):** retry/resubmit loops on
  `tefPAST_SEQ` / `tefMAX_LEDGER` and `TimeoutError` / `NotConnectedError` (documented, deliberate
  `no-await-in-loop` disables); the hardcoded genesis account/secret (`snoPBrX…`) is the public
  standalone test key, not a leaked credential; and string ledger fields (`Data` / `URI` /
  `DIDDocument` / `Domain` / `Memo`) are hex blobs (literal hex or `convertStringToHex`), not malformed.
