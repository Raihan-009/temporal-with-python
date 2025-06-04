# Temporal Installation with Docker Compose and Accessing Web UI on Port 8233

This guide walks you through installing Temporal locally using Docker Compose, remapping the Web UI to port 8233, and using the Temporal CLI (`tctl`) to interact with the server.

---

## 1. Prerequisites

- **Docker** (version ≥ 20.10)  
- **Docker Compose** (version ≥ 1.29)  
- **Git** (to clone the Temporal Docker Compose repo)  

Confirm that Docker and Docker Compose are installed and running:

```bash
docker --version
docker-compose --version
```

---

## 2. Clone the Official Docker Compose Repository

1. Open a terminal.
2. Clone the Temporal Docker Compose configurations:
   ```bash
   git clone https://github.com/temporalio/docker-compose.git
   cd docker-compose
   ```
3. You should now see files like `docker-compose.yml`, `README.md`, and subdirectories for different database options.

---

## 3. Update Port Mapping for the Web UI (→ 8233)

By default, the Temporal Web UI is bound to port 8080. To expose it on port 8233 instead:

1. Open `docker-compose.yml` in your editor.
2. Locate the **`temporal-web`** service section. By default, it looks like this:
   ```yaml
   temporal-web:
     image: temporalio/web:latest
     ports:
       - "8080:8080"
     environment:
       - TEMPORAL_GRPC_ENDPOINT=temporal:7233
     depends_on:
       - temporal
     healthcheck:
       test: ["CMD", "curl", "-f", "http://localhost:8080/"]
       interval: 10s
       timeout: 5s
       retries: 5
   ```
3. Change the `ports` mapping so that host 8233 → container 8080:
   ```yaml
   temporal-web:
     image: temporalio/web:latest
     ports:
       - "8233:8080"
     environment:
       - TEMPORAL_GRPC_ENDPOINT=temporal:7233
     depends_on:
       - temporal
     healthcheck:
       test: ["CMD", "curl", "-f", "http://localhost:8080/"]
       interval: 10s
       timeout: 5s
       retries: 5
   ```
4. Save `docker-compose.yml`.

---

## 4. Start Temporal via Docker Compose

1. From within the `docker-compose` directory (where `docker-compose.yml` lives), run:
   ```bash
   docker-compose up -d
   ```
   - The `-d` flag runs all services in detached mode.
   - This command will pull the necessary images and start:
     - PostgreSQL (persistence store)
     - ElasticSearch (visibility store)
     - Temporal services (frontend, history, matching, etc.)
     - Temporal Web UI (remapped to port 8233)
2. Wait a minute or so for all containers to initialize. You can monitor startup logs with:
   ```bash
   docker-compose logs -f
   ```
   When you see lines like `Temporal Web started on port:8080`, the stack is ready.

---

## 5. Access the Temporal Web UI

1. Open your browser.
2. Navigate to:
   ```
   http://localhost:8233
   ```
3. You should see the Temporal Web UI dashboard. The default namespace “default” is already registered. From here, you can:
   - Switch namespaces
   - View running or completed workflows
   - Inspect event histories
   - Send signals or terminate workflows manually

---

## 6. Install the Temporal CLI (`tctl`)

You need the Temporal CLI to register namespaces, list workflows, start workflows, etc.

### 6.1. macOS (Homebrew)

```bash
brew install temporal/tap/temporal
```

### 6.2. Linux

1. Download and extract the latest Linux release:
   ```bash
   curl -L https://github.com/temporalio/cli/releases/latest/download/temporal_linux_amd64.tar.gz      | tar -xz
   ```
2. Move the `temporal` binary to `/usr/local/bin` (or another directory in your `$PATH`):
   ```bash
   sudo mv temporal /usr/local/bin/
   ```

### 6.3. Windows

1. Visit [Temporal CLI Releases](https://github.com/temporalio/cli/releases).
2. Download the Windows `.exe` and add it to a folder in your `PATH`.

### 6.4. Verify Installation

```bash
tctl --help
# or, if using the newer binary name:
temporal --help
```

You should see a list of available subcommands (e.g., `namespace`, `workflow`, `task-queue`, etc.).

---

## 7. Point `tctl` to the Local Temporal Server

By default, `tctl` attempts to connect to `localhost:7233` over gRPC. The Docker Compose configuration exposes the Temporal frontend on port 7233, so no additional flags are required. If you ever need to override, use `--address`:

```bash
tctl --address localhost:7233 namespace list
```

---

## 8. Register a New Namespace Using `tctl`

Even though the `default` namespace exists, you can create additional namespaces. Use the `namespace register` command:

```bash
tctl namespace register   --namespace my-namespace   --retention 168h
```

- `--namespace my-namespace` → The name of your new namespace.
- `--retention 168h`      → Keep workflow history for 168 hours (7 days).

You should see output similar to:

```
Namespace 'my-namespace' successfully registered.
```

To confirm:

```bash
tctl namespace list
```

You’ll see both `default` and `my-namespace`, along with their retention settings.

---

## 9. Switching Namespaces in the Web UI

1. In the Temporal Web UI (`http://localhost:8233`), click the **Namespace** dropdown (top-left).
2. Select `my-namespace` (or any namespace you’ve registered).
3. Now any workflows you start or view will be scoped to that namespace.

---

## 10. Basic `tctl` Commands

Once your namespace exists, you can:

- **List All Workflows** (in a namespace):
  ```bash
  tctl --namespace my-namespace workflow list
  ```
- **Describe a Specific Namespace**:
  ```bash
  tctl namespace describe --namespace my-namespace
  ```
- **Disable a Namespace** (no new workflows allowed):
  ```bash
  tctl namespace update --namespace my-namespace --disable
  ```
- **Re-enable a Disabled Namespace**:
  ```bash
  tctl namespace update --namespace my-namespace --enable
  ```
- **Update Retention Period**:
  ```bash
  tctl namespace update     --namespace my-namespace     --retention 336h
  ```

---

## 11. Stopping and Cleaning Up

To stop all Temporal services:

```bash
docker-compose down
```

By default, `docker-compose down` will remove containers and networks but will preserve the PostgreSQL and Elasticsearch volumes. If you want to remove volumes as well:

```bash
docker-compose down --volumes
```

---

## Summary

1. **Clone** the official Temporal Docker Compose repo.  
2. **Modify** `docker-compose.yml` to remap the Web UI from port 8080 → 8233.  
3. **Start** the stack with `docker-compose up -d`.  
4. **Access** the Temporal Web UI at `http://localhost:8233`.  
5. **Install** the Temporal CLI (`tctl`/`temporal`) on your host machine.  
6. **Use** `tctl namespace register …` to create and manage namespaces.  
7. **Switch** namespaces in the Web UI dropdown to scope workflows.  

You now have a fully functional local Temporal environment—complete with a Web UI on port 8233 and a working `tctl` CLI. Happy workflow building!
