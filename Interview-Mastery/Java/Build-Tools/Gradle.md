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

## Scenario-Based Questions

**Q: Your Gradle build is noticeably slow on developer machines. What steps do you take to diagnose and fix the performance issue?**

- First, enable the build cache with `--build-cache` and check if it is properly configured. Run `gradle build --scan` to generate a build scan that reveals task execution times, cache hit rates, and configuration bottlenecks. Ensure incremental compilation is enabled. Check for top-level code that runs during configuration instead of execution. Enable parallel execution with `org.gradle.parallel=true` in gradle.properties. Upgrade to the latest Gradle version to leverage performance improvements.

- **Interview follow-up:** How would you set up a shared remote build cache for your team, and what security considerations would you address?

**Q: Two modules in your multi-project build depend on different versions of the same library, causing a runtime conflict. How do you resolve this?**

- Use the `gradle dependencies` task to inspect the dependency trees of both modules. Identify the conflicting versions. If one module can upgrade, change its dependency version. If both versions are required by third-party libraries, use dependency constraint declarations with `constraints { }` or force a consistent version using `resolutionStrategy.force()` in the subprojects block.

- **Interview follow-up:** What is the difference between dependency constraints and direct dependencies, and when would you use each?

## Interview Questions

- **What is the difference between Groovy DSL and Kotlin DSL in Gradle?**
  - Groovy DSL uses .gradle files with dynamic typing, concise syntax, and implicit closures. Kotlin DSL uses .gradle.kts files with static type checking, better IDE autocompletion, and compile-time error detection. Kotlin DSL is the recommended choice for new projects because of its safety and IDE support. Groovy DSL remains widely used in existing projects and has more concise syntax for simple builds.

- **Explain the configuration phase and execution phase in Gradle.**
  - The configuration phase evaluates the build script and constructs the task graph (DAG). Code placed at the top level of build.gradle runs during this phase. The execution phase runs the tasks in the order determined by the DAG. Code inside doFirst or doLast blocks runs during execution. A common mistake is placing expensive operations in the configuration phase, which slows down every build invocation even when those tasks are not executed.

- **What is the difference between implementation and api dependency configurations?**
  - implementation makes the dependency available to the current module's compile and runtime classpaths but does not leak it to consumers. api makes the dependency available to the current module and also leaks it to consumers' compile classpaths. Use implementation by default. Use api only when types from the dependency appear in the public API of your module (e.g., a class extends a library class or a method signature uses a library type).

- **How does the Gradle build cache work, and what are its benefits?**
  - The build cache stores task outputs keyed by task inputs (source files, dependency versions, task configuration). When a task is invoked again with identical inputs, Gradle skips execution and retrieves the cached output. This dramatically reduces build times for incremental changes and CI pipelines where only a subset of modules change. The cache can be local or remote.

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
