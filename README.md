# Build Tool Demonstrations

Three minimal projects showing I can build runnable artifacts with the
major JVM and Node build tools.

## Projects
- **my-app** — Java app built with **Maven** (`mvn package` → `target/*.jar`)
- **my-gradle-app** — Java app built with **Gradle** (`./gradlew build` → `build/libs/*.jar`)
- **my-node-app** — Node.js/Express app built with **npm** (`npm install`, `npm start`)

## Key points
- Each produces a deployable artifact from source
- Build output (target/, build/, node_modules/) is gitignored — only source and lock files are committed
- The Gradle wrapper (gradlew) is committed so the project builds with its pinned Gradle version
