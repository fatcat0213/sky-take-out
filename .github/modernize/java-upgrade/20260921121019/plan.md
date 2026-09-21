# Upgrade Plan: sky-take-out (20260921121019)

- **Generated**: 2026-09-21 20:14:29 +08:00
- **HEAD Branch**: main
- **HEAD Commit ID**: N/A

## Available Tools

**JDKs**
- JDK 24.0.2: `D:\Program Files\Java\jdk-24\bin` (current runtime used for baseline)
- JDK 25.0.1: `C:\Program Files\Eclipse Adoptium\jdk-25.0.1.8-hotspot\bin` (target runtime used for upgrade and final validation)

**Build Tools**
- Maven 3.9.11: `D:\L\apache-maven-3.9.11\bin` (used for baseline, compile, and test runs)

## Guidelines

- Keep the change set focused on Java 25 runtime compatibility and the minimum dependency updates needed to compile and test cleanly.
- Preserve the current Spring Boot 2.7 application structure; do not attempt a Jakarta/Spring Boot major migration in this session.
- Run in auto-execution mode and proceed through the plan without pausing for additional user input.

> Note: You can add any specific guidelines or constraints for the upgrade process here if needed, bullet points are preferred.

## Options

- Working branch: appmod/java-upgrade-20260921121019
- Run tests before and after the upgrade: true

## Upgrade Goals

- Upgrade the application runtime and build target to Java 25 LTS.

## Technology Stack

| Technology/Dependency | Current | Min Compatible Version | Why Incompatible |
| --------------------- | ------- | ---------------------- | ---------------- |
| Java | 24.0.2 | 25 | User requested latest LTS runtime |
| Maven | 3.9.11 | 3.9.0 | Already compatible; no build tool upgrade required |
| Spring Boot starter parent | 2.7.3 | 2.7.18 | Old line; retained to avoid a larger Jakarta migration |
| Lombok | 1.18.20 | 1.18.48 | Current release does not generate members reliably on the active JDKs |
| maven-compiler-plugin | 3.10.1 | 3.11.0 | Older plugin version is below the recommended baseline for Java 25 |
| maven-surefire-plugin | 2.22.2 | 3.0.0 | Older plugin line is below the recommended baseline for modern JDKs |
| fastjson | 1.2.76 | N/A | Legacy dependency; likely a CVE remediation candidate |
| commons-lang | 2.6 | N/A | Legacy dependency; likely a CVE remediation candidate |
| jjwt | 0.9.1 | N/A | Legacy dependency; likely a CVE remediation candidate |
| jackson-databind | 2.9.2 | N/A | Legacy dependency pinned in `sky-pojo` and likely a CVE remediation candidate |

## Derived Upgrades

- Upgrade Lombok to a Java 25-compatible release because the current version fails to generate the logger/getter members used by the codebase.
- Keep the Spring Boot 2.7 line for this session to avoid a large `javax` to `jakarta` migration; validate whether it remains stable on Java 25 before considering a framework jump.
- Use the installed Maven 3.9.11 installation; no wrapper or Maven upgrade is required for the target JDK.

## Impact Analysis

### Dependency Changes

| File | Dependency | Current | Action | Target | Reason |
| ---- | ---------- | ------- | ------ | ------ | ------ |
| pom.xml | `java.version` | not set | add | `25` | Set the build/runtime target to Java 25 LTS |
| pom.xml | `org.projectlombok:lombok` | `1.18.20` | upgrade | `1.18.48` | Baseline compile fails because the current Lombok release does not generate the members used by `AliOssUtil` and `WeChatProperties` |

### Source Code Changes

| File | Location | Current | Required Change | Reason |
| ---- | -------- | ------- | --------------- | ------ |
| sky-common/src/main/java/com/sky/utils/AliOssUtil.java | line 64 | `log.info(...)` depends on Lombok-generated logger field | No source rewrite; restore the generated logger by upgrading Lombok | Baseline compile failure under the current Lombok version |
| sky-common/src/main/java/com/sky/properties/WeChatProperties.java | lines 10-21 | `@Data`-generated getters are consumed by `WeChatPayUtil` | No source rewrite; restore the generated getters by upgrading Lombok | Baseline compile failure under the current Lombok version |

### Configuration Changes

- None identified beyond the root build properties update in `pom.xml`.

### CI/CD Changes

- None identified.

### Risks & Warnings

- **Spring Boot 2.7.3 on Java 25**: the app is staying on an older Boot line to avoid a major Jakarta migration. **Mitigation**: validate the full compile/test cycle on Java 25 and only widen scope if the runtime proves unstable.
- **Legacy direct dependencies**: `fastjson 1.2.76`, `commons-lang 2.6`, `jjwt 0.9.1`, and `jackson-databind 2.9.2` are old and may trigger CVE remediation or compatibility work during the security phase. **Mitigation**: run the direct-dependency CVE scan and upgrade only the affected artifacts, then re-run compilation.
- **Inherited Maven plugins**: Boot 2.7.3 still manages older compiler/surefire plugin versions. **Mitigation**: if Java 25 build or test execution exposes plugin issues, pin newer plugin versions in the root build before broadening the framework migration.

## Upgrade Steps

- Step 1: Setup Environment
  - **Rationale**: Confirm the target JDK and Maven installation are already available before changing project files.
  - **Changes to Make**: None; this is an environment verification step only.
  - **Verification**: `appmod-list-jdks` and `appmod-list-mavens` output, with JDK 25 and Maven 3.9.11 available.

- Step 2: Setup Baseline
  - **Rationale**: Capture the pre-upgrade state on the current Java 24 runtime so the final result can be compared against it.
  - **Changes to Make**: None; run the current build and tests as-is.
  - **Verification**: `JAVA_HOME=D:\Program Files\Java\jdk-24` then `mvn clean compile test-compile -q && mvn clean test -q`
  - **Expected Result**: Baseline compile currently fails in `sky-common` because Lombok 1.18.20 does not generate `log`/getter members on the active JDK.

- Step 3: Upgrade Java 25 Runtime Target and Lombok
  - **Rationale**: This is the functional upgrade step that moves the build target to Java 25 and fixes the baseline compile break at the same time.
  - **Changes to Make**: Update the root `pom.xml` to set `java.version` to `25` and upgrade Lombok to `1.18.48` in dependency management.
  - **Verification**: `JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-25.0.1.8-hotspot` then `mvn clean test-compile -q`
  - **Expected Result**: The project compiles cleanly on Java 25 with Lombok-generated members restored.

- Step 4: CVE Validation and Dependency Remediation
  - **Rationale**: The project carries several old direct dependencies that are likely to surface in the security scan and should be addressed before final sign-off.
  - **Changes to Make**: Extract direct dependencies, scan them with `appmod-validate-cves-for-java`, then upgrade any vulnerable direct dependency versions in the relevant `pom.xml` files and fix any code-level API fallout.
  - **Verification**: Re-run `mvn clean test-compile -q` after the dependency updates, then re-scan to confirm the vulnerabilities are resolved or documented with no patch available.
  - **Expected Result**: All reported CVEs are fixed or explicitly documented with a technical reason they cannot be remediated.

- Step 5: Final Validation
  - **Rationale**: Prove the upgrade is complete on the target runtime and that the application still compiles and tests successfully.
  - **Changes to Make**: Resolve any remaining build/test failures, remove temporary workarounds, and ensure the final source state is clean.
  - **Verification**: `JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-25.0.1.8-hotspot` then `mvn clean compile test-compile -q && mvn clean test -q`
  - **Expected Result**: Full compile and test success on Java 25.
