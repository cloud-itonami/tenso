# cloud-itonami/tenso

**This repo is a library, not the file-transfer service.** It holds one pure
`.cljc` namespace — `tenso.murakumo` — which turns *manifest cell + attestation
set* into *a plan of MST put-record effects*, and refuses to produce any effect
until every declared gate is attested. It has no crypto, no network, no storage
and no entry point.

The Signal-E2E file transfer product that the name "tenso" (転送) refers to lives
in a different repo. See [Boundary](#boundary).

```
tenso.murakumo :  (cell-key, attestations, input)  ->  {:status :blocked  :effects []}
                                                   ->  {:status :ready    :effects [...]}
```

## Boundary

| repo | kind | what is in it |
|---|---|---|
| **`cloud-itonami/tenso`** (here) | `lib` | `tenso.murakumo` — pure planner, 205 lines, no I/O |
| `kotoba-lang/tenso` | `app` (`com-etzhayyim-app-tenso`) | the actual service: `appview/etzhayyim-wasm-tenso-t3ns0f1l`, `kotoba/` |

⚠ **`CLAUDE.md` in this repo describes the app, not this library.** It is a
near-copy of the app repo's `CLAUDE.md` (134 diff lines between the two, measured
2026-09-02) and documents Signal X3DH, Double Ratchet, chunked B2 upload and a
`etzhayyim deploy` command against a path that does not exist here. **None of
that is implemented in this repo.** Read `CLAUDE.md` as background on the product
this planner belongs to; read this README for what the code does.

## What the planner does

`cell-specs` declares **13 cells**, each mapping one legacy cell name to one new
collection NSID. Every cell requires the same **7 gates** (`common-gates`).

- `cell-plan` — plan one cell. `:blocked` with `:missing-gates` naming every
  unattested gate, or `:ready` with one `:mst/put-record` effect per collection.
- `all-cell-plans` — the same for all 13 cells.
- `records-for` — the records a cell would write, stamped with `:actorDid`,
  `:legacyCell`, `:scaffold true`, `:constitutionalStatus "attested-plan"`.
- `missing-gates` / `gate-value` — the gate check. `gate-value` accepts a map
  (keyword or string key) or a set (keyword or string member).
- `safe-rkey` — arbitrary input to a record key. **See [untested](#known-untested-surface).**
- `put-record-effect` — the effect constructor.

**The emitted collection is not the manifest's NSID.** `actor-manifest.jsonld`
declares `com.etzhayyim.apps.tenso.transferRequest`; the planner emits
`com.etzhayyim.tenso.transferrequest` — `.apps.` dropped, name lowercased. The
manifest's spelling is preserved in `:legacy-cell`, so the mapping is recoverable
in both directions. This is what a migration scaffold is for; it is recorded here
because reading either file alone gives the wrong NSID.

## The actor DID is named three times, in two forms, and the resolving one is not the one in the code

Measured 2026-09-02:

| where | value | resolves? |
|---|---|---|
| `src/tenso/murakumo.kotoba` `actor-did` | `did:web:tenso.etzhayyim.com` | **no — `tenso.etzhayyim.com` is NXDOMAIN** |
| `actor-manifest.jsonld` `@id` | `did:web:tenso.etzhayyim.com` | same, no |
| `.well-known/did.json` `id` | `did:web:etzhayyim.com:actor:tenso` | **yes — 200** |

```
$ host tenso.etzhayyim.com
Host tenso.etzhayyim.com not found: 3(NXDOMAIN)
$ curl -o /dev/null -w '%{http_code}\n' https://etzhayyim.com/actor/tenso/did.json
200
```

Every `:actor` field this library stamps onto every effect therefore carries a
DID whose `did:web` host does not exist. That may be intended (the host is
planned, the planner is a scaffold) or it may be stale. **This README records the
measurement; it does not change the value** — `actor-did` is load-bearing for
every consumer and changing it is a behaviour decision, not a docs one.

The committed `.well-known/did.json` also differs from the document currently
served at that DID: served has `jws-2020` context, empty `alsoKnownAs`, and
services `pds.aozora.app` + an `AtprotoXrpc` dnsaddr; committed has
`ed25519-2020`, four `alsoKnownAs`, and `pds.etzhayyim.com` + `aozora.app`.
The `alsoKnownAs` GitHub URL (`etzhayyim/com-etzhayyim-tenso`) is a historical
name that still resolves here through GitHub's rename redirect.

## Run it

```bash
kbb -M:test     # 9 tests, 187 assertions
kbb -M:lint     # exit 0; 1 known warning (unused binding, murakumo.cljc:145)
```

`.cljc` with no reader conditionals in the source, so it also runs on nbb without
a JVM — see [`docs/operator-quickstart.md`](docs/operator-quickstart.md), which
walks both paths plus computing a plan and watching a gate refuse.

## Known untested surface

`safe-rkey` has **no test**. Measured 2026-09-02 by deleting its body entirely
(`(defn safe-rkey [s] (str s))`) and re-running the suite: **9 tests, 187
assertions, 0 failures — green.** No assertion ever feeds it a string that needs
sanitizing, so the sanitizer is unobserved.

Its measured behaviour today:

| in | out |
|---|---|
| `"did:web:tenso.etzhayyim.com"` | `"tenso.etzhayyim.com"` |
| `"did:web:tenso.etzhayyim.com:transfer:t3ns0f1l"` | `"tenso.etzhayyim.com-transfer-t3ns0f1l"` |
| `"../../etc/passwd"` | `"..-..-etc-passwd"` |
| `""` | `"unknown"` |
| `"  "` (whitespace only) | `"--"` — *not* `"unknown"` |

The last row is the one to look at: the `str/blank?` fallback runs *after* the
character replacement, so whitespace-only input becomes a run of dashes rather
than `unknown`. Whether that is correct is a decision for whoever adds the test.

The other five public fns are covered, and the coverage discriminates — three
mutations, three named failures (2026-09-02):

| mutation | caught by |
|---|---|
| `missing-gates` always returns `[]` | `cell-plan-blocks-when-gates-missing` |
| `put-record-effect` stamps a different actor | `put-record-effect-shape` |
| `records-for` drops `:scaffold true` | `records-for-produces-one-record-per-collection` |

## Licence

Apache-2.0 with the etzhayyim Charter Compliance Rider v3.1 — see `NOTICE`.
