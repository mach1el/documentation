# Nexus Repository with Proxy & Deploy User

This document describes how to run a **Nexus Repository Manager 3** instance in Docker behind a **corporate proxy**, and how to configure a **restricted deploy user** to publish artifacts to an internal Maven repository (e.g. `mich43l-libs`).

---

## 1. Overview

This setup provides:

- A **Nexus Repository 3** server running in Docker.
- Outbound traffic from Nexus going through a **HTTP/HTTPS proxy**.
- A dedicated **Maven hosted repo**: `mich43l-libs`.
- A **least-privilege deploy user** (`ci-deployer` or similar) allowed to publish artifacts.
- Instructions to configure **Maven** and use `mvn deploy:deploy-file` for manual artifact uploads.

---

## 2. Docker Setup (Nexus + Proxy)

Create a `docker-compose.yml` similar to:

```yaml
x-defaults: &defaults
  init: true
  restart: unless-stopped
  security_opt:
    - no-new-privileges:true
  cap_drop: [ "ALL" ]
  ulimits:
    nofile:
      soft: 65536
      hard: 65536
  networks:
    - nexus-net
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "5"

volumes:
  nexus-data:

networks:
  nexus-net:
    driver: bridge

services:
  nexus:
    <<: *defaults
    image: sonatype/nexus3:latest
    container_name: nexus
    ports:
      - "30081:8081"   # Host:Container
    volumes:
      - nexus-data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2048m -Xmx2048m -XX:MaxDirectMemorySize=2g
      - HTTP_PROXY=http://proxy.hcnet.local:3128
      - HTTPS_PROXY=http://proxy.hcnet.local:3128
      - NO_PROXY=localhost,127.0.0.1,10.0.0.1/8,*.hcnet.vn
      - http_proxy=http://proxy.hcnet.local:3128
      - https_proxy=http://proxy.hcnet.local:3128
      - no_proxy=localhost,127.0.0.1,10.0.0.1/8,*.hcnet.vn
```

### Start Nexus

```bash
docker compose up -d
```

Nexus UI will be available at:

```text
http://mynexus.repository:30081/
```

(default admin user is `admin`; fetch the initial password from `nexus-data/admin.password` inside the volume or container)

---

## 3. Configure Nexus HTTP Proxy (UI)

Inside Nexus:

1. Login as `admin`.
2. Go to **Administration → System → HTTP**.
3. Configure:
   - **HTTP/HTTPS proxy host**: `proxy.hcnet.local`
   - **Port**: `3128`
   - **Authentication**: if required by your proxy.
   - **Non-proxy hosts**: `localhost, 127.0.0.1, *.hcnet.vn, 10.0.0.0/8`, etc.

This allows Nexus itself (and its repository health / malware checks) to reach external registries through your corporate proxy.

---

## 4. Create the Maven Hosted Repository (`mich43l-libs`)

1. Go to **Administration → Repository → Repositories**.
2. Click **Create repository → maven2 (hosted)**.
3. Set:
   - **Name**: `mich43l-libs`
   - **Version policy**: `Release` (or `Mixed`, depending on usage).
   - **Layout policy**: `Strict`
   - **Deployment policy**:
     - `Allow redeploy` if you want to overwrite existing artifacts when needed, **or**
     - `Disable redeploy` if you want immutable artifacts for each version.
4. Save.

The repository URL will be:

```text
http://mynexus.repository:30081/repository/mich43l-libs/
```

---

## 5. Create a Role for Artifact Publishing

We will create a role that can **browse, read, add, and (optionally) edit** artifacts in `mich43l-libs`, without giving full admin rights.

1. Go to **Administration → Security → Roles**.
2. Click **Create role → Nexus role**.
3. Set:
   - **Role ID**: `mich43l-libs-publisher`
   - **Role name**: `Genesys SDK Publisher`
   - **Description**: `Role for publishing artifacts to mich43l-libs Maven repository.`
4. In **Privileges**, add the following:

   ```text
   nx-repository-view-maven2-mich43l-libs-browse
   nx-repository-view-maven2-mich43l-libs-read
   nx-repository-view-maven2-mich43l-libs-add
   nx-repository-view-maven2-mich43l-libs-edit    # optional, for redeploy/updates
   nx-search-read
   ```

5. Save the role.

> If you want a generic “publisher for all Maven repos”, you can alternatively use:
> `nx-repository-view-maven2-*-browse/read/add/edit`, but per-repo is safer.

---

## 6. Create the Deploy User

1. Go to **Administration → Security → Users**.
2. Click **Create local user**.
3. Fill in:
   - **User ID**: `ci-deployer` (or any name you like)
   - **First Name / Last Name**: e.g. `CI` / `Deployer`
   - **Email**: optional
   - **Status**: `Active`
   - Set a **strong password**.
4. In **Roles**, assign the role: `mich43l-libs-publisher`.
5. Save.

This user will be used by Maven (CLI or CI pipelines) to deploy artifacts.

---

## 7. Maven Client Configuration

On the machine that runs Maven (developer laptop or CI runner), configure `~/.m2/settings.xml`:

```xml
<settings>
  <servers>
    <server>
      <id>mich43l-libs</id>
      <username>ci-deployer</username>
      <password>CHANGE_ME_STRONG_PASSWORD</password>
    </server>
  </servers>
</settings>
```

> The `<id>` value (`mich43l-libs`) **must match** the `-DrepositoryId` used in Maven commands.

---

## 8. Deploying Artifacts with `maven-deploy-plugin`

### 8.1. Environment Variables

You can use environment variables to make the command shorter:

```bash
export REPO_URL="http://mynexus.repository:30081/repository/mich43l-libs"
export REPO_ID="mich43l-libs"
export SDK_HOME="/path/to/sdk"
```

### 8.2. Deploying a POM-only BOM

Example for `appblocks-bom` POM:

```bash
mvn deploy:deploy-file   -Dfile="$SDK_HOME/lib/pom/appblocks-bom.pom"   -DpomFile="$SDK_HOME/lib/pom/appblocks-bom.pom"   -Dpackaging=pom   -Durl="$REPO_URL"   -DrepositoryId="$REPO_ID"
```

### 8.3. Deploying a JAR with an existing POM

Example for `comappblock` JAR:

```bash
mvn deploy:deploy-file   -DgroupId=com.genesyslab.platform   -DartifactId=comappblock   -Dversion=900.10.33   -Dpackaging=jar   -Dfile="$SDK_HOME/lib/comappblock-900.10.33.jar"   -Durl="$REPO_URL"   -DrepositoryId="$REPO_ID"
```

If you also have a POM file:

```bash
mvn deploy:deploy-file   -DgroupId=com.genesyslab.platform   -DartifactId=comappblock   -Dversion=900.10.33   -Dpackaging=jar   -Dfile="$SDK_HOME/lib/comappblock-900.10.33.jar"   -DpomFile="$SDK_HOME/lib/pom/comappblock-900.10.33.pom"   -Durl="$REPO_URL"   -DrepositoryId="$REPO_ID"
```

---

## 9. Common Issues & Troubleshooting

### 9.1. 401 Unauthorized

**Symptom**: Maven error shows:

```text
Return code is: 401, ReasonPhrase: Unauthorized
```

**Causes & fixes**:

- Wrong username/password in `settings.xml`.
- `<id>` in `settings.xml` does not match `-DrepositoryId` in the Maven command.
- Proxy or reverse-proxy stripping `Authorization` header.

**Checklist**:

- Confirm `settings.xml` is in the correct home (the user that runs Maven).
- Confirm:

  ```xml
  <id>mich43l-libs</id>
  ```

  matches:

  ```bash
  -DrepositoryId=mich43l-libs
  ```

### 9.2. 403 Forbidden

**Symptom**:

```text
Return code is: 403, ReasonPhrase: Forbidden
```

**Causes & fixes**:

- User authenticated as `ci-deployer` but lacks privileges.
- Deploying to a **group** repo (e.g. `maven-public`) instead of a **hosted** repo.
- Role missing `nx-repository-view-maven2-mich43l-libs-add` or `-edit`.

**Checklist**:

- Confirm repo is **hosted** (`mich43l-libs`) and URL is:

  ```text
  http://mynexus.repository:30081/repository/mich43l-libs/
  ```

- Ensure the role assigned to the user has at least:

  ```text
  nx-repository-view-maven2-mich43l-libs-browse
  nx-repository-view-maven2-mich43l-libs-read
  nx-repository-view-maven2-mich43l-libs-add
  nx-repository-view-maven2-mich43l-libs-edit
  ```

### 9.3. 400: Artifact Cannot Be Updated

**Symptom**:

```text
Return code is: 400, ReasonPhrase: mich43l-libs/.../artifact-version.jar cannot be updated.
```

**Cause**:

- The artifact with the **same GAV + version** already exists.
- Repository **Deployment policy** is set to `Disable redeploy`.

**Options**:

1. Change repo **Deployment policy** to `Allow redeploy` in `mich43l-libs` settings.
2. Use a **new version** (e.g. `900.10.34`).
3. Manually delete the old component (`900.10.33`) in Nexus UI, then redeploy.

---

## 10. Notes on Repository Health & Proxy Errors

If Nexus cannot reach Sonatype cloud services through the proxy, you might see log entries like:

```text
Error fetching and caching enabled registries: Connect to rhc.sonatype.com:443 timed out
```

These affect only:

- Repository health / malware check features.

They **do not** prevent artifact storage or retrieval. To silence these:

- Ensure your proxy config is correct **OR**
- Disable the corresponding scheduled tasks under  
  **Administration → System → Tasks** (e.g. “Malicious Risk On Disk” / “Repository Health Check”).

---

## 11. Summary

- Nexus runs in Docker with outbound proxy configured via env vars.
- A hosted Maven repo `mich43l-libs` is created for internal artifacts.
- Role `mich43l-libs-publisher` grants minimal privileges to deploy artifacts.
- User `ci-deployer` is configured with that role and used by Maven via `settings.xml`.
- Artifacts are uploaded using `mvn deploy:deploy-file` targeting `mich43l-libs`.

This setup provides a clean, reproducible, and secure pattern for hosting internal Maven artifacts behind a corporate proxy.
