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

## Scenario-Based Questions

**Q: Your multi-module build fails at runtime with a "Duplicate class found" error, but compilation and tests pass without issues. How do you diagnose and fix the problem?**

- Run `mvn dependency:tree` on each module to identify the conflicting library appearing on multiple dependency paths. Use the dependency tree output to determine which direct dependency introduces the unwanted transitive library. Add an `<exclusion>` to that dependency's declaration to suppress the duplicate. If the conflict is widespread, use `<dependencyManagement>` in the parent POM to force a single version across all modules.

- **Interview follow-up:** How would you handle this if the duplicate class comes from an uber-JAR created by the maven-shade-plugin that shades its own copy of a library already on the classpath?

**Q: A teammate cannot build the project because their corporate proxy blocks access to Maven Central. How do you configure Maven to work in this environment?**

- Add a mirror entry in settings.xml pointing to the internal corporate repository (Nexus or Artifactory) that proxies Maven Central. Also configure any required HTTP proxy settings in settings.xml using `<proxies><proxy>`. Ensure the mirror is configured as a `<mirrorOf>central</mirrorOf>` to redirect all Central requests.

- **Interview follow-up:** How would you ensure that developers outside the corporate network can still build the project without modifying configuration files?

## Interview Questions

- **Explain the Maven build lifecycle and how phases, goals, and plugins relate to each other.**
  - The Maven build lifecycle defines a sequence of phases (validate, compile, test, package, verify, install, deploy) that execute in strict order. Each phase is a step in the build process. Goals are concrete build tasks provided by plugins. Plugins declare which goals bind to which lifecycle phases. When Maven executes a phase, it runs all goals bound to that phase and all goals bound to every preceding phase. For example, maven-compiler-plugin binds its compile goal to the compile phase, and maven-surefire-plugin binds its test goal to the test phase.

- **What is the difference between provided and compile scope?**
  - compile scope makes the dependency available on all classpaths (compile, test, runtime) and propagates transitively to consumers. provided scope makes the dependency available at compile time and test time but excludes it from the runtime classpath and does not propagate transitively. Use provided when the runtime environment (e.g., a servlet container) supplies the library.

- **How do you manage dependency versions across a multi-module Maven project?**
  - Use a parent POM with `<dependencyManagement>` to centralize version declarations. Submodules declare dependencies in their own `<dependencies>` without version tags — they inherit versions from the parent's `<dependencyManagement>`. For plugins, use `<pluginManagement>` in the parent POM. The parent POM also defines `<modules>` so a single command builds the entire project as a reactor.

- **What is a BOM (Bill of Materials) and when would you use it?**
  - A BOM is a specialized POM with only a `<dependencyManagement>` section and no dependencies of its own. It is imported using `<scope>import</scope>` and `<type>pom</type>` in the importing POM's `<dependencyManagement>`. BOMs centralize dependency versions for an entire library ecosystem, such as the Spring Boot BOM or the Jackson BOM, so consumers can declare dependencies without specifying versions.

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
