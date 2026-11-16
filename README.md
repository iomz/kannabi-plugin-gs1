# kannabi-plugin-gs1

GS1 identifiers for [Kannabi](https://github.com/iomz/kannabi), as a plugin.

Kannabi core records identifiers and issues none.
This plugin is what gives a Kannabi instance GS1: without it there is no GS1 scheme, route, tool, setting or address, and nothing is missing.

## What it contributes

- Identifier schemes: `gtin`, `sgtin`, `grai`, `giai`.
- GS1 Company Prefix namespaces, managed class keys and key issuance, with an issuance ledger.
- GS1 Digital Link addresses for individual-level identifiers.
- The GTIN consistency policy and its conflict set.
- A section of the Asset representation, `extensions.gs1`.
- HTTP routes under `/api/gs1/`.
- MCP tools: `list_gs1_namespaces`, `list_managed_class_keys`, `list_gs1_issuances`.

## Layout

- `toolkit/` is GS1 as Kannabi needs it: syntax policy, check digits, key composition and Digital Link encoding. It imports nothing of Kannabi.
- `adapter/` makes the toolkit a Kannabi capability: schemes, persistence, authority checks, routes, tools and addresses.
- `web/` holds every GS1 screen.
- `index.ts` is the plugin definition Kannabi composes.

The plugin's rules are in [AGENTS.md](AGENTS.md).

## Requirements

- Kannabi plugin contract 1.
- No dependency beyond those Kannabi already has.

This repository was split from Kannabi, where these files were at `plugins/gs1/` until v0.12.0.
The history of the code before it became a plugin is in the Kannabi repository.

## Installing it

Name the plugin in the Kannabi checkout's plugin list, `kannabi.plugins.json`, pinned to a tag:

```json
{
  "plugins": [
    { "id": "gs1", "repository": "https://github.com/iomz/kannabi-plugin-gs1.git", "ref": "v0.1.0" }
  ]
}
```

```bash
pnpm plugins:sync
```

That clones this repository into `plugins/gs1/` and composes it; build Kannabi as usual afterwards.
The plugin is source built with Kannabi, not a standalone package: it imports Kannabi's modules by relative path.

A Kannabi database that already holds GS1 identifiers must keep this plugin listed.
Core restores a stored identifier through its scheme, so without the plugin those identifiers have no scheme to restore them.

## Tests

The plugin's tests are under `tests/` and run from the Kannabi checkout, with `pnpm test` and `pnpm test:integration`.
They run under `test-composition.mjs`, which composes this plugin alone, so they hold what Kannabi does with GS1 installed whatever else a deployment lists.
CI checks this repository inside a Kannabi checkout at `plugins/gs1`, against Kannabi's `main` on every change here and once a day, because Kannabi's own CI names no plugin; a run started by hand may name another Kannabi ref.

## GS1 identifier issuance

A Group may configure GS1 Company Prefix namespaces and let Kannabi issue GS1 identifiers under them.
Configuring a prefix records an assertion by one of the Group's Group Managers, with who made it and when; Kannabi cannot verify GS1 licensing and never implies that it did.
General Specifications 26.0 §1.5 states that a licensed GS1 Company Prefix entitles its holder to allocate any GS1 identification key, which is why one namespace serves all three schemes below.

Four operations stay permanently distinct, and the UI keeps them distinct too — each has its own control, because collapsing two of them into one form where an empty field meant "allocate" hid exactly the difference the domain protects:

```text
record existing identifier   an observation. No authority required, none claimed, none conferred.
adopt managed class key      a Group asserts a GTIN or GRAI asset type it already allocated
                             into a prefix it manages, so Kannabi may issue serials under it.
allocate class key           Kannabi allocates a new GTIN or GRAI asset type from that prefix.
issue individual key         Kannabi issues a GIAI, serialised GRAI or SGTIN for one Asset.
```

**A recorded identifier can never become issuance authority.**
That is structural rather than a check: a serial is issuable only under a class-key record in a namespace the acting Group manages, and recording an identifier creates no such record, so there is no route from the first operation to the fourth.
Serialising another company's GTIN is unexpressible rather than forbidden.

The three schemes differ in where the sequence comes from:

```text
GIAI    GCP ──────────────────────────────► individual asset key
GRAI    GCP ──► asset type ──► serial ─────► individual returnable asset key
SGTIN   GCP ──► GTIN ──► serial ───────────► individual trade item key
```

`plugins/gs1/toolkit/gs1.ts` is the only place any issued value is built.
A GIAI is the configured prefix followed by an unpadded decimal reference.
A GRAI asset type and a base GTIN share one computation — the prefix, a zero-padded reference filling the remaining digits, and the modulo-10 check digit — because the arithmetic is genuinely identical; they draw on separate counters, because §2.3 states that a GTIN and a GRAI sharing the same digits are different keys that do not conflict.
Ordering is by the stored sequence, never by comparing identifier strings.
Unlike a value Kannabi merely stores, an issued value's prefix boundary is known because Kannabi built it from a configured prefix — a statement about construction, not about licensing, so `gcppos1` stays unenforced for identifiers Kannabi did not build.

A prefix is four to twelve digits (§1.2.3.3), may not begin with the digits of another configured prefix, and may not fall in Restricted Circulation Number space, because an RCN SHALL NOT be encoded using any GS1 Application Identifier (§1.2.2.2.1) and every value Kannabi issues is an AI element string.
Publishing that length range is not the same as locating a prefix boundary inside an arbitrary key, which is what the unpublished GCP Length Table answers; the two claims are kept apart.
A prefix too long to leave a twelve-digit class reference is capability-limited, not invalid: it still issues GIAIs, and says so.
GTIN-8 allocation and adoption are refused, because a GTIN-8 comes from a GS1-8 Prefix allocated to Member Organisations (§1.2.3.2); a GTIN-14 with an indicator digit is refused too, because 1–8 identifies a trade item grouping derived from a base GTIN (§2.1.7) and 9 is variable measure whose measure data completes the identity (§2.1.10).
An externally observed GTIN-8 stays recordable through the ordinary identifier path, which claims nothing.

Adoption exists because §4.2.3 states that no downstream party may assign a different GTIN to a trade item that already has one: an operator holding a prefix already has GTINs, and an allocate-only design would push them into minting a second GTIN for an already-identified trade item.
Adoption verifies that the key's digits fall within the prefix the Group asserted — containment with exactly the standing of that assertion, and not a licensing check.
The record says which happened: `allocated` means Kannabi produced the reference; `adopted` means the Group asserted one it already held. Neither is inferred from the other.

Each counter may declare existing-use ranges such as `1-4,9-11,200-300`: numbers that were already unavailable when it was configured.
They normalise to sorted, disjoint, non-adjacent intervals, and allocation jumps over them by range rather than by number, so an exclusion covering a trillion references costs no more than one covering a single reference.
Exclusions are configuration and never a free list; what Kannabi issued is tracked only by the ledger.
A serial counter's guarantee is weaker than a reference counter's and is documented as such: §3.5.2 places serial non-duplication for a GTIN on the GTIN allocator as a party, so Kannabi discharges it for the serials it issues and records a commitment it cannot enforce for serials issued elsewhere.
Counters are inline state on whichever record owns them rather than one shared node type, so that difference stays visible where each is read.

`:Gs1ClassKeyAllocation` holds a managed class key — scheme, canonical form, prefix, sequence, provenance, active state, its serial counter and who asserted it.
It carries no name, description, product attribute, dimension or image: allocation bookkeeping is not a trade item, a product or an asset class, and a label would be the cheapest way to answer #50 by accretion.
`:Gs1KeyIssuance` is an append-only issuance ledger — `key`, `scheme`, `canonical`, `gcp`, `sequence`, `allocatedAt`, `allocatedForAssetId` and `allocatedBy`, with `ISSUED_UNDER` to the class key for a serialised GRAI or an SGTIN.
`allocatedForAssetId` is a property rather than a relationship, so the record outlives the Asset.
Kannabi's claim to have issued a value rests on these ledgers alone; nothing on the identifier marks its origin.
Seven constraints make the invariants schema facts: an issued value is issued once, a class key is managed once, issuance is idempotent per scheme per Asset, one GCP has exactly one set of counters, and namespaces, class keys and issuances are stably addressable.

That idempotency constraint is `(scheme, allocatedForAssetId)` rather than `allocatedForAssetId` alone, and the difference is deliberate.
One Asset may carry a Kannabi-issued GIAI, a serialised GRAI and an SGTIN at once.
The older single-property rule was true only because there was one scheme, and the closest thing to a normative basis for keeping it — §2.1.8's note that only one GS1 key should be marked on a single instrument — is sector-scoped to medical-device direct part marking, is a SHOULD, and is about marking a physical instrument rather than what identities a record may hold.
Per-scheme idempotency is a guarantee about the operation, not a claim that one scheme excludes another.

Issuance runs in one transaction that takes the write lock before deciding anything, so a double-click returns the same value without consuming a sequence number.
Repeating the operation always returns that scheme's existing issuance, whether or not the identifier is still attached, and leaves any other scheme's alone.
An issued value may be detached and reattached to the Asset it was issued for, and can never be attached to another — one scheme-neutral check on the canonical form; an externally assigned value keeps its ordinary correction semantics.
Deactivating a namespace or a class key stops new issuance under it and nothing else: the counters, the exclusions and every issued record survive, so reactivation resumes the same namespace or key.

Kannabi never reuses an allocated reference, class key or serial.
Most of those rules are **stricter than the GS1 floor** and are documented as Kannabi policy rather than as GS1 requirements: §4.2.5 permits GTIN reuse when the value was never published externally, §4.4.1.2 makes asset-identifier reuse absolute only in named cases, and §3.5.2 defers serial reuse to sector constraints.
Each stricter rule is justified by a fact Kannabi cannot observe — publication state, whether a value stayed on an asset, sector constraints — and overstating what a standard requires would be the same class of error as understating it.

Allocating a GTIN does not make Kannabi a product catalogue.
§4.2.6.1 states that GTIN management and product listing are two entirely autonomous decisions, and §4.2.6 leaves communicating a trade item's characteristics to trading partners with its allocator, in their own systems.
Kannabi says so where a GTIN is allocated, and models no product data.

One managed GCP belongs to one Group; this is a Kannabi authority boundary, not a GS1 organizational claim.
Configuring a namespace, managing class keys in it and issuing from it require being a Group Manager of that Group, so a collaborator invited as a member for Asset work does not thereby acquire the power to allocate under the operator's prefix.
Reading a namespace and its class keys stays with membership, because the prefixes a Group manages are not secret from its members and hiding them would only stop somebody finding out who to ask.
Class-key management touches no Asset and grants no Asset access; issuing an individual key additionally requires that the managing Group collaborates on the Asset, which the Group Manager role never substitutes for.

Kannabi accepts normative 4–12 digit GS1 Company Prefixes for GS1 key allocation.
The EPC Tag Data Standard imposes additional treatment for 4- and 5-digit prefixes; EPC binary encoding and EPC URI generation are outside this release and tracked separately.

Asset links use `/asset/{id}`, and Asset-scoped photo requests use `/api/assets/{id}/photos/{key}`.
The native Asset URI carries no external identifier and never changes; what Kannabi surfaces beside it is described under GS1 Digital Link below.
Company-prefix inference and GS1-Conformant Resolver behaviour remain deferred and will be designed as explicit external interfaces.

### GS1 policy version

GS1 rules are versioned data, not application conditionals.
`plugins/gs1/toolkit/syntax.ts` is derived from a pinned release of the [GS1 Barcode Syntax Dictionary](https://github.com/gs1/gs1-syntax-dictionary) and holds only mechanical content: component structure, character sets, lengths, check-digit requirements, and the `req=`/`ex=` associations.
`plugins/gs1/toolkit/gs1.ts` holds the Kannabi overlay: scheme naming such as SGTIN being AI 01 with AI 21, the individual-versus-class derivation, and the policy identity itself.

The policy version is independent of the Kannabi software version:

```ts
{ version: '2026-01-27+kannabi.3', syntaxDictionaryRelease: '2026-01-27',
  generalSpecificationsRelease: '26.0', assertedBy: 'kannabi' }
```

GS1 date-versions the Syntax Dictionary and separately releases the General Specifications, and publishes no mapping between them.
`generalSpecificationsRelease` is therefore Kannabi's assertion, marked by `assertedBy`, and must never be presented as a GS1 statement.
`gcppos1` and `gcppos2` are declared unenforced rather than silently skipped, because locating a GS1 Company Prefix inside an arbitrary key needs the GCP Length Table, which GS1 does not publish openly.
That is a different claim from the prefix length range in §1.2.3.3, which is published and is enforced when a namespace is configured; the two are kept apart deliberately.

Bumping the version has exactly two triggers:

- pinning a newer Syntax Dictionary release changes `syntaxDictionaryRelease` and resets the suffix to `+kannabi.1`;
- changing the overlay — a scheme, a derivation, a Kannabi rule — while the pinned release is unchanged increments `+kannabi.N`.

A new policy governs future acceptance only.
Stored identifiers are never revalidated or rewritten, and there is no policy migration machinery.
Each identifier's stored `policyVersion` is historical provenance: it records which policy accepted that value, not a claim that the value would still be accepted today.
A missing stamp is rejected rather than replaced with the active version, because substituting it would assert an acceptance that never happened.
One thing a version bump may not change is a scheme's canonical layout: stored identifiers keep only their canonical form and are re-parsed with the active scheme definition, so altering a layout is a breaking data change requiring migration.

`IdentityStore.open(driver)` installs and verifies uniqueness constraints before returning the store.
It owns its sessions; the caller owns the driver.
Users, Groups, and Owners use internal keys, while Assets are addressed by their native `Asset.id`.
Reporting requires an existing User who belongs to the explicitly selected Group; the Group receives collaboration access atomically with the Asset.
`reportedBy` and `reportedAt` are captured by the reporting transaction; updates accept only name, Owner, and public visibility changes, so `Asset.id` cannot be changed and identifiers move only through their own explicit operations.
Owners remain independent from Users, and new Assets are private.
The API obtains the actor from the authenticated session; store reads and mutations check current Group membership in Neo4j.
Store actor arguments are trusted internal inputs, never accepted from request bodies.
Direct database access is trusted; application validation and Neo4j uniqueness constraints jointly enforce integrity.

## GS1 Digital Link

An Asset carrying a supported GS1 identity also has a standards-defined Web address, and Kannabi both constructs it and answers it.

Three levels are kept apart, and Kannabi implements the first two:

| Level | Status |
| --- | --- |
| Constructing standards-correct GS1 Digital Link URIs | implemented |
| Dereferencing the Digital Link forms Kannabi supports | implemented |
| GS1-Conformant Resolver behaviour | not implemented |

Kannabi is **not** a GS1-Conformant Resolver and must never be described as one.
It publishes no resolver description file at `/.well-known/gs1resolver`, declares no supported primary keys, answers no linkset, and handles no `linkType` or `context`.
Two status behaviours are taken from that standard because they are simply correct HTTP: a malformed Digital Link address is `400`, a well-formed one naming nothing Kannabi holds is `404`, and neither is ever `200`.

### Addresses

`Asset.id` is the identity. `/asset/{id}` is the native Asset URI: always valid, present for every Asset, and unchanged by any identifier the Asset gains or loses.
A Digital Link URI is derived from one identifier on every read and is never stored, so it cannot drift from the identifier it comes from or become a second source of truth.

| Kannabi scheme | AIs | Digital Link primary key | Path |
| --- | --- | --- | --- |
| GTIN | 01 | AI 01 | `/01/{gtin14}` |
| SGTIN | 01, 21 | AI 01 with the AI 21 qualifier | `/01/{gtin14}/21/{serial}` |
| GRAI | 8003 | AI 8003 | `/8003/0{assetType}{serial}` |
| GIAI | 8004 | AI 8004 | `/8004/{assetReference}` |

An SGTIN is not itself a Digital Link primary key: AI 01 is, and AI 21 qualifies it, so an SGTIN's path is rooted in a trade-item key.
The GRAI path carries the whole AI 8003 value, zero filler included, which is exactly what Kannabi stores after `(8003)`.
Which AI is a primary key and which qualifiers it takes in which order is the pinned Syntax Dictionary's `dlpkey` attribute, transcribed in full — including qualifiers Kannabi does not model, so the standard's data stays complete and Kannabi's narrower coverage stays a separate statement.
A guard holds every scheme to a declared primary key followed by an ordered subsequence of one declared qualifier group, so a reordered or undeclared qualifier is a test failure rather than a plausible-looking URI.

Six GS1 CSET 82 characters are not legal in a path segment and are percent-encoded: `"` `%` `/` `<` `>` `?`.
Encoding happens in one pass and decoding exactly once, and resolution matches against the raw path, because a path decoded before it is split cannot tell a separator from a `%2F` inside a GIAI or a serial.

Nothing Kannabi emits is a *canonical GS1 Digital Link URI*: the standard reserves that term for HTTPS on `id.gs1.org`.
Kannabi's own URIs are valid and non-canonical, and the word is not used for the native Asset URI either.

### Which URI is surfaced

An Asset with an eligible GS1 identity surfaces its Digital Link URI; every other Asset surfaces its native Asset URI, unqualified and not as a lesser case.
Selection is Kannabi presentation policy rather than GS1 policy: the standard expresses a GIAI and a GTIN in either order and states the two are not equivalent, so it defines what each form means without choosing between them.

- Only an individual-level identifier is eligible. A GTIN-only Digital Link denotes a trade item, not this Asset.
- GIAI, then a serialised GRAI, then SGTIN, because the first two are primary keys whose referent is the individual asset.
- A tie within one scheme breaks on the canonical form, so the same Asset surfaces the same URI on every read.
- Issuance decides nothing. A GIAI Kannabi issued does not outrank a recorded existing one; provenance grants no precedence.

Detaching the surfaced identifier changes the surfaced URI and nothing else.

The Asset page leads with its identifiers, because that is the identity the page is about; access and provenance follow under **Details**, and collaboration below them.
Each card shows the scheme, the value and its provenance, with what the scheme identifies available from the scheme name rather than spelled out beside the provenance badge, where two statements of different kinds read as one.

The native Asset URI is shown on the Asset page only when **Show Kannabi ID** is enabled in `/admin/settings`, because it is the UUIDv7 spelled as a URL and one setting governs both.
Hiding it is presentation alone: the address still resolves, still redirects, and remains what internal references and the API use.
A surfaced Digital Link URI is unaffected by the setting.

### Moving with the Asset

An identifier mutation can retire the address the viewer is standing on — detaching the GIAI whose Digital Link they navigated to — so after a successful attach, detach or issuance Kannabi recomputes where the Asset is served and navigates there if the current address is no longer one of them.
Losing the last eligible identity navigates back to the native Asset URI.

This is post-mutation client navigation, not canonicalization, and the distinction is load-bearing:

- a viewer at a **valid non-preferred** Digital Link stays there, because that address still serves the Asset;
- a retired Digital Link requested directly still resolves exactly as before, which is `404` once it no longer identifies the Asset;
- no Digital Link is ever redirected to another.

The rule is the same for every mutation rather than special-cased per scheme: leave an address that no longer serves this Asset, stay otherwise.

Mutations themselves are addressed by the native `Asset.id`, which the page carries with every submission, never by the address the form was submitted from.
A presentation address is for reaching a page: the router re-encodes a form's action URL, so a Digital Link carrying `%2F` in a serial arrives at the action as `%252F`, and an identity recovered from it would be a different identifier or none at all.
Changing an Asset's name or visibility, uploading a photo and attaching or detaching an identifier therefore behave identically however the page was reached.

### Resolution

Every supported Digital Link form renders the Asset directly with `200`. No Digital Link form ever redirects to another, including to the Asset's preferred one: preference governs presentation, and each supported form is an independent entry point to the same referent.

`/asset/{id}` answers with `307 Temporary Redirect` to the surfaced URI when one exists.
The redirect is temporary and uncacheable because the preferred identifier can be detached at any moment; `301` and `308` would invite a client to rewrite a link that is not permanent, and `303` would claim the target is a different resource when it is the same Asset.
`/api/assets/{id}/...`, internal references and persistence keys never redirect.

Rendering an Asset and redirecting to its surfaced URI are separate operations, which is what makes a loop impossible rather than merely unlikely: the only redirect target is a Digital Link address, and a Digital Link address never redirects.

Resolution grants nothing.
A Digital Link naming an Asset the reader may not see returns exactly what one naming nothing returns, because a GTIN and serial are printed on the object and a distinguishable answer would let a label tell an outsider that this instance is held here.
For the same reason `/asset/{id}` does not redirect for an Asset the reader cannot see.
Concretely, an unauthenticated request for a **private** Asset's Digital Link answers `404` even though the address is correct and the Asset exists — the same answer an unknown identifier gets, which is the point.
Signing in as a reader entitled to that Asset makes the same address resolve.

Class-level addresses such as `/01/{gtin}` return `404`.
Whether a GTIN becomes a Kannabi referent in its own right is an open architectural question, and answering with a list of Assets now would settle it by accident.

The numeric path space at the deployment's origin is reserved for Digital Link.
Digital Link URIs are constructed at the application origin; a separate Digital Link origin belongs with resolver work.
The status and redirect are decided in the server, which the single-page build cannot do for itself, so they apply to production builds rather than to the Vite dev server.

## Capability surfaces

These exist only in a Kannabi built with this plugin. Its HTTP routes live under `/api/gs1/`. Without it none of this is present, and nothing is missing from the table above.

| Capability | Web UI | HTTP API | MCP |
| --- | --- | --- | --- |
| **Identifiers** | | | |
| Record, detach and resolve a `gtin`, `sgtin`, `grai` or `giai` | yes | the core identifier routes | `resolve_external_identifier` |
| Issue a GIAI, serialised GRAI or SGTIN | yes | `POST /api/gs1/assets/{id}/giai`, `/grai`, `/sgtin` | no |
| Read an Asset's own issuances | yes | `GET /api/gs1/assets/{id}/issuances`, and `extensions.gs1` in the Asset representation | `get_asset` |
| List the issuance ledger | no | no | `list_gs1_issuances` |
| **GS1 namespaces** | | | |
| Configure or deactivate a prefix namespace | yes | `POST /api/gs1/groups/{key}/namespaces`, `PATCH /api/gs1/namespaces/{key}` | no |
| List namespaces | yes | `GET /api/gs1/namespaces` | `list_gs1_namespaces` |
| Allocate or adopt a class key | yes | `POST /api/gs1/namespaces/{key}/class-keys` | no |
| List class keys | yes | `GET /api/gs1/namespaces/{key}/class-keys` | `list_managed_class_keys` |
| Deactivate or reactivate a class key | yes | `PATCH /api/gs1/namespaces/{key}/class-keys/{classKeyKey}` | no |
| **GS1 Digital Link** | | | |
| Dereference a supported address | yes | document address, not an `/api` route | path returned, never dereferenced |
| **Settings** | | | |
| Read the GTIN consistency policy | yes | `GET /api/gs1/settings` | no |
| Enforce or relax GTIN consistency | yes | `PATCH /api/gs1/settings` | no |
| List Assets blocking GTIN consistency | yes, when enabling is refused | `GET /api/gs1/admin/gtin-conflicts`, and the `409` refusing `PATCH /api/gs1/settings` | no |

## Development/demo dataset

The demo tools create 140 entirely synthetic Assets, three local email/password accounts, four Groups, three Owners, 47 photos using three bundled illustrations, and three GS1 Company Prefix namespaces.
Names, supported identifiers, relationships, and visibility are deterministic; internal keys, password hashes, and immutable reporting timestamps are generated normally.
Identifiers vary deliberately: the 140 Assets cycle through seven patterns, 20 each — no identifier, a class-level GTIN, an SGTIN with its trade-item GTIN, an externally assigned GIAI, an SGTIN beside an externally assigned GIAI, a serialised GRAI, and a type-level GRAI.
The checksum-valid values are synthetic examples, not identifiers for real inventory.
Demo Workshop manages two namespaces, Shared Studio one, and the remaining Groups none, so every allocation state is reachable.
Kannabi issues five GIAIs from `0614141`, whose existing-use exclusions make the issued references 5, 6, 7, 8 and 12.
Search by name using terms such as `camera`, `bench`, `studio`, or `backpack`; Owners appear in Asset details.

Run the tools on the host with Node 24, installed dependencies (`pnpm install`), and your existing local `.env` credentials.
For a fresh checkout, create `.env` with `pnpm setup:env --dev` first.
Stop `pnpm dev`/`pnpm start` before seeding, or stop the Compose app as below; leave Neo4j and Alarik running.

```sh
docker compose stop app
docker compose up -d --wait neo4j alarik
KANNABI_DEMO=local node --import tsx --env-file-if-exists=.env plugins/gs1/demo/cli.ts seed
```

Seed requires empty application data and an empty media bucket; an initialized Settings node is allowed and receives optional-photo/UTC defaults.
A repeated seed fails without modifying the existing dataset rather than duplicating it.

**Reset permanently deletes all application data in the configured Neo4j database and every object in the configured `kannabi-photos` bucket, including data you created yourself, before recreating the demo.**
It preserves database schema, bucket configuration, and infrastructure credentials.
To replace an existing local development dataset yourself, stop the app and run:

```sh
KANNABI_DEMO=local node --import tsx --env-file-if-exists=.env plugins/gs1/demo/cli.ts reset -- --yes
```

Both commands require explicit `KANNABI_DEMO=local` opt-in, non-production `NODE_ENV`, an HTTP loopback `APP_URL`, a direct loopback `bolt://` Neo4j endpoint, and a loopback HTTP S3 endpoint using the dedicated `kannabi-photos` bucket.
Use the default local Alarik setup; container service names, remote hosts, and wildcard addresses are rejected.
Demo commands also require object listing and bucket-versioning inspection permissions; enabled or suspended bucket versioning is rejected so reset cannot leave hidden media versions.
The opt-in asserts that these are dedicated, disposable development services: do not use forwarded remote ports or shared stores.
The tools refuse a responding application port; stop any other processes connected to the same stores and do not run demo commands concurrently.
If reset or seed fails partway through, keep the app stopped, fix the reported cause, and rerun the confirmed reset; database and object-storage changes cannot commit atomically together.

Resume `pnpm dev`, or `docker compose up -d --build app` for the full local deployment, with `APP_URL=http://127.0.0.1:3000`.
Use the same local configuration and infrastructure credentials as before.
All demo accounts use password **`Kannabi-demo-only-2026!`**; never expose this dataset or these credentials on a public deployment.

| Account | Role | All | Mine | Group access | Public |
| --- | --- | ---: | ---: | ---: | ---: |
| `evaluator@demo.invalid` | System administrator | 124 | 64 | 120 | 34 |
| `collaborator@demo.invalid` | Shared-Group collaborator | 88 | 56 | 72 | 34 |
| `outsider@demo.invalid` | Unrelated Group member | 50 | 20 | 20 | 34 |

Counts above apply before searching or editing; scopes overlap.
The evaluator cannot read 16 private Assets in Private Store, despite being a system administrator.
Mine filters immutable reporting provenance within readable Assets and grants no access.
The evaluator can open Administration → Members and Settings; the other accounts cannot.
The tools add no demo fields or relationships to the domain model.
`pnpm test:integration` verifies seed and destructive reset on disposable Neo4j and Alarik containers only.

## License

MIT. See [LICENSE](LICENSE).
