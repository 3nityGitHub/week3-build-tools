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
## Deploying to a cloud server

How the `my-gradle-app` Java application was deployed to a remote, secured
Linux server on AWS EC2.

### 1. Provision
- Launched an Ubuntu `t2.micro` instance on AWS EC2 (x86_64).
- Created an RSA key pair (`.pem`) for SSH access.

### 2. Secure the network (security group)
- Allowed inbound SSH (port 22) from **my IP only** — scoped to my IPv4 `/32`,
  never `0.0.0.0/0`.
- Gotcha I hit: AWS "My IP" captured my **IPv6** address, but SSH connects over
  **IPv4**, so the rule never matched and every attempt timed out. Fixed by
  setting the source manually to my IPv4 `/32` (from `curl -4 ifconfig.me`).

### 3. Harden the server
- Created a non-root user with sudo:
  `sudo adduser rapheal` then `sudo usermod -aG sudo rapheal`.
- Gave that user key-based login (`~/.ssh/authorized_keys`, dir `700`, file `600`).
- Edited `/etc/ssh/sshd_config`: `PermitRootLogin no`, `PasswordAuthentication no`.
- Restarted SSH and verified the effective config with `sudo sshd -T`.

### 4. Deploy and run
- Installed Java: `sudo apt install -y openjdk-21-jdk-headless`.
- Cloned the repo and built + ran with the Gradle wrapper:
  `./gradlew run` → prints `Hello World!`.

### 5. Teardown (cost control)
- Stopped the instance when pausing; terminated it when finished.
- Set a billing alert as a safety net.

### Why a non-root user matters
Running the app and day-to-day work as a limited user instead of root means that
if the app or my session is ever compromised, the attacker only inherits that
user's limited permissions — not control of the whole server. Admin actions use
`sudo` only when needed. This is the principle of least privilege.
