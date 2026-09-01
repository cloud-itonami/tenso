# Operator quickstart — cloud-itonami/tenso

Five tasks. Every command below was run against this tree on **2026-09-02** and
the pasted output is what came back. If a command does something other than what
is written here, that is a finding — not a typo in your terminal.

This repo is a **pure library** (see [`../README.md`](../README.md)). There is
nothing to deploy, nothing to start and no credential to hold. Everything here is
local.

## 0. Get the tree

The repo is a west project of the `com-junkawasaki/root` superproject.

```bash
cd <superproject>
west update --fetch smart cloud-itonami-tenso
cd orgs/cloud-itonami/tenso
```

⚠ The west checkout names its remote after the **org**, not `origin` — so
`git fetch origin` fails here with "repository exists?" and that failure means
nothing. Use the org name:

```bash
$ git remote -v
cloud-itonami	git@github.com:cloud-itonami/tenso (fetch)
$ git fetch cloud-itonami
$ git rev-list --left-right --count HEAD...cloud-itonami/main
0	0
```

The checkout sits on a detached HEAD at the pinned commit. That is normal for a
west project; do not "fix" it by checking out `main`.

## 1. Run the tests — JVM path

This is the path `deps.edn` declares.

```bash
$ clojure -M:test

Running tests in #{"test"}

Testing tenso.murakumo-test

Ran 9 tests containing 187 assertions.
0 failures, 0 errors.
```

Exit 0. First run resolves `io.github.cognitect-labs/test-runner` and needs
network; later runs are offline.

## 2. Run the tests — no JVM

`src/tenso/murakumo.cljc` contains **no reader conditionals**, and the test file's
only one is the `ExceptionInfo` class in `thrown?`. So the same suite runs on
nbb, which matters on a machine already carrying JVMs.

```bash
cat > /tmp/tenso-run.cljs <<'EOF'
(ns tenso-run (:require [clojure.test :as t] [tenso.murakumo-test]))
(def r (t/run-tests 'tenso.murakumo-test))
(when (or (pos? (:fail r)) (pos? (:error r))) (js/process.exit 1))
EOF
nbb --classpath "src:test" /tmp/tenso-run.cljs
```

```
Testing tenso.murakumo-test

Ran 9 tests containing 187 assertions.
0 failures, 0 errors.
```

Same 9 / 187 as the JVM path. Exit 0.

⚠ Redirect to a file and read `$?` from that, not from a pipe — `nbb ... | tail`
reports `tail`'s exit code, and a suite killed by `timeout` then looks identical
to a suite that passed.

## 3. Lint

```bash
$ clojure -M:lint
src/tenso/murakumo.cljc:145:14: warning: unused binding input
linting took 2356ms, errors: 0, warnings: 1
```

(The `2356ms` is wall-clock and will differ; `errors: 0, warnings: 1` is the part
that should hold.) Exit 0 — the alias sets `--fail-level error`, so this one warning is not a
failure. **It is expected.** If you see a second warning, you introduced it.

## 4. Compute a plan, and watch the gate refuse

This is the whole point of the library. `cell-plan` will not hand you an effect
until all 7 gates in `common-gates` are attested.

```bash
cat > /tmp/tenso-plan.cljs <<'EOF'
(ns tenso-plan (:require [tenso.murakumo :as m] [clojure.pprint :as pp]))
(println "cells:" (count m/cell-specs) " gates:" (count m/common-gates))

(println "\n-- no attestations --")
(let [p (m/cell-plan :transferrequest {})]
  (println "status" (:status p) "missing" (count (:missing-gates p))
           "effects" (count (:effects p))))

(println "\n-- all gates attested --")
(let [att (into {} (map (fn [g] [g true])) m/common-gates)
      p (m/cell-plan :transferrequest {:attestations att :request-id "req-demo-1"})]
  (println "status" (:status p) "effects" (count (:effects p)))
  (pp/pprint (first (:effects p))))

(println "\n-- one gate withheld --")
(let [att (into {} (map (fn [g] [g true])) (rest m/common-gates))
      p (m/cell-plan :transferrequest {:attestations att :request-id "req-demo-2"})]
  (println "status" (:status p) "missing" (pr-str (:missing-gates p))))
EOF
nbb --classpath src /tmp/tenso-plan.cljs
```

```
cells: 13  gates: 7

-- no attestations --
status :blocked missing 7 effects 0

-- all gates attested --
status :ready effects 1
{:op :mst/put-record,
 :actor "did:web:tenso.etzhayyim.com",
 :collection "com.etzhayyim.tenso.transferrequest",
 :rkey "req-demo-1",
 :record
 {:$type "com.etzhayyim.tenso.transferrequest",
  :actorBoundary "cljc-migration-scaffold",
  :legacyCell "com-etzhayyim-apps-tenso-transferRequest",
  :phase :event,
  :computedAt nil,
  :requestId "req-demo-1",
  :constitutionalStatus "attested-plan",
  :actorDid "did:web:tenso.etzhayyim.com",
  :scaffold true}}

-- one gate withheld --
status :blocked missing [:council-charter-attestation]
```

Three things to read off that output:

1. **The refusal names the gate it refused for.** Withholding
   `:council-charter-attestation` blocks and says so. A refusal that does not
   name its reason is not this library refusing — it is something else failing.
2. **`:collection` is `com.etzhayyim.tenso.transferrequest`**, which is *not*
   the `com.etzhayyim.apps.tenso.transferRequest` in `actor-manifest.jsonld`.
   The manifest's spelling survives in `:legacyCell`. Expected; see README.
3. **`:computedAt` is `nil`** unless you pass `:computed-at`. The planner does
   not read a clock — it is pure. Passing the timestamp is the caller's job.

## 5. Add or change a cell

Edit the `cell-specs` map in `src/tenso/murakumo.cljc`. The tests **introspect
`cell-specs`** rather than hardcoding cell names, so a new cell is covered by all
9 tests the moment you add it — nothing in `test/` needs editing.

Confirm the coverage is real rather than assumed, by breaking one thing on
purpose and checking that a *named* test catches it:

```bash
# make the gate stop refusing
python3 - <<'PY'
p='src/tenso/murakumo.cljc'; s=open(p).read()
s=s.replace("""(defn missing-gates
  [spec attestations]
  (->> (:required-gates spec)
       (remove #(boolean (gate-value attestations %)))
       vec))""","""(defn missing-gates
  [spec attestations]
  [])""")
open(p,'w').write(s)
PY
nbb --classpath "src:test" /tmp/tenso-run.cljs; echo "exit=$?"
git checkout src/tenso/murakumo.cljc     # put it back
```

```
FAIL in (missing-gates-computes-the-diff)
FAIL in (cell-plan-blocks-when-gates-missing)
exit=1
```

Two more that were checked the same way: stamping a different `:actor` in
`put-record-effect` fails `put-record-effect-shape`; flipping `:scaffold true`
to `false` fails `records-for-produces-one-record-per-collection`.

## What was **not** walked

Named so the next operator does not read silence as a pass.

- **`safe-rkey` has no test, and this quickstart does not add one.** Deleting its
  body (`(defn safe-rkey [s] (str s))`) leaves the suite green at 9/187. Its
  current behaviour is tabulated in the README; the whitespace-only case returns
  `"--"` rather than `"unknown"`, and nothing asserts either answer.
- **No effect was ever executed.** `:mst/put-record` is a plan. Nothing in this
  repo writes to a PDS, and no PDS was contacted while writing this document.
- **`did:web:tenso.etzhayyim.com` was resolved and it does not exist**
  (NXDOMAIN, 2026-09-02) — but nothing here was changed in response. See README.
- **The service described in `CLAUDE.md` was not exercised at all.** It is not in
  this repo. Its `etzhayyim deploy --smoke-url …` command targets a path
  (`60-apps/etzhayyim-project-tenso/…`) that does not exist in this tree.
- **`clojure -M:test` was run online.** A first run on an offline machine will
  fail at dependency resolution, not at the tests; that failure mode was not
  reproduced here.
