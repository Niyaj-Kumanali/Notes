# Gradle

- Purpose: build automation tool emphasizing flexibility, performance, and a DSL-based approach
- Groovy DSL: first scripting language for Gradle builds, uses .gradle files, more concise
- Kotlin DSL: newer, type-safe alternative using .gradle.kts files, better IDE support
- build.gradle (or build.gradle.kts) declares plugins, dependencies, and repositories
- Plugins extend build functionality — applied via `plugins { }` block
- Repositories: Maven Central, Google, jcenter, or custom URLs resolved in `repositories { }`
- Dependencies declared in `dependencies { }` block with configurations
- Tasks: units of work written declaratively with actions (doLast, doFirst) and dependsOn
- Task graph: Gradle resolves task dependencies into a directed acyclic graph (DAG) before executing
- Configuration phase: evaluates build scripts to construct the task graph
- Execution phase: runs tasks in order defined by the DAG
- Build cache: reuses outputs from previous builds when inputs haven't changed — enables incremental builds
- Dependency configurations: implementation (internal dep not leaked to consumers), api (leaked to consumers), compileOnly (like provided scope), runtimeOnly (needed at runtime), testImplementation (test deps)
- Dependency locking: `dependencyLocking { }` pins exact transitive versions for reproducibility
- Multi-project builds: settings.gradle defines project names and locations; subprojects { } or allprojects { } blocks apply common config
- Gradle Wrapper (gradlew): generates a script that downloads and runs a specific Gradle version
- Comparison with Maven: Gradle is more flexible and faster (incremental, build cache, daemon) but Maven has stricter conventions and is more predictable; Gradle uses a DAG instead of a linear lifecycle; Maven uses XML, Gradle uses a DSL
