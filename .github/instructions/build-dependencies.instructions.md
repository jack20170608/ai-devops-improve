---
applyTo: "**/pom.xml,**/*.gradle,**/*.gradle.kts,**/settings.gradle,**/settings.gradle.kts,**/gradle-wrapper.properties,**/maven-wrapper.properties"
---

# Build and dependency instructions

Human-readable sources:

- `10-Devops-SOP/01-ai-assisted-project-structure.md`
- `10-Devops-SOP/02-java-coding-standard.md`
- `10-Devops-SOP/03-secure-coding-standard.md`

## Toolchain

- Use the repository's Maven or Gradle Wrapper, never assume a globally installed version.
- Keep compiler, tests, static analysis, and CI on the same Java toolchain.
- Use a supported LTS JDK defined by the project; do not silently change the Java version.
- Verify Wrapper distribution checksums.
- Keep source and reporting encodings at UTF-8.
- Production builds must succeed from a clean environment without unpublished local artifacts.

## Dependencies

- Check whether the JDK or an existing dependency already provides the needed behavior.
- Explain why every new production dependency is necessary.
- Use fixed dependency and plugin versions.
- Do not use dynamic versions, Maven version ranges, snapshots, or unreviewed repositories for production builds.
- Use the existing BOM, dependency management, or version catalog.
- Keep compile, runtime, annotation processor, and test scopes correct.
- Do not allow test libraries into the production runtime.
- Review direct and transitive vulnerabilities, maintenance status, provenance, and license.
- Update lockfiles or dependency metadata consistently.

## Plugins and repositories

- Use approved artifact repositories.
- Require HTTPS and repository authentication through environment or settings, never literals in the build file.
- Pin build plugins and review code-executing plugins carefully.
- Do not add arbitrary plugin repositories to solve a transient resolution error.
- Do not disable checksum, signature, dependency, or license verification to make a build pass.

## Examples

Unsafe:

```xml
<version>[1.0,)</version>
```

Safe:

```xml
<version>${approved.library.version}</version>
```

Unsafe:

```groovy
implementation "com.example:library:+"
```

Safe:

```kotlin
implementation(libs.example.library)
```

## Completion

- Run dependency resolution from a clean environment.
- Run the repository's full verification command.
- Run dependency and license review for additions or upgrades.
- Confirm no credential, internal token, or production endpoint was added.
- Report new dependencies, their purpose, version source, and security impact.

