# Artifact Management with Nexus

This document records how I publish build artifacts from this repository
(`my-app` — Maven, and `my-gradle-app` — Gradle) to a self-hosted
**Sonatype Nexus** repository. It is Week 5 of my DevOps programme (Portfolio
Project P2).

---

## What an artifact repository is

When a build tool finishes, it produces an **artifact** — a single packaged,
versioned file ready to run or ship (here, a `.jar`). An **artifact
repository** is a server whose job is to store and serve those built
artifacts, each under a name and version, so any machine or pipeline can fetch
the exact one it needs. **Nexus** is the tool I run for this.

## Source repo (Git) vs artifact repo (Nexus) — why both exist

| Git (GitHub)                              | Nexus                                   |
| ----------------------------------------- | --------------------------------------- |
| Holds **source code** (human-written text)| Holds **built artifacts** (compiled jars)|
| Changes constantly, many small commits    | Stores finished, versioned releases     |
| Diffed, branched, merged, reviewed        | Fetched by version; immutable once published |
| `git clone` / `git pull`                  | build tool `deploy` / `publish`         |

In one line: **Git holds the recipe; the build tool cooks it; Nexus stores the
finished dish so anyone can collect the exact version they ordered.**

Why both: you don't rebuild a dependency from source every time you need it, and
you don't commit heavy binary `.jar` files into Git (it bloats history and isn't
what Git is for). Binaries belong in the artifact repo — fast to fetch,
versioned, shared across pipelines.

**Release vs snapshot:** a release (e.g. `1.0.0`) is final and immutable once
published. A snapshot (e.g. `1.0-SNAPSHOT`) is a moving work-in-progress build
that can be overwritten. Nexus keeps them in separate repos (`maven-releases`,
`maven-snapshots`). Both artifacts here publish as `1.0-SNAPSHOT`.

---

## How I run Nexus

Nexus runs locally in Docker (free; needs ~4 GB RAM, so Docker Desktop memory is
set to 6 GB — a free-tier `t2.micro` cannot run it):

```bash
docker volume create nexus-data
docker run -d -p 8081:8081 --name nexus -v nexus-data:/nexus-data sonatype/nexus3
# first-login password:
docker exec nexus cat /nexus-data/admin.password
# UI: http://localhost:8081   (stop with: docker stop nexus / start with: docker start nexus)
```

## Security: least-privilege deploy user

Builds authenticate as a dedicated **`publisher`** user, never as `admin`. The
`publisher` user holds a role with only the `view` privileges on the two Maven
repos — `add`, `edit`, `read`, `browse`:

```
nx-repository-view-maven2-maven-snapshots-add
nx-repository-view-maven2-maven-snapshots-edit
nx-repository-view-maven2-maven-snapshots-read
nx-repository-view-maven2-maven-snapshots-browse
   (and the maven-releases equivalents)
```

> **`view` vs `admin` privileges:** `view` grants access to the repo's
> *contents* (the artifacts). `admin` grants access to the repo's
> *configuration*. A deploy user needs `view`, never `admin`.

**Credentials never live in the repository.** They sit in home-directory config
outside every project, so nothing secret can be committed to Git:
- Maven → `~/.m2/settings.xml`
- Gradle → `~/.gradle/gradle.properties`

---

## Publishing `my-app` (Maven)

**`pom.xml`** — where to publish (URLs only, committed):

```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>http://localhost:8081/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>http://localhost:8081/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

**`~/.m2/settings.xml`** — credentials (NOT committed; the `<id>` must match the
pom exactly, or the request falls back to anonymous and fails):

```xml
<settings>
  <servers>
    <server>
      <id>nexus-snapshots</id>
      <username>publisher</username>
      <password>REDACTED</password>   <!-- escape & as &amp; in XML -->
    </server>
    <!-- nexus-releases block identical -->
  </servers>
</settings>
```

Publish:

```bash
cd my-app
mvn deploy
```

Result: `com/talium/my-app/1.0-SNAPSHOT/` appears in `maven-snapshots`, with a
timestamped jar (e.g. `my-app-1.0-20261020.145528-2.jar`).

## Publishing `my-gradle-app` (Gradle)

Gradle uses the `maven-publish` plugin instead of `distributionManagement`.

**`app/build.gradle`:**

```groovy
plugins {
    id 'application'
    id 'maven-publish'
}

group = 'com.talium'
version = '1.0-SNAPSHOT'

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
            artifactId = 'my-gradle-app'   // override default module name 'app'
        }
    }
    repositories {
        maven {
            def releasesUrl  = 'http://localhost:8081/repository/maven-releases/'
            def snapshotsUrl = 'http://localhost:8081/repository/maven-snapshots/'
            url = version.toString().endsWith('SNAPSHOT') ? snapshotsUrl : releasesUrl
            allowInsecureProtocol = true   // only because this is plain http on localhost
            credentials {
                username = project.findProperty('nexusUser')
                password = project.findProperty('nexusPass')
            }
        }
    }
}
```

**`~/.gradle/gradle.properties`** (NOT committed; properties file, so no XML
escaping needed):

```
nexusUser=publisher
nexusPass=REDACTED
```

Publish:

```bash
cd my-gradle-app/app
./gradlew publish
```

## Verifying a publish

Via the Nexus REST API (more pipeline-friendly than clicking the UI):

```bash
curl -s -u publisher "http://localhost:8081/service/rest/v1/search?group=com.talium"
```

Returns the components under `com.talium` — confirms both `my-app` and
`my-gradle-app` are present.

---

## Errors I hit (and fixes)

- **403 Forbidden on deploy** — the deploy user had `nx-repository-admin-*`
  privileges instead of `nx-repository-view-*`. Fixed by granting `view`
  privileges (admin = config, view = contents).
- **`Non-parseable settings` / `entity reference` error** — a `&` in the Nexus
  password broke the XML in `settings.xml`. Fixed by escaping it as `&amp;`
  (`&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`). Gradle's `gradle.properties` is
  not XML, so it needs no escaping.
- **`401` vs `403`** — `401` = Nexus doesn't know who you are (bad/missing
  credentials or an `<id>` mismatch causing an anonymous request). `403` = it
  knows you but you lack the permission.
- **Gradle insecure-protocol error** — Gradle refuses plain `http` by default;
  `allowInsecureProtocol = true` is required for a localhost Nexus (use `https`
  in production).
- **Nexus won't start / read-only filesystem** — Nexus needs ~4 GB RAM;
  under-provisioning crashes it. A read-only Docker VM filesystem was fixed by
  restarting Docker Desktop and pruning the build cache.

