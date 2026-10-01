# maven-310-strict-resolver

Probe for Maven 3.10.0: strict POM validation compliance,
explicit `<relativePath>` in multi-module parent references,
and diamond-dependency conflict resolution under Maven
Resolver 2.0.21.

## Feature exercised

Three Maven 3.10.0 behaviours are exercised in a single
three-module reactor:

1. **Strict POM validation** — all `pom.xml` files are
   well-formed, carry complete required coordinates, use the
   canonical XSD namespace, and avoid deprecated elements.
   Maven 3.10.0 enables stricter validation by default; a
   malformed manifest now fails the build before any resolution
   runs, causing Mend to fall back to partial-tree mode.

2. **Explicit `<relativePath>`** — each child module's
   `<parent>` block declares `<relativePath>../pom.xml</relativePath>`
   explicitly.  Maven 3.10.0 tightened relativePath validation:
   it now warns (or errors in strict mode) when the default
   `../pom.xml` path cannot be resolved on disk.  Declaring it
   explicitly is now the recommended practice and is what
   Mend's UA will encounter in real 3.10.0 projects.

3. **Diamond-dependency conflict resolution (Resolver 2.0.21)**
   — the dependency graph is:

   ```
   maven-310-strict-resolver (root, aggregator)
   ├── service-a
   │   ├── commons-collections4:4.4
   │   └── commons-text:1.11.0
   │       └── commons-lang3:3.14.0   ← diamond vertex (path 1)
   └── service-b
       ├── service-a                  ← inter-module dep
       └── commons-compress:1.26.0
   ```

   `commons-lang3` appears through path 1 (via `commons-text`).
   The root `<dependencyManagement>` pins `commons-lang3:3.14.0`.
   Maven Resolver 2.0.21 applies nearest-wins + constraint
   intersection: the root-level pin at depth 1 wins, resolving
   `commons-lang3` to `3.14.0`.

## Project layout

```
maven-310-strict-resolver-20261001-000000/
├── pom.xml              root aggregator (packaging=pom)
├── service-a/
│   └── pom.xml          jar module; deps on commons-collections4
│                        and commons-text
├── service-b/
│   └── pom.xml          jar module; deps on service-a and
│                        commons-compress
├── .whitesource         versioning pin (Bucket A)
├── README.md            this file
└── expected-tree.json   ground-truth dependency tree
```

## Expected dependency tree summary

Root module: `maven-310-strict-resolver` (aggregator, no
direct external deps).

`service-a` direct dependencies:
- `commons-collections4:4.4` (compile, registry)
- `commons-text:1.11.0` (compile, registry)

`service-a` transitive dependencies:
- `commons-lang3:3.14.0` (compile, via commons-text)

`service-b` direct dependencies:
- `service-a:1.0.0` (compile, local inter-module)
- `commons-compress:1.26.0` (compile, registry)

`service-b` transitives (inherited through service-a):
- `commons-collections4:4.4`
- `commons-text:1.11.0`
- `commons-lang3:3.14.0`

Diamond resolution: `commons-lang3` resolves to `3.14.0`
(nearest-wins from root `dependencyManagement` pin, honoured
by Maven Resolver 2.0.21 tighter constraint intersection).

## Mend config

**Bucket A** — `java-maven` has no dynamic version detection
from the manifest. `.whitesource` pins:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "maven": "3.10.0",
      "java": "17.0.10"
    }
  }
}
```

`configMode` is `AUTO` because no `whitesource.config` ships
with this probe. The versioning block ensures `install-tool`
provisions exactly Maven 3.10.0 and JDK 17.0.10, matching the
`pm_version_under_test` in `expected-tree.json`.

## Mend failure modes this probe targets

- UA falls back to flat direct-POM parsing when strict POM
  validation rejects a malformed manifest — confirmed absent
  here; all POMs are valid.
- Child modules missed when `<relativePath>` can't be resolved
  (old behaviour) — explicit paths prevent this.
- Duplicate `commons-lang3` entries (one per diamond path)
  instead of a single resolved entry.
- Wrong `commons-lang3` version (not 3.14.0) — indicates
  nearest-wins or constraint intersection was ignored.
- Inter-module dep `service-a` reported as a registry dep
  instead of a local dep.

## Resolver provenance

- Resolver file: `java-jvm.md`
- Upstream SHA: `3daf3de727b3e8f05588937e776294f8b1d72d09`
- Fetched at: `2026-10-01T08:30:52+00:00`
