# dbt-docker
dbt sandbox with a docker test and transformation to production

Created inside WSL2 for better performances with Docker

# Architecture

### 🏗️ Layer 1: Infrastructure (Docker Compose)

This is the box containing everything else. Its mission is to orchestrate two virtual machines (containers) that talk to each other.

* **`database` Service**: A lightweight PostgreSQL 15-alpine server that acts as the container for your data.
* **`dbt_dev` Service**: Your dbt work environment, based on Python 3.11-slim.
* **The Network**: Docker creates an internal network where the dbt container can call the `database` container simply by its name (host: `database`).

### 📦 Layer 2: Python Environment (`uv`)

This is the internal engine of the `dbt_dev` container. We chose `uv` for its speed and strictness.

* **`pyproject.toml` & `uv.lock**`: They define the exact "recipe" of necessary libraries (dbt-core, dbt-postgres).
* **Isolation**: The environment is installed in `/app/.venv` inside the container.
* **Persistence**: The named volume `venv_data` protects this environment. Even if you shut everything down, Docker won't reinstall anything on restart because it keeps this folder safely aside.

---

### 🔗 Layer 3: The Bridge (Volumes & Configuration)

This connects your work on your PC (VS Code) to the execution inside Docker.

* **The Volume `../..:/app**`: This is a real-time mirror. You modify a `.sql` file on your Linux host, and the container sees the change instantly in its `/app` folder.
* **`profiles.yml`**: This is the connection's "ID card". It tells dbt: "For the `dev` environment, use the `root` user on the `database` server".

### ⚡ Layer 4: Transformation (dbt)

This is the business logic. dbt doesn't store anything; it gives orders.

* **Parsing**: dbt reads your `dbt_project.yml` to understand the project structure.
* **Compilation**: It transforms your Jinja/SQL code (like `ref('model')`) into pure SQL queries that PostgreSQL can understand.
* **Execution**: It sends these queries to the database to create **tables** (static data) or **views** (dynamic queries).

---

### 🛠️ Workflow Summary

| Step | Action | Component Involved |
| --- | --- | --- |
| **1. Writing** | You write SQL in VS Code | PC (Host) |
| **2. Syncing** | The file appears in the container | Docker Volume |
| **3. Command** | You type `dbt run` via `exec` | `dbt_dev` Container |
| **4. Calculation** | dbt compiles and contacts the DB | dbt + `profiles.yml` |
| **5. Storage** | The table is created in Postgres | `database` Container |

# Execution

1. project path : cd ~/projects/dbt
2. Start the containers : docker compose -f dev/docker/docker-compose.yaml up -d #fast environnement with volume venv-data
3. dbt run : docker compose -f dev/docker/docker-compose.yaml exec dbt_dev dbt run
4. Stop : docker compose -f dev/docker/docker-compose.yaml stop #shutdown containers without delete them -> fast launch for the next time

Issues:

- Issue with library, file doens't exist, hard reset:
docker compose -f dev/docker/docker-compose.yaml down -v
docker compose -f dev/docker/docker-compose.yaml up -d --build

Bonus:

to create shot command for dbt, inside ~/.bashrc or ~/.zshrc:
alias dbtrun='docker compose -f dev/docker/docker-compose.yaml exec dbt_dev dbt run'
alias dbttest='docker compose -f dev/docker/docker-compose.yaml exec dbt_dev dbt test'
