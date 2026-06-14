# Gradle

## Overview

- **Definition** — Gradle is a build automation tool that uses a DSL (Groovy or Kotlin) with a directed acyclic graph (DAG) task model and incremental build support for Java, Kotlin, and other JVM projects.
- **Why It Exists** — Provides a faster, more flexible alternative to Maven by combining convention-over-configuration with a programmable build environment, incremental compilation, build caching, and a daemon for persistent performance.
- **Historical Context** — First released in 2012, Gradle was designed to address Maven's rigidity and XML verbosity while improving build performance through incremental builds, a build cache, and a long-running daemon process. It became the default build system for Android and gained wide adoption in the JVM ecosystem.
- **Key Concepts** — **DSL** (domain-specific language for build scripts), **Task** (unit of work), **Task Graph** (DAG of task dependencies), **Configuration phase** (script evaluation to build the graph), **Execution phase** (running tasks in dependency order), **Plugin** (extensible build capability), **Dependency configuration** (classpath bucket), **Build cache** (reuses task outputs), **Daemon** (long-lived JVM for fast builds), **Wrapper** (pinned Gradle version).

## Core Concepts

- Groovy DSL uses .gradle files with a concise, dynamic syntax. Kotlin DSL uses .gradle.kts files with type-safe accessors, better IDE autocompletion, and compile-time error detection. Kotlin DSL is the recommended choice for new projects.
- build.gradle or build.gradle.kts is structured around three core blocks: `plugins { }` to apply Gradle plugins, `repositories { }` to configure dependency sources (mavenCentral, google, jcenter, mavenLocal, custom URLs), and `dependencies { }` to declare dependencies using configuration names.
- Tasks are the fundamental unit of work. Tasks are declared declaratively with typed configuration (e.g., `Copy`, `JavaCompile`), wired using `dependsOn`, and enhanced with `doFirst` and `doLast` closures that run during execution. Custom tasks extend `DefaultTask`.
- The task graph is a directed acyclic graph (DAG) that Gradle constructs during the configuration phase. It resolves task dependencies, detects circular dependencies, and determines the optimal execution order.
- The configuration phase executes the build script to construct the task graph. The execution phase runs the tasks in the order determined by the DAG. Code inside `doFirst`/`doLast` runs during execution. Code at the top level of the build script runs during configuration.
- The build cache stores outputs from previous builds. When inputs to a task have not changed, Gradle reuses the cached output instead of re-executing the task. This enables incremental builds. The build cache can be local or remote (shared across a CI fleet).
- Dependency configurations control how libraries appear on classpaths and whether they are exposed to consumers:
  - **implementation**: dependency is available at compile and runtime for the current module, not leaked to consumers
  - **api**: dependency is available at compile and runtime for the current module and leaked to consumers (replaces deprecated compile)
  - **compileOnly**: dependency is available at compile time only, not at runtime (equivalent to Maven provided scope)
  - **runtimeOnly**: dependency is available at runtime only, not at compile time
  - **testImplementation**: dependency is available for test compilation and execution

- Dependency locking pins exact transitive versions for reproducibility. Enable it with `dependencyLocking { }` and use `gradle dependencies --write-locks` to generate a lock file. This is equivalent to Maven's enforcer plugin or npm's lock file.
- Multi-project builds are defined in settings.gradle by declaring project names and their locations. The `subprojects { }` block applies configuration to all subprojects. Dependencies between subprojects are declared as `implementation project(':module-name')`.
- The Gradle Wrapper (gradlew) generates scripts that download and run a specific Gradle version. Generate it with `gradle wrapper --gradle-version x.y.z` and commit the scripts to version control. This ensures reproducible builds across all environments.
- Compared to Maven, Gradle offers faster builds (daemon, incremental compilation, build cache), a more concise DSL instead of verbose XML, a flexible DAG task model instead of a rigid three-lifecycle structure, and better multi-project support. Maven has stricter conventions and more predictable behavior, making it simpler for teams that prefer a fixed build model.

## Common Mistakes

- **Using compile instead of implementation**
  - The deprecated compile configuration exposes internal dependencies to consumers, causing unnecessary transitive dependencies and classpath pollution. The module's internal implementation details become part of the public API contract.
  - **Why it looks correct:** compile was the standard configuration in older Gradle versions, and many online examples still use it. The build works correctly with compile.
  - Replace compile with implementation for internal dependencies. Use api only when a dependency's types appear in the module's public API. Run the Gradle built-in upgrade task to migrate.

- **Misunderstanding configuration vs execution phase**
  - Code that should execute at build time (e.g., file operations, network calls) is placed at the top level of the build script instead of inside a task action or a doLast/doFirst block. This code runs during configuration, which executes even when the task is not needed, slowing down every invocation.
  - **Why it looks correct:** The code works and produces the expected output during normal builds. The performance impact is not immediately noticeable on small projects.
  - Wrap all build-time logic inside `doLast { }` or `doFirst { }` closures inside a task definition. Use `project.afterEvaluate { }` only for configuration that genuinely depends on the fully resolved project state.

- **Not enabling the build cache**
  - The build cache is disabled by default in local environments, causing full rebuilds on every invocation even when inputs have not changed. This wastes developer time and CI resources.
  - **Why it looks correct:** Builds complete successfully, and without a baseline comparison the slowdown is invisible.
  - Enable the build cache by adding `--build-cache` to the command line or setting `org.gradle.caching=true` in gradle.properties. Configure a remote build cache in CI for team-wide sharing.

## Real-World Scenarios

### Migrating a legacy Maven project to Gradle

- A team maintains a 50-module Maven project with a complex parent POM, custom plugins, and profiles for different environments. They migrate to Gradle by generating build.gradle.kts files for each module using the built-in Maven-to-Gradle converter (`gradle init --type pom`). The parent POM configuration is translated into a Kotlin DSL root build.gradle.kts with subprojects blocks. Maven profiles become Gradle build variants. The team adopts implementation over compile, configures the build cache, and sets up the Gradle Wrapper for reproducible builds. Build time drops from 12 minutes to 4 minutes.

### Optimizing build times in a large monorepo

- A company with a 100-module Java monorepo experiences 20-minute CI build times. The team enables the local build cache on developer machines and sets up a shared remote build cache using an HTTP server. They configure parallel execution (`org.gradle.parallel=true`) and use the `--build-cache` flag in CI. Modules that change infrequently are cached and reused across builds. CI times drop to 6 minutes, and developer rebuilds after small changes take seconds.

## Use Cases

Reach for Gradle when build speed, flexibility, and a programmable build model matter more than a rigid, declarative lifecycle.

- **Large monorepo builds** — Build cache, daemon, incremental compilation, and parallel execution keep 100+ module projects fast.
  - CI times drop from minutes to seconds for unchanged modules.
  - **Avoid when:** the project is small/simple — Maven's predictability may be a better fit.

- **Android application development** — Gradle is the official Android build system with product flavors, build variants, and APK/AAB packaging.
  - Declare different build types (debug/release), signing configs, and resource sets per variant.
  - **Avoid when:** you build a pure-Java library that doesn't need Android-specific features.

- **Custom build logic and code generation** — Gradle's Kotlin/Groovy DSL handles custom tasks, annotation processing, and code generation inline.
  - Write a `Task` class or add `doFirst`/`doLast` hooks without leaving the build script.
  - **Avoid when:** the team prefers a purely declarative, XML-based build configuration.

- **Migrating from Maven** — `gradle init --type pom` converts an existing Maven project to Gradle, translating POM structure into Kotlin DSL.
  - Ideal when Maven builds have become unacceptably slow or the team needs more flexibility.
  - **Avoid when:** the existing Maven setup is fast and the team is comfortable with it.

- **Polyglot multi-language builds** — A single Gradle build can compile Java, Kotlin, Groovy, Scala, C++, and JavaScript.
  - Useful for full-stack JVM projects that span multiple JVM languages.
  - **Avoid when:** the project uses only Java — Maven's simplicity may be preferable.

---

## Scenario-Based Questions

**Q: Your Gradle build is noticeably slow on developer machines. What steps do you take to diagnose and fix the performance issue?**

- First, enable the build cache with `--build-cache` and check if it is properly configured. Run `gradle build --scan` to generate a build scan that reveals task execution times, cache hit rates, and configuration bottlenecks. Ensure incremental compilation is enabled. Check for top-level code that runs during configuration instead of execution. Enable parallel execution with `org.gradle.parallel=true` in gradle.properties. Upgrade to the latest Gradle version to leverage performance improvements.

- **Interview follow-up:** How would you set up a shared remote build cache for your team, and what security considerations would you address?

**Q: Two modules in your multi-project build depend on different versions of the same library, causing a runtime conflict. How do you resolve this?**

- Use the `gradle dependencies` task to inspect the dependency trees of both modules. Identify the conflicting versions. If one module can upgrade, change its dependency version. If both versions are required by third-party libraries, use dependency constraint declarations with `constraints { }` or force a consistent version using `resolutionStrategy.force()` in the subprojects block.

- **Interview follow-up:** What is the difference between dependency constraints and direct dependencies, and when would you use each?


**Q: Your Gradle build fails with "Could not resolve dependency" for a library that exists in Maven Central. Other developers can build successfully. What could be the issue?**

- Check if the local Gradle cache is corrupted by deleting the cached version in `~/.gradle/caches/` or running `gradle clean build --refresh-dependencies`. Verify that `repositories { mavenCentral() }` is declared in the correct build script. Check for network proxy settings in `gradle.properties` with systemProp.http.proxyHost and systemProp.http.proxyPort. Ensure the Gradle version is the same as the team's — run `gradle --version`.

- **Interview follow-up:** How would you configure a corporate Artifactory or Nexus repository as a Gradle repository replacement for Maven Central?


**Q: You add a new library to your Java module using the api configuration, and suddenly all downstream modules fail to compile. What happened?**

- The `api` configuration exposes the dependency to consumers, meaning any module that depends on your module now requires that library on its compile classpath. If the library's transitive dependencies conflict with the downstream module's dependencies, compilation fails. Change the configuration to `implementation` if the library's types are not part of your module's public API. Only use `api` when the dependency's types appear in your exposed interfaces or superclasses.

- **Interview follow-up:** How would you identify which transitive dependency is causing the compilation failure in the downstream module?


**Q: Your CI build times have doubled after upgrading Gradle. How do you identify and fix the regression?**

- Generate a build scan with `gradle build --scan` to compare task durations before and after the upgrade. Look for tasks that lost cacheability due to input changes, or tasks that now run during configuration that previously ran during execution. Check if the new Gradle version changed default task inputs or outputs. Review the upgrade notes for deprecated features that might have changed behavior. Compare the build scan with a pre-upgrade scan to pinpoint the regression.

- **Interview follow-up:** What Gradle properties would you tune (org.gradle.parallel, org.gradle.daemon, org.gradle.caching) for optimal CI performance, and why?


**Q: Your multi-project build has subproject A depending on subproject B. When you change a file in B, A does not rebuild. How do you fix this?**

- Ensure the dependency is declared correctly with `implementation(project(":B"))` or `api(project(":B"))` in A's build script. Verify that incremental compilation is enabled by default. Check if the build cache is returning stale cached outputs for A — run with `--no-build-cache` to test. Ensure the settings.gradle file includes both projects. If using composite builds, verify the substitution rules are correctly configured.

- **Interview follow-up:** How does Gradle determine which tasks are out-of-date, and what are task inputs and outputs?


**Q: Your team wants to enforce that no build ever uses a snapshot dependency in a release build. How do you configure Gradle for this?**

- Configure resolution strategy to fail on dynamic versions. Use `resolutionStrategy { failOnDynamicVersions() }` or `resolutionStrategy { failOnChangingVersions() }` in the subprojects block. For more control, use the `dependencyLocking` plugin to pin exact versions and verify the lock file is up to date during release builds. Use CI pipeline validation that runs `gradle dependencies --write-locks` and fails if the lock file changed.

- **Interview follow-up:** How would you allow snapshot dependencies in development builds while blocking them in release builds?


**Q: A developer reports that a Gradle plugin they added works on their machine but the CI server says the plugin does not exist. What is likely wrong?**

- The CI server may not have access to the plugin repository. If the plugin is from the Gradle Plugin Portal, ensure `pluginManagement { repositories { gradlePluginPortal() } }` is configured. If the plugin is from a custom repository or Artifactory, that repository URL must be accessible from CI. The Gradle version on CI may be too old to support the plugin's required API. Check the plugin's `buildscript` declaration versus `plugins` block — the `plugins` block is preferred.

- **Interview follow-up:** How do you apply a plugin that is only needed for a specific subproject without affecting the others?


**Q: Your Gradle build produces an uber-JAR with a FAT file system error when unzipping. How do you fix this?**

- Duplicate files in the uber-JAR cause zip format issues. When multiple dependencies contain the same file path (e.g., META-INF/services, META-INF/LICENSE), the shadow plugin fails or produces a corrupted JAR. Use the shadow plugin's `mergeServiceFiles()` to merge service files, or use `exclude('META-INF/*.SF', 'META-INF/*.DSA')` to exclude signing files. For resources, use `append('META-INF/foo')` to concatenate files with the same name.

- **Interview follow-up:** What is the difference between the Shadow plugin and the Spring Boot Gradle plugin for creating executable JARs?


**Q: You need to publish a library to an internal Maven repository with signed artifacts. How do you configure Gradle for signing and publishing?**

- Apply the `maven-publish` plugin and the `signing` plugin. Configure a publication block with `from(components.java)` and an artifact. Configure signing with `signing { sign(publishing.publications) }`. Store the signing key in a secure location (not in the build script) using environment variables or a CI secrets store. Use `signing.keyId`, `signing.password`, and `signing.secretKeyRingFile` configured in `gradle.properties` or passed as project properties.

- **Interview follow-up:** How would you publish to different repositories (snapshots vs releases) based on the version string?

## Interview Questions

- **What is the difference between Groovy DSL and Kotlin DSL in Gradle?**
  - Groovy DSL uses .gradle files with dynamic typing, concise syntax, and implicit closures. Kotlin DSL uses .gradle.kts files with static type checking, better IDE autocompletion, and compile-time error detection. Kotlin DSL is the recommended choice for new projects because of its safety and IDE support. Groovy DSL remains widely used in existing projects and has more concise syntax for simple builds.

- **Explain the configuration phase and execution phase in Gradle.**
  - The configuration phase evaluates the build script and constructs the task graph (DAG). Code placed at the top level of build.gradle runs during this phase. The execution phase runs the tasks in the order determined by the DAG. Code inside doFirst or doLast blocks runs during execution. A common mistake is placing expensive operations in the configuration phase, which slows down every build invocation even when those tasks are not executed.

- **What is the difference between implementation and api dependency configurations?**
  - implementation makes the dependency available to the current module's compile and runtime classpaths but does not leak it to consumers. api makes the dependency available to the current module and also leaks it to consumers' compile classpaths. Use implementation by default. Use api only when types from the dependency appear in the public API of your module (e.g., a class extends a library class or a method signature uses a library type).

- **How does the Gradle build cache work, and what are its benefits?**
  - The build cache stores task outputs keyed by task inputs (source files, dependency versions, task configuration). When a task is invoked again with identical inputs, Gradle skips execution and retrieves the cached output. This dramatically reduces build times for incremental changes and CI pipelines where only a subset of modules change. The cache can be local or remote.

- **What is a Gradle task and how do you create a custom one?**
  - A task is the fundamental unit of work in Gradle. Create a custom task by defining a class extending `DefaultTask` in `buildSrc` or the build script. Add `@TaskAction` methods for execution logic and `@Input`/`@Output` annotations for cache key inputs. Register tasks with `tasks.register<MyTask>("myTask")` in Kotlin DSL or `tasks.register('myTask', MyTask)` in Groovy DSL. Wire dependencies using `dependsOn` and configure with typed extensions.

- **Explain the difference between plugins applied in the plugins block versus the buildscript block.**
  - The `plugins` block (DSL-first) resolves plugins from the Gradle Plugin Portal or a custom pluginManagement repository. It provides type-safe accessors and automatic classpath management. The `buildscript` block (legacy) manually manages the plugin's classpath dependency and applies it with `apply plugin:`. The `plugins` block is preferred for new projects. Buildscript is still needed for some older or complex plugin configurations.

- **How do you configure Gradle to run tasks in parallel?**
  - Set `org.gradle.parallel=true` in gradle.properties to enable parallel project execution. Use `--max-workers` or `org.gradle.workers.max` to control the number of parallel threads (default is number of CPU cores). For test parallelism, configure `maxParallelForks` in the test task. Individual tasks can also control parallelism, such as `compileJava.options.fork = true` for parallel compilation.

- **What is the difference between gradle.properties and local.properties files?**
  - `gradle.properties` is a standard Gradle configuration file that sets JVM args, system properties, project properties, and build behavior (like `org.gradle.daemon`, `org.gradle.parallel`). It is typically committed to version control with safe defaults. `local.properties` is Android-specific (but can be used in any project) for machine-specific settings like SDK paths, and should not be committed to version control.

- **How do you exclude a transitive dependency in Gradle?**
  - Use the `exclude` keyword inside the dependency declaration block. For example: `implementation("com.example:lib:1.0") { exclude(group = "com.unwanted", module = "unwanted-lib") }`. For project-wide exclusions, use `configurations.all { exclude(group = "com.unwanted") }`. For more control, use dependency constraints or resolution strategy to force a specific version or substitute one dependency for another.

- **What is the purpose of the settings.gradle file?**
  - The settings.gradle (or settings.gradle.kts) file configures the project structure. It declares which subprojects exist via `include(":moduleA", ":moduleB")`, configures plugin management repositories, defines project names, and includes convention plugins from `buildSrc`. It runs before any build.gradle script and is required for multi-project builds.

- **How do you create a fat JAR (uber-JAR) in Gradle?**
  - Use the Shadow plugin (`com.github.johnrengelman.shadow`), the Gradle built-in `jar` task with `from(configurations.runtimeClasspath.map { ... })`, or the Spring Boot plugin's `bootJar` task. The Shadow plugin is most common — it merges service files, relocates packages to avoid conflicts, and creates a single executable JAR with all dependencies. Configure it with `shadowJar { mergeServiceFiles() }`.

- **What is a Gradle init script and when would you use one?**
  - An init script (init.gradle or .gradle/init.gradle.kts) runs before any project build script. It is used for environment-wide configuration like setting up corporate artifact repositories, applying plugins to all projects on a machine, configuring proxy settings, or enforcing global build policies. Init scripts are not committed to the project — they are placed in the Gradle user home directory or specified via the `-I` flag.

- **How does Gradle handle incremental compilation?**
  - Gradle's incremental Java compilation tracks source file changes at the class level rather than recompiling all sources. When a source file changes, only its class and dependent classes are recompiled. The compiler plugin analyzes the dependency graph of types within the module. Enable it with `options.incremental = true` in the compile task (enabled by default in modern Gradle versions). Incremental compilation significantly reduces build times during development.

- **Explain the difference between `gradle assemble` and `gradle build`.**
  - `gradle assemble` runs tasks that produce artifacts (JAR, WAR, classes) without running tests. `gradle build` runs the full lifecycle including compilation, testing, and packaging. Use `assemble` for quick artifact generation when tests are not needed. Use `build` for CI pipelines and release verification. `build` depends on `check` (test tasks) and `assemble` (packaging tasks).

- **How do you manage credentials for a private Maven repository in Gradle?**
  - Store credentials in `gradle.properties` with properties like `myRepoUsername=user` and `myRepoPassword=pass`, then reference them in the repository block: `maven { url = uri("..."); credentials { username = properties["myRepoUsername"]; password = properties["myRepoPassword"] } }`. Never hard-code credentials in build scripts. For CI, use environment variables or CI secrets injected via `-P` or system properties.

- **What is a Gradle configuration and how is it different from a dependency?**
  - A configuration is a named set of dependencies and artifacts that serves a specific classpath purpose (compile, runtime, test). Configurations are buckets — `implementation`, `api`, `compileOnly`, `testImplementation` are all configurations. Dependencies are libraries placed inside these buckets. Configurations extend each other (e.g., `testImplementation` extends `implementation`), and you can create custom configurations for custom classpaths.

- **How do you test a Gradle plugin?**
  - Use the `java-gradle-plugin` development plugin along with `gradleTestKit()` for integration testing. Write Spock or JUnit tests that apply the plugin to a temporary project, run Gradle tasks programmatically, and assert on build results. Test plugin extension configuration, task registration, and error handling. Use `@TempDir` for isolated test project directories. Run tests with `gradle check` as part of the normal build.

- **What is the purpose of the buildSrc directory?**
  - `buildSrc` is a precompiled build script directory that contains shared build logic (custom plugins, task types, extension classes) written in Kotlin DSL or Groovy. It is automatically compiled and added to the classpath of all project build scripts. Use it to avoid duplicating build logic across modules. Convention plugins in `buildSrc/src/main/kotlin` are the idiomatic way to share configuration in multi-project builds.

- **How does Gradle's dependency locking work?**
  - Dependency locking generates a lock file (`gradle.lockfile` or per-configuration lock files) that pins exact transitive versions. Enable it with `dependencyLocking { lockAllConfigurations() }`. Run `gradle dependencies --write-locks` to create or update lock files. Gradle then enforces these exact versions until the lock file is regenerated. This ensures reproducible builds and prevents unexpected dependency upgrades.

- **How do you configure Gradle to fail the build if a specific dependency version is used?**
  - Use resolution strategy with `failOnVersionConflict()` or define custom rules using `resolutionStrategy.eachDependency { if (it.target.name == "unwanted-lib") it.useVersion("2.0") }`. To ban a dependency entirely, use `resolutionStrategy.eachDependency { if (it.target.group == "com.unwanted") throw new DependencyRejection() }`. Configure this in the subprojects block for project-wide enforcement.

## Developer Recommendations

- **Use Kotlin DSL for all new projects**
  - Provides type safety, compile-time error detection, and superior IDE support compared to Groovy DSL. Reduces debugging time for build script errors.
  - Write build scripts as .gradle.kts files. Use type-safe accessors generated by the Kotlin DSL for plugin and dependency declarations.
  - **Production story:** A team spent hours debugging a Groovy DSL build that failed silently due to a missing import. The Kotlin DSL caught the same error at compile time in the IDE, eliminating the debugging cycle entirely.

- **Always enable the build cache**
  - The single highest-impact setting for improving build performance. Reuses unchanged task outputs across builds and machines.
  - Set `org.gradle.caching=true` in gradle.properties. Use `--build-cache` in CI commands. Configure a remote build cache for team-wide sharing.

- **Prefer implementation over compile**
  - Encapsulates internal dependencies and prevents classpath pollution in downstream projects. Matches modern Gradle best practices.
  - Replace all compile configurations with implementation. Only use api when a dependency's types are part of your module's public API surface. Run tests after migration to catch any missing transitive dependencies.

- **Adopt dependency locking for production projects**
  - Ensures reproducible builds by pinning exact transitive dependency versions. Prevents unexpected changes from dependency updates.
  - Add a `dependencyLocking { }` block in each module's build script. Run `gradle dependencies --write-locks` to generate lock files. Commit the lock files to version control and update them intentionally during dependency upgrades.
