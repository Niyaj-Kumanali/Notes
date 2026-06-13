# Maven

- Purpose: build automation and dependency management for Java projects
- pom.xml is the project descriptor — XML-based configuration file
- groupId: reverse domain name identifying the project organization (e.g., com.company)
- artifactId: name of the project or module being built
- version: release version of the artifact (e.g., 1.0.0-SNAPSHOT)
- dependencies section declares libraries the project depends on
- Lifecycle phases executed in order: validate, compile, test, package, verify, install, deploy
- A phase is a step in the build lifecycle
- A goal is a specific task bound to a phase
- A plugin bundles one or more goals and binds them to lifecycle phases
- Dependency scopes: compile (default, available everywhere), provided (JDK or container supplies it), runtime (needed at runtime only), test (test classpath only), system (like provided but must include path), import (used only in dependencyManagement in a POM)
- Transitive dependencies are pulled in automatically from direct dependencies
- Exclusions: `<exclusions><exclusion>` block inside a dependency to omit a transitive dep
- Maven Wrapper (mvnw): a script that downloads and runs a pinned Maven version, ensuring build reproducibility
- settings.xml: user/global config for profiles (environment-specific settings), mirrors (redirect repos), servers (auth credentials for repositories)
- Multi-module projects: parent POM with `<modules>` list defining submodules; reactor build resolves inter-module dependencies and builds them in order
- Common plugins: compiler (compile Java sources), surefire (unit tests), failsafe (integration tests), shade (create uber-JAR), assembly (custom distribution archives)
