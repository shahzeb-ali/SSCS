# Task 02 | Is the Service Affected?

Slide source: one `##` heading = one slide. Screenshots are in `screenshots/`.
"Say" = speaker notes (not on the slide).

---

## The task

The university's security team will scan our SBOMs. We scan them first, together with the vendor library
`audit` 2.3.0.

- **Decide for every finding whether the service is affected**
- **Q1:** Back each verdict with evidence from the code of the service and its libraries
- **Q2:** Record the verdicts as VEX: the next scan reports only the open findings

**Say:** "Affected" doesn't mean "the vulnerable library is in the jar". It means an attacker can actually trigger the
vulnerable code in our service.

---

## Scan: findings per SBOM

```bash
grype sbom:build-maven.cdx.json --by-cve -o table=grype-maven.txt -o json=grype-maven.json   # build SBOM (Task 01)
grype sbom:jar.cdx.json                  -o table=grype-jar.txt                              # jar SBOM (Task 01)
grype dir:$R/libs/audit          --by-cve -o table=grype-audit.txt                           # vendor library audit 2.3.0
```

![scans](screenshots/T2-1-scans.png)

| Scanned | Findings |
|---|---|
| build SBOM | **4** in guava 20.0: CVE-2018-10237 (Medium), CVE-2020-8908 (Low), CVE-2023-2976 (Medium), **CVE-2026-102554 (High)** |
| jar SBOM | 0: blind to the relocated Guava 20.0 (Task 01) |
| vendor library `audit` 2.3.0 | the **same 4** |

- `--by-cve`: report CVE IDs instead of GitHub advisory IDs (GHSA), so the IDs match the vendor's advisory and our VEX
- All 4 findings come from **one** component (Guava 20.0) that enters through **one** door: `audit`
- (npm SBOM, vite and its tree: 0 findings)

**Say:** We scan the build SBOM because it is the one we deliver. The vendor scan shows where the findings come from:
audit brings Guava 20.0, and with it all four CVEs.

---

## Q1 · The vendor's claim vs. the scan (3 ≠ 4)

```bash
cat $R/libs/audit/SECURITY.md
grype dir:$R/libs/audit -q --by-cve
```

![vendor vs scan](screenshots/T2-2-vendor-vs-scan.png)

| CVE | Severity | In vendor's SECURITY.md? | Vendor verdict |
|---|---|---|---|
| CVE-2018-10237 | Medium | ✅ | not affected |
| CVE-2020-8908 | Low | ✅ | not affected |
| CVE-2023-2976 | Medium | ✅ | not affected |
| **CVE-2026-102554** | **High** | ❌ | **none** |

- **Outdated:** the statement predates the newest, most severe CVE
- **A claim, not evidence:** "calls none of the APIs" must be verified by us
- **Only covers audit:** it can't say whether *our service* passes attacker data into Guava
- "No plans to move to a newer Guava" → every future Guava 20.0 CVE is ours to assess

**Say:** We reuse the vendor's idea (check which Guava APIs are called), but verify all four ourselves, especially the
one they don't mention.

---

## Q1 · What each vulnerability needs (advisory → vulnerable API)

| CVE | Severity | Vulnerable API (in our jar under `com.example.platform.audit.shaded.guava`) | Attacker needs | Impact |
|---|---|---|---|---|
| CVE-2018-10237 | Medium | `AtomicDoubleArray`, `CompoundOrdering`: unbounded allocation on deserialization | the service deserializes their bytes | DoS |
| CVE-2020-8908 | Low | `io.Files.createTempDir()`: world-readable temp dir | local account + the service calls it | info disclosure |
| CVE-2023-2976 | Medium | `io.FileBackedOutputStream`: silently spills data to a world-readable temp file | local account + the service uses it | info disclosure |
| CVE-2026-102554 | High | `CompactHashMap`, `CompactHashSet`, `MapMakerInternalMap` `readObject()`: pre-allocates the element count read from the stream | the service deserializes their bytes | DoS |

**Two questions decide all four verdicts:**
1. Does any code (ours or audit's) **deserialize** with `ObjectInputStream`? → CVE-2018-10237, CVE-2026-102554
2. Does any code **call** `createTempDir()` or `FileBackedOutputStream`? → CVE-2020-8908, CVE-2023-2976

**Say:** A CVE is about an API, not a jar. Before we can say "not affected", we need to know which API and what the
attacker must be able to do.

---

## Q1 · Evidence 1: AuditLog's only Guava 20.0 call (shipped bytecode)

Only AuditLog can reach Guava 20.0: shading renamed it to `com/example/platform/audit/shaded/guava/…` and rewrote only
audit's code to that name. So we list every method call in the AuditLog **we ship**:

```bash
cd /tmp/jar && unzip -o -q $JAR                                    # the delivered jar
javap -c -p com/example/platform/audit/AuditLog.class | grep invoke   # -c code, -p incl. private; invoke = method call
```

![javap AuditLog](screenshots/T2-4-javap-auditlog.png)

| Call | Guava 20.0? |
|---|---|
| `…/audit/shaded/guava/base/Objects.firstNonNull` | ✅ **the only one**: a null check (`user` or `"anonymous"`) |
| `Object.<init>`, `Instant.now`, `String.valueOf`, `makeConcatWithConstants`, `println` | no: plain Java |

- None of the vulnerable APIs, no `ObjectInputStream`
- Source (vendor's claim) = bytecode (what we ship) → the vendor's statement is now **verified**, for all 4 CVEs

**What a vulnerable result would look like** (hypothetical: AuditLog writing its log to a temp folder):
```java
File dir = Files.createTempDir();
```
```
 3: invokestatic  // Method com/example/platform/audit/shaded/guava/base/Objects.firstNonNull…
10: invokestatic  // Method com/example/platform/audit/shaded/guava/io/Files.createTempDir:()Ljava/io/File;   ⚠️
```
→ CVE-2020-8908's vulnerable method is in the execute path → **affected** (or analyse further: is sensitive data
written there, are there other local users?).

**Say:** We trust the bytecode in the jar, not the source: shading rewrote it. One extra line in this output would flip
a verdict.

---

## Q1 · Evidence 2: vulnerable code present, but never called

**Present?** List the vulnerable classes in the delivered jar:
```bash
unzip -l $JAR | grep -E 'shaded/guava/(io/Files|io/FileBackedOutputStream|util/concurrent/AtomicDoubleArray|collect/CompactHashMap|collect/MapMakerInternalMap)\.class'
```
```
22085  2016-10-28 16:56  com/example/platform/audit/shaded/guava/collect/MapMakerInternalMap.class
 4333  2016-10-28 16:56  com/example/platform/audit/shaded/guava/io/FileBackedOutputStream.class
15880  2016-10-28 16:56  com/example/platform/audit/shaded/guava/io/Files.class
 3902  2016-10-28 16:56  com/example/platform/audit/shaded/guava/util/concurrent/AtomicDoubleArray.class
```

**Called?** Disassemble every class outside Guava (server, rooms, audit) and search for calls into them or into
`ObjectInputStream`:
```bash
for c in $(find de com/example -name '*.class' -not -path '*/shaded/*'); do
  javap -c -p "$c" | grep -E 'invoke.*(io/Files\.createTempDir|FileBackedOutputStream|AtomicDoubleArray|CompactHash|MapMaker|ObjectInputStream)'
done
```
```
(no output: never called)
```

| | Present in jar | Called | → VEX justification |
|---|---|---|---|
| Vulnerable Guava 20.0 classes | ✅ yes (2016 build) | ❌ no | **`vulnerable_code_not_in_execute_path`** |
| Deserialization (`ObjectInputStream`) | — | ❌ nowhere | attacker can't deliver serialized bytes |

- Present → not `component_not_present`; the scanner is right that Guava 20.0 ships
- (`CompactHashMap` is not in this Guava 20.0 copy; `MapMakerInternalMap` is, so CVE-2026-102554 applies in principle)
- Limitation: static analysis misses reflection / dependency injection → runtime check next

**Say:** The scanner sees the code is there. We show it never runs. That gap is exactly what VEX records.

---

## Q1 · Evidence 3: runtime check (which classes really load)

Start the real server, use every endpoint, let the JVM log every class it loads:
```bash
java -Xlog:class+load:file=/tmp/classload.txt -jar $JAR > /dev/null &
curl -s localhost:8080/rooms
curl -s -X POST -H "X-User: alice" -d "A101" localhost:8080/bookings
curl -s -X POST -d "B204" localhost:8080/bookings
kill %1
grep -o 'com.example.platform.audit.shaded.guava[^ ]*' /tmp/classload.txt
```

![runtime class loading](screenshots/T2-6-runtime-classload.png)

| Loaded Guava 20.0 class | Why |
|---|---|
| `base.Objects` | AuditLog's `Objects.firstNonNull` |
| `base.ExtraObjectsMethodsForWeb` | parent class of `Objects` |
| `base.MoreObjects`, `base.Preconditions` | `firstNonNull` delegates here inside Guava 20.0 |

- **Not loaded:** `io.Files`, `io.FileBackedOutputStream`, `AtomicDoubleArray`, `MapMakerInternalMap`
- Booking without user → `missing X-User`: our server rejects it before AuditLog

| Check | Type | Result |
|---|---|---|
| javap AuditLog | static | only `Objects.firstNonNull` |
| javap all classes | static | no vulnerable calls, no deserialization |
| class loading | **dynamic** | only `base.*` loaded |

**Static and dynamic agree.** Limitation: dynamic covers only exercised paths; static covers the rest.

**Say:** Static analysis can miss reflection; dynamic analysis can miss untested paths. Together they cover each
other's blind spots.

---

## Q1 · Verdicts

| CVE | Severity | Vulnerable API present? | Reachable? | Verdict | VEX justification |
|---|---|---|---|---|---|
| CVE-2018-10237 | Medium | ✅ `AtomicDoubleArray` | ❌ no deserialization anywhere | **not affected** | `vulnerable_code_not_in_execute_path` |
| CVE-2020-8908 | Low | ✅ `io.Files` | ❌ `createTempDir` never called | **not affected** | `vulnerable_code_not_in_execute_path` |
| CVE-2023-2976 | Medium | ✅ `FileBackedOutputStream` | ❌ never used | **not affected** | `vulnerable_code_not_in_execute_path` |
| CVE-2026-102554 | High | ✅ `MapMakerInternalMap` | ❌ no deserialization anywhere | **not affected** | `vulnerable_code_not_in_execute_path` |

**Say:** Not `component_not_present`: the code ships. Not "the vendor said so": we checked the shipped bytecode and the
running service. The verdict holds only as long as audit keeps calling `firstNonNull` alone and nothing deserializes
input.

---

## Q2 · The VEX document (OpenVEX)

```bash
jq '.statements[] | {cve: .vulnerability.name, status, justification}' $V
```

One statement per CVE (screenshot: bottom half of the rescan slide) (`task02/roombooking.vex.json`):

| Field | Value | Meaning |
|---|---|---|
| `vulnerability` | CVE ID + GHSA alias | which finding |
| `products` | `pkg:maven/de.hbrs.sscs/roombooking@1.0.0?type=pom` with subcomponent `pkg:maven/com.google.guava/guava@20.0?type=jar` | in which product, which package |
| `status` | `not_affected` | our verdict |
| `justification` | `vulnerable_code_not_in_execute_path` | why (standard OpenVEX value) |
| `impact_statement` | e.g. "Files.createTempDir() is never called …" | our evidence in one sentence |
| `author`, `timestamp` | Room booking service team, 2026-11-17 | who stands behind it, when |

- PURLs copied **exactly** from the build SBOM (incl. `?type=jar`): grype applies a statement only if CVE, product and
  package match
- Second product entry = the Guava PURL itself: for SBOM scans grype can match the vulnerable package directly

**Say:** VEX turns our analysis into a machine-readable claim. It doesn't hide the finding; it states our verdict and
who is accountable for it.

---

## Q2 · Rescan: only open findings remain

```bash
grype sbom:build-maven.cdx.json --by-cve --vex $V                    # what the university sees
grype sbom:build-maven.cdx.json --by-cve --vex $V --show-suppressed  # audit view
```

![rescan with vex](screenshots/T2-8-rescan-vex.png)

| Scan | Result |
|---|---|
| without VEX | 4 findings |
| with VEX | **No vulnerabilities found** (`4 ignored`) |
| with VEX, `--show-suppressed` | the same 4, each marked `(suppressed by VEX)` |

- The next scan reports **only open findings**: currently none
- Nothing is deleted: suppressed findings stay visible and traceable to our VEX statement
- A **new** Guava 20.0 CVE would appear as an open finding: no statement covers it

**Say:** We deliver SBOM plus VEX together. The university's scan then shows only what still needs attention. A VEX
is a snapshot as well: when audit or our code changes, we re-check and update it.

---

## Reflect · If audit called another Guava API, which verdict changes?

| audit (or our code) starts to … | Verdict that flips to **affected** | Others |
|---|---|---|
| call `Files.createTempDir()` | CVE-2020-8908 | stay not affected |
| use `FileBackedOutputStream` | CVE-2023-2976 | stay not affected |
| deserialize input (`ObjectInputStream.readObject`) | **both** CVE-2018-10237 and CVE-2026-102554 | stay not affected |
| call another harmless API (e.g. `Strings.isNullOrEmpty`) | none | all stay not affected |

**Deciding input:** audit's **shipped bytecode** (javap on the jar), not the vendor's source or SECURITY.md, plus **our
data flow** (does untrusted input reach that call?).

- Each verdict is tied to one API, so one new call line in javap flips exactly that CVE
- → Re-run the bytecode check and update the VEX whenever `audit` (or our code) changes

**Say:** Our VEX is only as current as the bytecode we checked. That's why the evidence comes from the shipped jar.

---

## Reflect · Through which station did the findings enter?

```bash
cd $R/app && mvn -B dependency:tree -pl audit-shaded,server 2>&1 | grep -E -- '--- dependency|de.hbrs|com.example|guava'
```

```
de.hbrs.sscs:audit-shaded:jar:1.0.0
\- com.example.platform:audit:jar:2.3.0:compile
   \- com.google.guava:guava:jar:20.0:compile                ← entry: Dependency (V)

de.hbrs.sscs:server:jar:1.0.0
+- com.example.booking:rooms:jar:2.0.0:compile
|  \- com.google.guava:guava:jar:33.7.2-jre:compile
|     +- com.google.guava:failureaccess:jar:1.0.3:compile
|     +- com.google.guava:listenablefuture:jar:9999.0-empty-to-avoid-conflict-with-guava:compile
\- de.hbrs.sscs:audit-shaded:jar:1.0.0:compile
   \- com.example.platform:audit:jar:2.3.0:compile           ← guava 20.0 not listed: hidden by the build
```

| Station (course map, Ch1 sl. 40) | Role |
|---|---|
| **Dependency (V)**: "the build includes a known vulnerable dependency" | all 4 CVEs enter **transitively** through the vendor library `audit` 2.3.0, not by our choice |
| **Build** | shading relocates Guava 20.0 and Maven's version mediation (managed 33.7.2-jre) drops it from server's tree → invisible in jar SBOM and server's tree (Ch4 sl. 59) |

**Say:** The vulnerability came in through a dependency; the build made it invisible. Defence at V is SCA, which only
works if the SBOM sees through the build: the aggregate build SBOM did.
