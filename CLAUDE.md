# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **Bnd/Gradle workspace whose sole purpose is to re-package third-party Maven
libraries as OSGi bundles** ("OSGi-ify" them) for the Gecko projects ecosystem. There is
almost no hand-written Java here — each top-level directory is a *wrapper project* that pulls
a Maven artifact (or set of artifacts) from a repository and re-emits it as a properly-manifested
OSGi bundle with correct `Export-Package`/`Import-Package` headers.

## Build & test commands

Uses the Gradle wrapper with the `biz.aQute.bnd.workspace` plugin (bnd 7.1.0, **Java 17**).

- `./gradlew build` — build every wrapper bundle (this is what CI runs).
- `./gradlew clean build release` — full build + publish to `cnf/release/` (main-branch flow).
- `./gradlew release` — snapshot release into `cnf/release/` (snapshot-branch flow).
- `./gradlew :<project.dir>:build` — build a single wrapper, e.g. `./gradlew :com.squareup.okhttp3:build`.
- `./gradlew codeCoverageReport` — aggregate Jacoco coverage from all subprojects.
- `./gradlew sonar` — run SonarCloud analysis (depends on the coverage report).

There is **no manual subproject registration**: `settings.gradle` only applies the bnd workspace
plugin, which auto-discovers every directory containing a `bnd.bnd` as a project. Adding a
directory with a `bnd.bnd` is enough to add a build target.

## Architecture

### The wrapper project pattern
Each library lives in a directory named after its bundle symbolic name (e.g.
`com.squareup.okhttp3`, `org.apache.lucene.core`). The only meaningful file is `bnd.bnd`, which:
- `-buildpath` — the Maven coordinate(s) to wrap, referenced by a version macro.
- `Export-Package` / `Import-Package` — the OSGi package wiring (often with
  `resolution:=optional` for platform-specific or absent deps, and `-split-package:=first`
  when multiple jars share a package).
- `-includeresource` — pulls extra files (sources into `OSGI-OPT/src`, `META-INF/services`,
  `META-INF/versions`, etc.) directly out of the resolved Maven jars via the `${repo;...}` macro.
- `Bundle-Version` — usually `${lib.version}.SNAPSHOT`.

The `.project`, `.classpath`, `.settings/` files are Eclipse/bndtools metadata — do not edit by hand.

### `cnf/` — the bnd workspace configuration
- `cnf/build.bnd` — global config shared by all projects: Java 17 source/target, group id
  (`org.geckoprojects.libraries`), baselining, the `-library` includes (`geckoDIMC`,
  `geckoJacoco`), shared version properties (e.g. `lucene-version`), and the numbered
  `-plugin.N.<Name>` declarations that register each **Maven repository** the workspace reads from.
- `cnf/central.mvn` — the Maven coordinate index for the shared "Central" repository.
- `cnf/library_specific/*.mvn` — per-library coordinate index files (one per repo plugin, e.g.
  `lucene.mvn`, `jena.mvn`, `okhttp.mvn`). Each line is a Maven GAV; `#` comments a line out.
  Bundles can only resolve a coordinate that is listed in one of these `.mvn` index files.

### Adding or upgrading a library
1. Add/update the Maven coordinate(s) (including the `:jar:sources` variants) in the appropriate
   `cnf/**/*.mvn` index. If it needs a brand-new repository, also add a `-plugin.N.<Name>` block
   in `cnf/build.bnd` pointing `index=` at a new `library_specific/<name>.mvn`.
2. Create a directory named for the bundle symbolic name with a `bnd.bnd` following the wrapper
   pattern above (copy an existing sibling as a template).
3. Version numbers are typically centralized as a property (in `cnf/build.bnd` or at the top of
   the `bnd.bnd`) and referenced via `${...}` macros.

### `org.gecko.libraries.workspace.library` (special project)
This does not wrap a library — it packages the *entire workspace* as a reusable bnd **library**
named `geckoLibraries`. Downstream bnd workspaces include this one library to get a Maven repo
plugin (`resources/template/workspace.bnd`) and an aggregated index
(`resources/template/geckoLibraries.maven`, generated from `mavendeps`) exposing every bundle
built here. `buildpath.bnd` derives its build path from `required.bndrun`'s `-runbundles`.

## Conventions

- Do **not** edit generated output (`generated/`, `build/`, `bin/`, `cnf/cache`, `cnf/release`) —
  all are git-ignored and produced by the build.
- All text files are stored **LF** (`.gitattributes` enforces `crlf=input`); `.jar` and other
  binaries are marked `binary`.
- A license header check runs in CI (`.licenserc.yaml`, skywalking-eyes). Run locally with:
  `docker run -it --rm -v $(pwd):/github/workspace ghcr.io/apache/skywalking-eyes/license-eye header check`
- Branch model (see `Jenkinsfile`): `main` → full release, `snapshot` → snapshot release. Both
  are protected; changes land via PR.
