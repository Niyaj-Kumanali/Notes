# Maven

## Overview

- **Definition** — Apache Maven is a build automation and dependency management tool for Java projects that uses a declarative XML-based Project Object Model (POM).
- **Why It Exists** — Eliminates repetitive, imperative build scripts by providing a convention-over-configuration lifecycle, centralized dependency resolution through repositories, and reproducible builds across environments.
- **Historical Context** — Created in 2004 as an Apache project to supersede Ant's procedural XML scripts, introducing a standardized build lifecycle, transitive dependency management, and the Maven Central Repository that became the industry standard.
- **Key Concepts** — **POM** (project descriptor XML at pom.xml), **groupId** (organization identifier), **artifactId** (module name), **version** (release identifier), **packaging** (artifact type), **Lifecycle** (ordered build phases), **Phase** (lifecycle step), **Goal** (build task bound to a phase), **Plugin** (container of goals), **Dependency** (external library declaration), **Scope** (dependency visibility), **Transitive dependency** (inherited library), **Exclusion** (suppressed transitive dep), **Reactor** (multi-module coordination).

## Core Concepts

- pom.xml is the fundamental configuration file containing project coordinates (groupId, artifactId, version), packaging type, dependencies, plugins, build profiles, and distribution settings.
- groupId uses a reverse domain name (e.g., com.company.app), artifactId is the module or project name, version follows semantic versioning with optional -SNAPSHOT suffix for development builds. These three coordinates uniquely identify every artifact in the Maven repository.
- The default build lifecycle consists of three built-in lifecycles: default (main build), clean (artifact cleanup), and site (documentation generation). The default lifecycle phases execute in strict order: validate, compile, test, package, verify, install, deploy. Each phase is executed sequentially — running deploy runs all preceding phases first.
- Phases are build steps, goals are concrete tasks, and plugins bundle one or more goals bound to lifecycle phases. For example, the maven-compiler-plugin declares the compile goal bound to the compile phase, and maven-surefire-plugin binds the test goal to the test phase.
- Common plugins include maven-compiler-plugin (compile Java sources at compile phase), maven-surefire-plugin (unit test execution at test phase), maven-failsafe-plugin (integration test execution at verify phase), maven-shade-plugin (uber-JAR with relocated dependencies at package phase), and maven-assembly-plugin (custom distribution archives at package phase).
- Dependency scopes control classpath visibility and transitivity:

  | Scope | Compile | Test | Runtime | Transitive |
  |-------|---------|------|---------|------------|
  | compile | Yes | Yes | Yes | Yes |
  | provided | Yes | Yes | No | No |
  | runtime | No | Yes | Yes | Yes |
  | test | No | Yes | No | No |
  | system | Yes | Yes | No | No |
  | import | No | No | No | No |

- Transitive dependencies are automatically resolved from direct dependencies' POM files. Maven resolves the dependency tree from each direct dependency's declared dependencies. Use `mvn dependency:tree` to inspect the full tree.
- Exclusions suppress unwanted transitive dependencies by declaring `<exclusions><exclusion>` inside a dependency block. This is essential when two dependencies pull in conflicting versions of the same library.
- The Maven Wrapper (mvnw) is a script that downloads and runs a pinned Maven version. It ensures all developers and CI systems use the identical Maven version without manual installation. Generate it with `mvn -N wrapper:wrapper`.
- settings.xml is a user-level or global configuration file that defines mirrors (redirect repository requests to alternative URLs like a corporate Nexus), servers (store authentication credentials for secured repositories), and profiles (define environment-specific properties and activation triggers like JDK version or OS family).
- Multi-module projects use a parent POM with a `<modules>` block listing each submodule path. The reactor build resolves inter-module dependencies and determines the optimal build order. Parent POMs typically contain shared configuration via `<dependencyManagement>` and `<pluginManagement>`, while submodules define only their specific dependencies and plugins.

## Common Mistakes

- **Forgetting to exclude transitive dependencies**
  - A dependency pulls in unwanted transitive libraries that introduce classpath conflicts, version mismatches, or runtime errors such as NoSuchMethodError.
  - **Why it looks correct:** The build succeeds and the application runs during initial development because the conflicting code paths are never exercised early on.
  - Always inspect the dependency tree with `mvn dependency:tree` before adding new dependencies. Exclude conflicting transitive libraries explicitly using `<exclusions>` inside the dependency declaration.

- **Misuse of provided scope**
  - Libraries are declared as provided because they exist in the local JDK or container, when they are actually needed by downstream consumers at compile time.
  - **Why it looks correct:** The project compiles and all tests pass locally since the library is present in the development environment.
  - Use provided only when the runtime environment genuinely supplies the library (e.g., Servlet API in a Tomcat container). If consumers need the library for compilation, use compile scope. If it is only needed at runtime, use runtime scope.

- **Failing to pin plugin versions**
  - Plugin versions are left unbound, allowing Maven to resolve them from the super POM defaults or the local repository cache, which can differ across Maven versions or environments.
  - **Why it looks correct:** The build works on the developer's machine, so explicit versioning appears unnecessary.
  - Always specify explicit versions on every plugin declaration. Use `<pluginManagement>` in the parent POM to centralize and enforce plugin versions across all modules.

## Real-World Scenarios

### Building a multi-module microservices project

- A financial services company structures its platform as a parent POM with modules for common-lib, domain-model, service-gateway, transaction-engine, and integration-tests. The parent POM declares all dependency versions in `<dependencyManagement>` and all plugin versions in `<pluginManagement>`. The reactor build ensures correct build order: common-lib builds first, then domain-model, then the services that depend on them. Integration-tests builds last because it depends on all modules. A single `mvn clean install` command builds the entire project with consistent versions.

### Setting up release management with Nexus

- An organization uses a private Nexus repository for internal artifact storage. Developers publish snapshot builds to the Nexus snapshots repository via `mvn deploy`, and the release manager publishes release versions to the Nexus releases repository using the maven-release-plugin. settings.xml defines server credentials for both repositories, and a corporate mirror in settings.xml redirects all Maven Central requests through Nexus for caching and security scanning.

## Use Cases

Reach for Maven when you need a predictable, declarative build system that enforces conventions across a team or organization.

- **Multi-module project coordination** — Maven's reactor build orders modules by dependency, compiling, testing, and packaging in a single command.
  - Parent POM with `<dependencyManagement>` centralizes versions across dozens of modules.
  - **Avoid when:** you need dynamic or conditional build logic; Gradle's DSL is more flexible.

- **Standardized builds across teams** — Convention-over-configuration ensures every developer and CI system builds identically with `mvn clean install`.
  - No scripting decisions — the lifecycle (validate → compile → test → package → verify → install → deploy) is fixed and well-understood.
  - **Avoid when:** your build has unusual steps that don't fit the standard lifecycle phases.

- **Publishing artifacts to a repository** — `mvn deploy` pushes versioned artifacts to Nexus/Artifactory with full metadata, enabling downstream consumption.
  - Essential for library distribution across microservices or external consumers.
  - **Avoid when:** you only need local builds with no artifact sharing.

- **Reproducible CI/CD builds** — POM files and the Maven Wrapper (mvnw) pin exact plugin versions and Maven distributions for deterministic builds.
  - Critical for audit compliance and debugging production issues.
  - **Avoid when:** the team prioritizes fast, incremental builds over strict determinism.

- **Microservice project scaffolding** — Multi-module POMs structure shared libraries, domain models, and service modules under one parent, enforcing consistent dependency versions.
  - **Avoid when:** the project is a single-module application — Maven's boilerplate may be unnecessary overhead.

---

## Scenario-Based Questions

**Q: Your multi-module build fails at runtime with a "Duplicate class found" error, but compilation and tests pass without issues. How do you diagnose and fix the problem?**

- Run `mvn dependency:tree` on each module to identify the conflicting library appearing on multiple dependency paths. Use the dependency tree output to determine which direct dependency introduces the unwanted transitive library. Add an `<exclusion>` to that dependency's declaration to suppress the duplicate. If the conflict is widespread, use `<dependencyManagement>` in the parent POM to force a single version across all modules.

- **Interview follow-up:** How would you handle this if the duplicate class comes from an uber-JAR created by the maven-shade-plugin that shades its own copy of a library already on the classpath?

**Q: A teammate cannot build the project because their corporate proxy blocks access to Maven Central. How do you configure Maven to work in this environment?**

- Add a mirror entry in settings.xml pointing to the internal corporate repository (Nexus or Artifactory) that proxies Maven Central. Also configure any required HTTP proxy settings in settings.xml using `<proxies><proxy>`. Ensure the mirror is configured as a `<mirrorOf>central</mirrorOf>` to redirect all Central requests.

- **Interview follow-up:** How would you ensure that developers outside the corporate network can still build the project without modifying configuration files?


**Q: You need to add a new library dependency that is only available from a private repository hosted by a vendor. How do you configure Maven to resolve this dependency?**

- Add the private repository URL in the `<repositories>` section of pom.xml or in settings.xml with a unique ID. If the repository requires authentication, add server credentials in settings.xml under `<servers>`, referencing the same repository ID. Use `<repository>` and optionally `<snapshotRepository>` if the vendor publishes snapshots. Verify resolution with `mvn dependency:resolve`.

- **Interview follow-up:** How would you configure this so that only specific modules in a multi-module project use the vendor repository?


**Q: Your CI pipeline runs mvn clean test, but integration tests are not being executed even though you configured the maven-failsafe-plugin. What is the likely cause?**

- The maven-failsafe-plugin binds to the `verify` phase, not the `test` phase. Running `mvn clean test` stops before the verify phase, so failsafe goals never execute. Change the CI command to `mvn clean verify` instead. The verify phase runs after package and includes both surefire (unit tests at test phase) and failsafe (integration tests at verify phase).

- **Interview follow-up:** How would you configure the build so that integration tests run during the test phase for local development but still follow the standard lifecycle in CI?


**Q: A developer accidentally committed a dependency with a -SNAPSHOT version to a release branch. How do you enforce that release builds never contain SNAPSHOT dependencies?**

- Use the maven-enforcer-plugin with the `requireReleaseDeps` rule or the `bannedDependencies` rule to fail the build if any SNAPSHOT dependency is present. Bind the enforcer plugin to the `validate` phase so it fails early. Alternatively, use the Nexus Staging plugin or the maven-release-plugin, which automatically checks for SNAPSHOT dependencies before performing a release.

- **Interview follow-up:** What if a transitive dependency pulls in a SNAPSHOT — how do you override it to a release version?


**Q: Your multi-module project takes 15 minutes to build, but each module only changed slightly. What Maven features can you leverage to speed this up?**

- Use `mvn -T <threads>` for parallel module building (e.g., `mvn -T 4` for 4 threads). Enable `-o` (offline mode) to skip remote metadata lookups if all dependencies are already cached. Use `mvn -pl <module-list> -am` to build only changed modules and their dependencies. Consider using a build cache tool like mvnd (Maven Daemon) which uses a long-lived JVM. Ensure `-U` is not used unnecessarily as it forces repository updates.

- **Interview follow-up:** How does the Maven reactor determine the build order for multi-module projects, and how can you enforce a specific order?


**Q: You push a pom.xml change to a shared repository, and suddenly downstream consumers report that their builds are failing because a new required dependency appeared. What happened?**

- You likely changed a dependency from `<scope>test</scope>` or `<scope>provided</scope>` to `compile` scope without adding an exclusion. The compile-scope dependency now propagates transitively to consumers, forcing them to have it on their classpath. Alternatively, you removed an `<exclusion>` from a dependency. Inspect the diff of pom.xml and revert the unintended scope change, then use `mvn dependency:tree` to verify the transitive impact before committing.

- **Interview follow-up:** How would you introduce a new compile-scope dependency to a common library module without affecting consumers?


**Q: Your build fails on a CI server with "Could not resolve dependency" but works fine on your local machine. All dependencies are from Maven Central. What could be wrong?**

- The CI server likely cannot reach Maven Central due to network restrictions or proxy configuration. Check if the CI environment requires a corporate mirror configured in settings.xml or CI-managed settings. The CI machine may also be using an older cached metadata — use `mvn -U` to force update snapshots. Additionally, verify the Maven version on CI matches your local version, as newer Maven may enforce stricter checksum validation.

- **Interview follow-up:** How would you set up a local repository mirror on the CI server to improve build reliability and speed?


**Q: Your team wants to ensure every commit produces a reproducible build — the exact same artifact bytes for the same source code. How do you configure Maven for this?**

- Pin all plugin versions explicitly in `<pluginManagement>`. Set `maven.compiler.release` (or `source`/`target`) to fix the Java version. Use `<timestamp>false</timestamp>` in the maven-jar-plugin to disable timestamps in the manifest. Disable the maven-jar-plugin's digest algorithm randomization. Use the reproducible-build-maven-plugin which normalizes archive entries. Finally, pin the same Maven wrapper version across all environments.

- **Interview follow-up:** What are the differences between Maven's approach to reproducible builds and Gradle's approach?


**Q: You need to deploy different artifact versions for different environments (dev, staging, production) without modifying the pom.xml each time. How do you achieve this?**

- Use Maven profiles in pom.xml activated by properties, environment variables, or JDK version. Each profile can override properties like `build.version` or the `<distributionManagement>` URL. Pass the active profile via `mvn deploy -P production`. For more complex cases, use `<activation><property>` blocks in settings.xml or the `-D` flag to dynamically set version qualifiers.

- **Interview follow-up:** What are the risks of using too many profiles, and how would you simplify a project with dozens of profile combinations?

## Interview Questions

- **Explain the Maven build lifecycle and how phases, goals, and plugins relate to each other.**
  - The Maven build lifecycle defines a sequence of phases (validate, compile, test, package, verify, install, deploy) that execute in strict order. Each phase is a step in the build process. Goals are concrete build tasks provided by plugins. Plugins declare which goals bind to which lifecycle phases. When Maven executes a phase, it runs all goals bound to that phase and all goals bound to every preceding phase. For example, maven-compiler-plugin binds its compile goal to the compile phase, and maven-surefire-plugin binds its test goal to the test phase.

- **What is the difference between provided and compile scope?**
  - compile scope makes the dependency available on all classpaths (compile, test, runtime) and propagates transitively to consumers. provided scope makes the dependency available at compile time and test time but excludes it from the runtime classpath and does not propagate transitively. Use provided when the runtime environment (e.g., a servlet container) supplies the library.

- **How do you manage dependency versions across a multi-module Maven project?**
  - Use a parent POM with `<dependencyManagement>` to centralize version declarations. Submodules declare dependencies in their own `<dependencies>` without version tags — they inherit versions from the parent's `<dependencyManagement>`. For plugins, use `<pluginManagement>` in the parent POM. The parent POM also defines `<modules>` so a single command builds the entire project as a reactor.

- **What is a BOM (Bill of Materials) and when would you use it?**
  - A BOM is a specialized POM with only a `<dependencyManagement>` section and no dependencies of its own. It is imported using `<scope>import</scope>` and `<type>pom</type>` in the importing POM's `<dependencyManagement>`. BOMs centralize dependency versions for an entire library ecosystem, such as the Spring Boot BOM or the Jackson BOM, so consumers can declare dependencies without specifying versions.

- **How does Maven resolve dependency conflicts and which version wins?**
  - Maven uses a nearest-definition strategy: the dependency closest to the root in the dependency tree wins. If the same dependency appears at the same depth, the first declaration wins. This can lead to surprising results when transitive dependencies introduce different versions. Use `<dependencyManagement>` to take explicit control and `mvn dependency:tree` to inspect the resolved versions.

- **What is the purpose of the maven-surefire-plugin and how do you customize test execution?**
  - Maven-surefire-plugin runs unit tests during the test phase. It supports JUnit, TestNG, and other test frameworks. Customize it with `<includes>` and `<excludes>` patterns, `<forkCount>` for parallel execution, `<argLine>` for JVM arguments, and `<reportsDirectory>` for output location. It automatically picks up classes matching `*Test.java`, `Test*.java`, and `*TestCase.java`.

- **Explain the difference between maven-shade-plugin and maven-assembly-plugin.**
  - The maven-shade-plugin creates an uber-JAR by merging classes from all dependencies and provides relocation (renaming packages) to avoid classpath conflicts. The maven-assembly-plugin creates distribution archives (ZIP, TAR, JAR) with any desired format, including dependencies as separate JARs in a lib folder. Shade is preferred for executable JARs; assembly is preferred for full distribution packages.

- **How do you handle optional dependencies in Maven?**
  - Declare a dependency with `<optional>true</optional>` to indicate that it is not required by consumers. Optional dependencies are not propagated transitively — consumers must explicitly declare them if needed. This is useful for libraries that offer multiple features (e.g., a logging library that supports Logback and Log4j) where consumers choose one implementation.

- **What is the difference between the clean lifecycle and the default lifecycle?**
  - The clean lifecycle has three phases: pre-clean, clean, and post-clean. The clean phase deletes the build output directory (typically `target/`). The default lifecycle handles compilation, testing, packaging, and deployment. They are separate lifecycles, so `mvn clean install` runs clean's clean phase first, then the default lifecycle up to install.

- **How does the Maven dependency mediator work for version conflict resolution?**
  - Maven uses a nearest-wins strategy: it walks the dependency tree from the root and selects the first occurrence of each groupId:artifactId. If a dependency appears at multiple depths, the shallowest wins. At equal depth, the first declared dependency wins. This is why `<dependencyManagement>` in the parent POM is critical — it inserts entries at the root level to force specific versions.

- **What is the purpose of the maven-enforcer-plugin and what are common rules?**
  - The maven-enforcer-plugin enforces build rules during the validate phase. Common built-in rules include `requireJavaVersion`, `requireMavenVersion`, `requireReleaseDeps`, `bannedDependencies`, and `dependencyConvergence`. Custom rules can be written by extending `AbstractEnforcerRule`. Failures stop the build early, preventing deployment of non-conforming artifacts.

- **How do you exclude unwanted files from a JAR built by Maven?**
  - Configure the maven-jar-plugin with `<excludes>` inside `<configuration>` to exclude specific file patterns (e.g., `**/*.properties` for environment-specific configs). For more control, use the maven-resources-plugin with `<excludes>` to filter resources before they reach the build directory. Alternatively, configure the compiler plugin to exclude specific source files.

- **Explain the concept of a Maven archetype and when you would create one.**
  - A Maven archetype is a project template that generates a working project structure with preconfigured pom.xml, source directories, and sample code. Use the `mvn archetype:generate` goal. Teams create custom archetypes to standardize project layouts, enforce corporate conventions (logging, testing, reporting), and bootstrap microservices with consistent configurations.

- **What is the difference between `mvn install` and `mvn deploy`?**
  - `mvn install` copies the built artifact (JAR, WAR, etc.) into the local Maven repository (`~/.m2/repository`), making it available for local projects that depend on it. `mvn deploy` additionally uploads the artifact to a remote repository defined in `<distributionManagement>`, such as Nexus or Artifactory, making it available to all developers and CI systems.

- **How do you configure Maven to use a specific Java version for compilation?**
  - Use the maven-compiler-plugin with `<source>` and `<target>` tags, or the newer `<release>` tag which sets both source, target, and the API check level for the specified Java version. For example, `<release>11</release>` ensures compilation against the Java 11 API. Set these in `<properties>` as `maven.compiler.source`, `maven.compiler.target`, or `maven.compiler.release`.

- **What is a Maven wrapper and why would you use it?**
  - The Maven Wrapper (mvnw) is a script that downloads and runs a specific, pinned version of Maven. It ensures all developers and CI environments use the exact same Maven version, eliminating build inconsistencies caused by version differences. Generate it with `mvn -N wrapper:wrapper` and commit the mvnw script and the `.mvn` directory to version control.

- **How do you skip tests in Maven and what are the different options?**
  - Use `-DskipTests` to skip test execution but still compile test classes. Use `-Dmaven.test.skip=true` to skip both compilation and execution of tests. Use `-Dit.test=none` to skip integration tests with the failsafe plugin. For selective skipping, use `-Dtest=!SomeTest` to exclude specific test classes. Skipping tests should only be done in development, never in release builds.

- **Explain how Maven's dependency scopes affect the classpath of a web application deployed to a servlet container.**
  - For a WAR deployed to Tomcat, use `provided` scope for Servlet API and other container-provided libraries — they are on the container classpath but should not be bundled in WEB-INF/lib. Use `compile` for application libraries that must be packaged. Use `runtime` for JDBC drivers and similar libraries needed only at runtime. Use `test` for JUnit and test frameworks.

- **What is the purpose of the dependency:tree goal and how do you interpret its output?**
  - `mvn dependency:tree` prints a tree of all resolved dependencies showing transitive relationships, scopes, and versions. Each level indentation shows the dependency chain. Conflicts are marked with `(version managed from X)` or omitted when suppressed. It is the primary diagnostic tool for understanding why a specific library version was chosen or why duplicates appear on the classpath.

- **How do you configure Maven to fail the build if test coverage drops below a threshold?**
  - Use the maven-surefire-plugin with the JaCoCo Maven plugin for code coverage. Configure JaCoCo's `check` goal with `<rules><rule><limits><limit><counter>LINE</counter><value>COVEREDRATIO</value><minimum>0.80</minimum></limit></limits></rule></rules></rules>`. Bind the check goal to the `verify` phase so it runs after tests. The build fails if coverage falls below the configured threshold.

## Developer Recommendations

- **Always use the Maven Wrapper**
  - Pins the exact Maven version for all environments, eliminating build inconsistencies caused by different Maven versions.
  - Generate the wrapper with `mvn -N wrapper:wrapper` and commit the mvnw script, the mvnw.cmd script, and the .mvn directory to version control.
  - **Production story:** A team spent three days debugging a CI build failure that only occurred on the build server. The root cause was a newer Maven version changing the default compiler source/target levels. Switching to the wrapper eliminated this class of problem instantly.

- **Centralize dependency and plugin versions in the parent POM**
  - Prevents version drift across modules and makes upgrades a single-point change.
  - Declare all dependency versions under `<dependencyManagement>` in the parent POM and omit versions in submodule `<dependencies>` blocks. Do the same for plugins using `<pluginManagement>`.

- **Inspect and exclude transitive dependencies proactively**
  - Keeps the classpath clean and prevents runtime conflicts before they reach production.
  - Before adding a new dependency, run `mvn dependency:tree` to understand its transitive footprint. Add exclusions for any conflicting or unnecessary libraries immediately, not when a conflict surfaces.

- **Pin every plugin version explicitly**
  - Protects against build breakage from Maven version upgrades that change default plugin versions.
  - Specify the version attribute on every `<plugin>` declaration. Use `<pluginManagement>` to centralize these versions across all modules rather than repeating them in each submodule.
