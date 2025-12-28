# LangGraph CLI Documentation

## Overview

The LangGraph CLI is the official command-line interface for creating, developing, building, and deploying LangGraph applications. It provides a streamlined workflow from project scaffolding to production deployment.

**Current Version:** 0.4.11

### Installation

Basic installation:
```bash
pip install langgraph-cli
```

For development mode with in-memory server and hot reloading (requires Python 3.11+):
```bash
pip install "langgraph-cli[inmem]"
```

### Requirements

- **Docker**: Required for `langgraph up`, `langgraph build`, and `langgraph dockerfile`
- **Python 3.11+**: Required for `langgraph dev` (in-memory development server)
- **Docker Compose**: Either as a plugin or standalone installation

## Command Reference

### `langgraph new` - Project Scaffolding

Create a new LangGraph project from a predefined template.

**Usage:**
```bash
langgraph new [PATH] [OPTIONS]
```

**Options:**
- `PATH` - Directory path where the project will be created (optional, will prompt if not provided)
- `--template TEXT` - Template identifier to use (optional, interactive selection if not provided)

**Examples:**
```bash
# Interactive mode - prompts for path and template selection
langgraph new

# Specify path, interactive template selection
langgraph new ./my-agent

# Use specific template
langgraph new ./my-agent --template react-agent-python
```

#### Available Templates

The CLI provides several production-ready templates:

| Template ID | Description | Languages |
|------------|-------------|-----------|
| `new-langgraph-project-python` | A simple, minimal chatbot with memory | Python |
| `new-langgraph-project-js` | A simple, minimal chatbot with memory | JavaScript/TypeScript |
| `react-agent-python` | A simple agent that can be flexibly extended to many tools | Python |
| `react-agent-js` | A simple agent that can be flexibly extended to many tools | JavaScript/TypeScript |
| `memory-agent-python` | ReAct-style agent with additional tool to store memories across threads | Python |
| `memory-agent-js` | ReAct-style agent with additional tool to store memories across threads | JavaScript/TypeScript |
| `retrieval-agent-python` | Agent with retrieval-based question-answering system | Python |
| `retrieval-agent-js` | Agent with retrieval-based question-answering system | JavaScript/TypeScript |
| `data-enrichment-agent-python` | Agent that performs web searches and organizes findings | Python |
| `data-enrichment-agent-js` | Agent that performs web searches and organizes findings | JavaScript/TypeScript |

#### Interactive Selection

When running without `--template`, the CLI presents an interactive menu:

1. Select template type (numbered list with descriptions)
2. Choose language (1 for Python, 2 for JavaScript/TypeScript)

The selected template is downloaded from GitHub and extracted to the specified path.

#### Project Structure

Templates typically include:
- `langgraph.json` - Configuration file
- Source code for the graph implementation
- `requirements.txt` or `package.json` - Dependencies
- `.env.example` - Environment variable template
- `README.md` - Project documentation

---

### `langgraph dev` - Development Server

Run the LangGraph API server locally with hot reloading for rapid development. This command uses an in-memory server that doesn't require Docker.

**Requirements:** Python 3.11+ and `langgraph-cli[inmem]` installation

**Usage:**
```bash
langgraph dev [OPTIONS]
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--host` | TEXT | 127.0.0.1 | Network interface to bind to. Use 0.0.0.0 only in trusted networks |
| `--port` | INTEGER | 2024 | Port number for the development server |
| `--no-reload` | FLAG | False | Disable automatic reloading on code changes |
| `--config` | PATH | langgraph.json | Path to configuration file |
| `--n-jobs-per-worker` | INTEGER | 10 | Maximum concurrent jobs per worker process |
| `--no-browser` | FLAG | False | Skip automatically opening browser |
| `--debug-port` | INTEGER | None | Enable remote debugging on specified port (requires debugpy) |
| `--wait-for-client` | FLAG | False | Wait for debugger client before starting |
| `--studio-url` | TEXT | https://smith.langchain.com | LangGraph Studio URL |
| `--allow-blocking` | FLAG | False | Don't raise errors for synchronous I/O operations |
| `--tunnel` | FLAG | False | Expose server via Cloudflare tunnel for remote access |
| `--server-log-level` | TEXT | WARNING | API server log level (DEBUG, INFO, WARNING, ERROR) |

**Examples:**

```bash
# Basic development server
langgraph dev

# Custom port and host
langgraph dev --port 8000 --host 0.0.0.0

# With remote debugging enabled
langgraph dev --debug-port 5678 --wait-for-client

# Disable hot reload
langgraph dev --no-reload

# Enable tunnel for remote access
langgraph dev --tunnel

# Verbose logging
langgraph dev --server-log-level DEBUG
```

#### Hot Reload Behavior

The development server automatically watches for file changes and reloads when:
- Python files in the working directory change
- The `langgraph.json` configuration is modified
- Dependencies listed in the config are updated

Hot reload is enabled by default. Use `--no-reload` to disable this behavior.

#### Environment Variables

The dev server loads environment variables from:
1. System environment
2. `.env` file (if specified in `langgraph.json`)
3. Inline environment variables (if specified in `langgraph.json`)

#### Browser Integration

By default, the server:
- Opens the default browser automatically
- Points to LangGraph Studio at the configured `--studio-url`
- Passes the local API base URL as a query parameter

Access points:
- **API**: http://localhost:2024
- **Docs**: http://localhost:2024/docs
- **Studio**: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

Use `--no-browser` to disable automatic browser opening.

#### Debugging Support

Enable remote debugging with `--debug-port`:

```bash
langgraph dev --debug-port 5678 --wait-for-client
```

This requires `debugpy` to be installed in your environment. The server will wait for a debugger to attach before starting when `--wait-for-client` is used.

**VS Code launch.json example:**
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to LangGraph",
            "type": "python",
            "request": "attach",
            "connect": {
                "host": "localhost",
                "port": 5678
            }
        }
    ]
}
```

#### Limitations

- **JavaScript/TypeScript graphs**: Not supported in dev mode. Use `npx @langchain/langgraph-cli` for JS/TS projects
- **Python version**: Requires Python 3.11 or higher
- **In-memory only**: Does not persist state across restarts (use `langgraph up` for persistent storage)

---

### `langgraph build` - Building Docker Images

Build a Docker image for your LangGraph application with all dependencies included.

**Usage:**
```bash
langgraph build -t IMAGE_TAG [OPTIONS] [DOCKER_BUILD_ARGS]
```

**Required Options:**
- `-t, --tag TEXT` - Docker image tag (required)

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `-c, --config` | PATH | langgraph.json | Configuration file path |
| `--pull / --no-pull` | BOOLEAN | True | Pull latest base images before building |
| `--base-image` | TEXT | Auto-detected | Base image for the server (e.g., langchain/langgraph-api:0.2.18) |
| `--api-version` | TEXT | Latest | API server version to use |
| `--install-command` | TEXT | Auto-detected | Custom install command for JS projects |
| `--build-command` | TEXT | Auto-detected | Custom build command for JS projects |

**Additional Docker Build Args:**
Any additional arguments are passed directly to `docker build`. Common examples:
- `--platform linux/amd64,linux/arm64` - Multi-platform builds
- `--build-arg KEY=VALUE` - Build-time variables
- `--no-cache` - Disable build cache
- `--progress plain` - Show detailed build output

**Examples:**

```bash
# Basic build
langgraph build -t my-agent:latest

# Pin to specific API version
langgraph build -t my-agent:v1.0 --api-version 0.2.18

# Multi-platform build
langgraph build -t my-agent:latest --platform linux/amd64,linux/arm64

# Use local base image (no pull)
langgraph build -t my-agent:dev --no-pull

# Custom base image
langgraph build -t my-agent:latest --base-image langchain/langgraph-server:0.2.18

# With Docker build arguments
langgraph build -t my-agent:latest --build-arg HTTP_PROXY=http://proxy:8080

# Show detailed build output
langgraph build -t my-agent:latest --progress plain

# JavaScript project with custom commands
langgraph build -t my-js-agent:latest \
  --install-command "pnpm install --frozen-lockfile" \
  --build-command "pnpm build"
```

#### Build Process

The build command:

1. **Validates** the `langgraph.json` configuration
2. **Pulls** the base image (unless `--no-pull`)
3. **Generates** a Dockerfile from the configuration
4. **Copies** local dependencies into the image
5. **Installs** Python/Node.js dependencies
6. **Configures** environment variables and graphs
7. **Removes** build tools (pip, setuptools, wheel) to reduce image size

#### Multi-Stage Build

For Python projects, the CLI automatically creates an optimized multi-stage build:

```dockerfile
FROM langchain/langgraph-api:3.11
# Install dependencies
RUN pip install langchain_openai
# Copy local packages
ADD ./my_package /deps/my_package
# Install local packages
RUN pip install -e /deps/my_package
# Set environment variables
ENV LANGSERVE_GRAPHS='{"my_graph": "./my_package/graph.py:graph"}'
# Clean up build tools
RUN pip uninstall -y pip setuptools wheel
```

#### Base Image Selection

The CLI automatically selects the appropriate base image:

- **Python projects**: `langchain/langgraph-api` (default)
- **JavaScript projects**: `langchain/langgraphjs-api`
- **Custom**: Use `--base-image` to override

Base images support version pinning:
```bash
# Pin to specific patch version
--base-image langchain/langgraph-api:0.2.18

# Pin to minor version (gets latest patch)
--base-image langchain/langgraph-api:0.2
```

#### Image Distros

Supported Linux distributions (configured in `langgraph.json`):
- `debian` (default)
- `wolfi`
- `bullseye`
- `bookworm`

#### Package Manager Detection

For JavaScript projects, the CLI auto-detects the package manager:

| Lockfile | Package Manager | Install Command |
|----------|----------------|-----------------|
| `package-lock.json` | npm | `npm ci` |
| `yarn.lock` | yarn | `yarn install --frozen-lockfile` |
| `pnpm-lock.yaml` | pnpm | `pnpm i --frozen-lockfile` |
| `bun.lockb` | bun | `bun i` |
| `package.json` only | npm | `npm i` |

Override with `--install-command` if needed.

---

### `langgraph up` - Running with Docker Compose

Launch the LangGraph API server with all required services (PostgreSQL, Redis) using Docker Compose.

**Usage:**
```bash
langgraph up [OPTIONS]
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `-c, --config` | PATH | langgraph.json | Configuration file path |
| `-p, --port` | INTEGER | 8123 | Port to expose the API |
| `--pull / --no-pull` | BOOLEAN | True | Pull latest images before starting |
| `--recreate / --no-recreate` | BOOLEAN | False | Force recreate containers and volumes |
| `--watch` | FLAG | False | Restart on file changes |
| `--wait` | FLAG | False | Wait for services to start (implies --detach) |
| `--verbose` | FLAG | False | Show detailed server logs |
| `-d, --docker-compose` | PATH | None | Path to additional docker-compose.yml |
| `--debugger-port` | INTEGER | None | Serve LangGraph Studio locally on specified port |
| `--debugger-base-url` | TEXT | http://127.0.0.1:PORT | Base URL for debugger to access API |
| `--postgres-uri` | TEXT | Auto-managed | Custom PostgreSQL connection URI |
| `--image` | TEXT | Built on-the-fly | Use pre-built image instead of building |
| `--base-image` | TEXT | Auto-detected | Base image to use |
| `--api-version` | TEXT | Latest | API version to use |

**Examples:**

```bash
# Basic startup
langgraph up

# Custom port
langgraph up --port 8000

# Watch mode - auto-restart on changes
langgraph up --watch

# Use pre-built image
langgraph up --image my-agent:v1.0

# Include local LangGraph Studio
langgraph up --debugger-port 8080

# Force fresh start
langgraph up --recreate

# Use external PostgreSQL
langgraph up --postgres-uri postgresql://user:pass@db.example.com:5432/mydb

# Combine with custom docker-compose
langgraph up --docker-compose ./docker-compose.extra.yml

# Verbose logging for debugging
langgraph up --verbose

# Wait for services to be healthy before returning
langgraph up --wait
```

#### Service Architecture

The `langgraph up` command orchestrates multiple Docker services:

**Core Services:**

1. **langgraph-api** - Main API server
   - Ports: `8123:8000` (configurable via `--port`)
   - Depends on: postgres, redis
   - Environment: `POSTGRES_URI`, `REDIS_URI`

2. **langgraph-postgres** - PostgreSQL database with pgvector
   - Image: `pgvector/pgvector:pg16`
   - Ports: `5433:5432` (internal only)
   - Volume: `langgraph-data` (persistent)
   - Credentials: `postgres:postgres:postgres`

3. **langgraph-redis** - Redis for task queue
   - Image: `redis:6`
   - Internal networking only
   - Healthcheck: `redis-cli ping`

**Optional Services:**

4. **langgraph-debugger** - Local LangGraph Studio (if `--debugger-port` specified)
   - Image: `langchain/langgraph-debugger`
   - Ports: Custom port mapped to `3968`
   - Connects to langgraph-api

#### Port Mappings

Default port configuration:

```yaml
services:
  langgraph-api:
    ports:
      - "8123:8000"    # API server (configurable)
  langgraph-postgres:
    ports:
      - "5433:5432"    # PostgreSQL (for external access)
  langgraph-debugger:
    ports:
      - "3968:3968"    # Studio (if enabled)
```

Access points:
- **API**: http://localhost:8123
- **API Docs**: http://localhost:8123/docs
- **PostgreSQL**: postgresql://postgres:postgres@localhost:5433/postgres
- **Studio**: http://localhost:3968 (if debugger enabled) or https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:8123

#### Volume Mounts

**Persistent Data:**
```yaml
volumes:
  langgraph-data:
    driver: local
```

This volume stores:
- PostgreSQL database files
- Checkpoint data
- Thread states
- Store data

**Development Mode with --watch:**
```yaml
develop:
  watch:
    - path: langgraph.json
      action: rebuild
    - path: ./my_package
      action: rebuild
```

When `--watch` is enabled, the CLI monitors:
- `langgraph.json` configuration
- All local dependencies (paths starting with `.`)

Changes trigger automatic container rebuild and restart.

#### Environment Variables

The API service receives these environment variables:

**Required (auto-configured):**
- `POSTGRES_URI` - Database connection string
- `REDIS_URI` - Redis connection string
- `LANGSERVE_GRAPHS` - JSON-encoded graph definitions

**Optional (from langgraph.json):**
- Custom environment variables from `env` field
- Authentication configuration
- Store configuration
- HTTP configuration

**License/API Key (required for production):**
- `LANGSMITH_API_KEY` - For LangSmith Deployment (development)
- `LANGGRAPH_CLOUD_LICENSE_KEY` - For production use

#### Using Pre-built Images

Skip the build step by using `--image`:

```bash
# Build once
langgraph build -t my-agent:v1.0

# Run multiple times without rebuilding
langgraph up --image my-agent:v1.0
```

This is useful for:
- Testing built images locally
- Faster startup times
- Consistency with CI/CD builds

#### Custom Docker Compose Files

Extend the default services with `--docker-compose`:

```bash
langgraph up --docker-compose ./docker-compose.extra.yml
```

**Example extra services:**

```yaml
# docker-compose.extra.yml
services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - langgraph-api
```

The CLI merges these services with the auto-generated configuration.

#### Healthchecks

All services include healthchecks:

**PostgreSQL:**
```yaml
healthcheck:
  test: pg_isready -U postgres
  interval: 5s (or 60s with start_interval on Docker 25+)
  start_interval: 1s (Docker 25+)
  start_period: 10s
  timeout: 1s
  retries: 5
```

**Redis:**
```yaml
healthcheck:
  test: redis-cli ping
  interval: 5s
  timeout: 1s
  retries: 5
```

**API Server (Docker 25+):**
```yaml
healthcheck:
  test: python /api/healthcheck.py
  interval: 60s
  start_interval: 1s
  start_period: 10s
```

The API service waits for database and Redis to be healthy before starting.

#### Cleanup and Reset

**Recreate everything:**
```bash
langgraph up --recreate
```

This:
- Stops and removes all containers
- Deletes the `langgraph-data` volume
- Rebuilds from scratch

**Manual cleanup:**
```bash
# Stop services
docker compose down

# Remove volumes
docker volume rm langgraph-data

# Remove images
docker rmi $(docker images -q 'langgraph-api*')
```

---

### `langgraph dockerfile` - Dockerfile Generation

Generate a Dockerfile for manual builds or custom deployment pipelines.

**Usage:**
```bash
langgraph dockerfile SAVE_PATH [OPTIONS]
```

**Arguments:**
- `SAVE_PATH` - Where to save the generated Dockerfile (e.g., `./Dockerfile`)

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `-c, --config` | PATH | langgraph.json | Configuration file path |
| `--add-docker-compose` | FLAG | False | Also generate docker-compose.yml, .env, and .dockerignore |
| `--base-image` | TEXT | Auto-detected | Base image to use |
| `--api-version` | TEXT | Latest | API version to use |

**Examples:**

```bash
# Generate Dockerfile only
langgraph dockerfile ./Dockerfile

# Generate complete Docker setup
langgraph dockerfile ./Dockerfile --add-docker-compose

# Specify API version
langgraph dockerfile ./Dockerfile --api-version 0.2.18

# Custom base image
langgraph dockerfile ./Dockerfile --base-image langchain/langgraph-api:py3.11
```

#### Generated Files

**With `--add-docker-compose` flag:**

1. **Dockerfile** - Multi-stage build for the application
2. **docker-compose.yml** - Complete service orchestration
3. **.dockerignore** - Files to exclude from build context
4. **.env** - Environment variable template (if doesn't exist)

#### Dockerfile Structure

The generated Dockerfile includes:

**Python projects:**
```dockerfile
FROM langchain/langgraph-api:3.11

# Custom Dockerfile lines (if any)
RUN apt-get update && apt-get install -y libmagic-dev

# Install PyPI dependencies
RUN pip install langchain_openai anthropic

# Install local requirements
ADD ./requirements.txt /deps/my_package/requirements.txt
RUN pip install -r /deps/my_package/requirements.txt

# Copy local packages
ADD ./my_package /deps/my_package

# Install local packages
RUN pip install -e /deps/my_package

# Set environment variables
ENV LANGSERVE_GRAPHS='{"my_graph": "./my_package/graph.py:graph"}'

# Clean up build tools
RUN pip uninstall -y pip setuptools wheel

WORKDIR /deps/my_package
```

**JavaScript/TypeScript projects:**
```dockerfile
FROM langchain/langgraphjs-api:20

# Copy project files
ADD . /deps/my_project

# Install dependencies
RUN cd /deps/my_project && npm ci

# Set environment variables
ENV LANGSERVE_GRAPHS='{"my_graph": "./src/graph.ts:graph"}'

WORKDIR /deps/my_project

# Build project
RUN tsx /api/langgraph_api/js/build.mts
```

#### Generated docker-compose.yml

When using `--add-docker-compose`, the CLI generates a complete docker-compose.yml:

```yaml
services:
  langgraph-redis:
    image: redis:6
    healthcheck:
      test: redis-cli ping
      interval: 5s
      timeout: 1s
      retries: 5

  langgraph-postgres:
    image: pgvector/pgvector:pg16
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - langgraph-data:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready -U postgres
      interval: 5s
      timeout: 1s
      retries: 5

  langgraph-api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8123:8000"
    env_file:
      - .env
    depends_on:
      langgraph-redis:
        condition: service_healthy
      langgraph-postgres:
        condition: service_healthy
    environment:
      REDIS_URI: redis://langgraph-redis:6379
      POSTGRES_URI: postgres://postgres:postgres@langgraph-postgres:5432/postgres?sslmode=disable

volumes:
  langgraph-data:
    driver: local
```

#### Generated .dockerignore

```dockerignore
# Dependency directories
node_modules
bower_components
vendor

# Logs and temporary files
*.log
*.tmp
*.swp

# Environment files
.env
.env.*
*.local

# Git files
.git
.gitignore

# Docker files
.dockerignore
docker-compose.yml

# Build and cache directories
dist
build
.cache
__pycache__

# IDE configurations
.vscode
.idea
*.sublime-project
*.sublime-workspace
.DS_Store

# Test files
coverage
*.coverage
*.test.js
*.spec.js
tests
```

#### Generated .env Template

```bash
# Uncomment the following line to add your LangSmith API key
# LANGSMITH_API_KEY=your-api-key

# Or if you have a LangSmith Deployment license key, then uncomment the following line:
# LANGGRAPH_CLOUD_LICENSE_KEY=your-license-key

# Add any other environment variables go below...
```

#### Using Generated Files

After generating the files:

**Build the image:**
```bash
docker build -t my-agent:latest .
```

**Run with docker-compose:**
```bash
docker compose up
```

**Or run manually:**
```bash
docker run -p 8000:8000 \
  -e LANGSMITH_API_KEY=your-key \
  -e POSTGRES_URI=postgresql://... \
  -e REDIS_URI=redis://... \
  my-agent:latest
```

#### Additional Build Contexts

For projects with dependencies in parent directories, the CLI generates additional build contexts:

```bash
# Displayed in CLI output
Run docker build with these additional build contexts `--build-context outer-my_lib=../my_lib`

# Use in build command
docker build \
  --build-context outer-my_lib=../my_lib \
  -t my-agent:latest .
```

This is needed when `langgraph.json` references dependencies like `../shared_lib`.

---

## Configuration File: langgraph.json

The `langgraph.json` file is the central configuration for LangGraph CLI commands. It defines dependencies, graphs, environment variables, and deployment settings.

### Schema Overview

```typescript
{
  // Runtime versions
  "python_version"?: "3.11" | "3.12" | "3.13",
  "node_version"?: "20" | "21" | "22",

  // API server version
  "api_version"?: "0.2.18" | "0.2" | "latest",

  // Dependencies
  "dependencies": string[],

  // Graph definitions
  "graphs": {
    [graphId: string]: string | GraphConfig
  },

  // Environment configuration
  "env"?: string | { [key: string]: string },

  // Advanced configuration
  "base_image"?: string,
  "image_distro"?: "debian" | "wolfi" | "bullseye" | "bookworm",
  "pip_config_file"?: string,
  "pip_installer"?: "auto" | "pip" | "uv",
  "dockerfile_lines"?: string[],
  "keep_pkg_tools"?: boolean | string[],

  // Service configuration
  "store"?: StoreConfig,
  "checkpointer"?: CheckpointerConfig,
  "auth"?: AuthConfig,
  "encryption"?: EncryptionConfig,
  "http"?: HttpConfig,
  "webhooks"?: WebhooksConfig,
  "ui"?: { [key: string]: string }
}
```

### Required Fields

#### dependencies

List of Python/Node.js packages to install.

**Syntax:**
```json
{
  "dependencies": [
    ".",                              // Current directory (local package)
    "./my_package",                   // Local directory
    "langchain_openai",               // PyPI/npm package
    "anthropic>=0.25.0",              // Version constraint
    "git+https://github.com/..."      // Git repository
  ]
}
```

**Local dependencies:**
- Must start with `.` or `./`
- Can be relative paths to directories
- Should contain Python packages or Node.js projects

**Python packages:**
- Installed via pip or uv
- Support standard pip version specifiers

**Node.js packages:**
- Specified in package.json dependencies
- CLI reads package.json for install

**Important:** Python projects must have at least one dependency. Use `"."` if you only have local code.

#### graphs

Mapping from graph IDs to graph implementations.

**String format:**
```json
{
  "graphs": {
    "agent": "./my_agent/graph.py:agent",
    "chatbot": "./chatbot.py:compiled_graph"
  }
}
```

**Object format (with options):**
```json
{
  "graphs": {
    "agent": {
      "path": "./my_agent/graph.py:agent",
      "description": "Customer service agent"
    }
  }
}
```

**Path syntax:** `<file_path>:<attribute_name>`
- `<file_path>`: Relative path to Python/JS file
- `<attribute_name>`: Variable name of the compiled graph

**Supported graph types:**
- `StateGraph` (compiled)
- `MessageGraph` (compiled)
- `@entrypoint` decorated functions
- Any `Pregel` object
- Context managers returning graphs

**Example graph file:**

```python
# my_agent/graph.py
from langgraph.graph import StateGraph, MessagesState

builder = StateGraph(MessagesState)
# ... add nodes and edges ...
agent = builder.compile()  # This is what graph.py:agent references
```

### Optional Fields

#### python_version

Python runtime version for the container.

**Valid values:** `"3.11"`, `"3.12"`, `"3.13"`

**Default:** `"3.11"` (if Python graphs detected)

**Example:**
```json
{
  "python_version": "3.12",
  "dependencies": ["."],
  "graphs": { "agent": "./agent.py:graph" }
}
```

**Notes:**
- Minimum version: 3.11
- Only major.minor allowed (no patch version)
- Must have Python dependencies to use this field

#### node_version

Node.js runtime version for the container.

**Valid values:** `"20"`, `"21"`, `"22"`

**Default:** `"20"` (if JavaScript graphs detected)

**Example:**
```json
{
  "node_version": "20",
  "dependencies": ["."],
  "graphs": { "agent": "./src/agent.ts:graph" }
}
```

**Notes:**
- Minimum version: 20
- Only major version (no minor/patch)
- Auto-detected from graph file extensions (.ts, .js, .mjs, etc.)

#### api_version

LangGraph API server version.

**Format:** Semantic version or partial version

**Examples:**
```json
// Pin to exact version
{ "api_version": "0.2.18" }

// Pin to minor version (gets latest patch)
{ "api_version": "0.2" }

// Use latest (default)
{ "api_version": null }
```

**Image tag resolution:**
```
api_version: "0.2.18" -> langchain/langgraph-api:0.2.18-py3.11
api_version: "0.2"    -> langchain/langgraph-api:0.2-py3.11
api_version: null     -> langchain/langgraph-api:3.11
```

#### env

Environment variables for the application.

**String format (file path):**
```json
{
  "env": "./.env"
}
```

The file should contain KEY=VALUE pairs:
```bash
# .env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
DEBUG=true
```

**Object format (inline):**
```json
{
  "env": {
    "OPENAI_API_KEY": "sk-...",
    "DEBUG": "true",
    "MAX_RETRIES": "3"
  }
}
```

**Notes:**
- File path is relative to langgraph.json location
- Inline values are embedded in the Docker image (less secure)
- For production, use secrets management or environment injection

#### base_image

Custom base image for the container.

**Default values:**
- Python: `langchain/langgraph-api`
- JavaScript: `langchain/langgraphjs-api`

**Examples:**
```json
// Use specific version
{ "base_image": "langchain/langgraph-api:0.2.18" }

// Use custom registry
{ "base_image": "myregistry.com/langgraph-api:custom" }

// Pin to minor version
{ "base_image": "langchain/langgraph-server:0.2" }
```

#### image_distro

Linux distribution for the base image.

**Valid values:** `"debian"`, `"wolfi"`, `"bullseye"`, `"bookworm"`

**Default:** `"debian"`

**Example:**
```json
{
  "image_distro": "wolfi"
}
```

**Impact on image tag:**
```
image_distro: "debian"   -> langchain/langgraph-api:3.11
image_distro: "wolfi"    -> langchain/langgraph-api:3.11-wolfi
image_distro: "bullseye" -> langchain/langgraph-api:3.11-bullseye
```

#### pip_config_file

Path to pip configuration file for custom package indices or credentials.

**Example:**
```json
{
  "pip_config_file": "./pip.conf"
}
```

**pip.conf example:**
```ini
[global]
index-url = https://pypi.company.com/simple
trusted-host = pypi.company.com
```

**Use cases:**
- Private PyPI mirror
- Corporate package repositories
- Custom SSL certificates

#### pip_installer

Python package installer to use.

**Valid values:** `"auto"`, `"pip"`, `"uv"`

**Default:** `"auto"`

**Behavior:**
- `"auto"`: Use uv if base image supports it (>= 0.2.47), otherwise pip
- `"pip"`: Force use of pip
- `"uv"`: Force use of uv (fails if unsupported)

**Example:**
```json
{
  "pip_installer": "uv"
}
```

**Benefits of uv:**
- Faster dependency resolution
- Better caching
- Improved error messages

#### dockerfile_lines

Additional Dockerfile instructions to include.

**Example:**
```json
{
  "dockerfile_lines": [
    "RUN apt-get update && apt-get install -y libmagic-dev",
    "ENV CUSTOM_VAR=value",
    "RUN pip install some-system-package"
  ]
}
```

**Common use cases:**
- Install system packages (e.g., libmagic, ffmpeg)
- Set environment variables
- Add custom certificates
- Configure timezone

**Placement:** These lines are inserted after the FROM statement but before dependency installation.

#### keep_pkg_tools

Control whether to keep Python packaging tools in the final image.

**Valid values:**
- `true`: Keep all tools (pip, setuptools, wheel)
- `false` or omitted: Remove all tools (default)
- Array: Keep specific tools

**Examples:**
```json
// Keep all tools
{ "keep_pkg_tools": true }

// Remove all tools (default)
{ "keep_pkg_tools": false }

// Keep only pip
{ "keep_pkg_tools": ["pip"] }

// Keep pip and setuptools
{ "keep_pkg_tools": ["pip", "setuptools"] }
```

**Impact on image size:**
Removing tools reduces image size by 50-100MB but prevents runtime package installation.

### Service Configuration

#### store

Configuration for long-term memory store with optional semantic search.

**Example:**
```json
{
  "store": {
    "index": {
      "dims": 1536,
      "embed": "openai:text-embedding-3-small",
      "fields": ["title", "content"]
    },
    "ttl": {
      "refresh_on_read": true,
      "default_ttl": 1440,
      "sweep_interval_minutes": 60
    }
  }
}
```

**Fields:**

- `index.dims` (required): Embedding dimension (must match model output)
- `index.embed` (required): Embedding model identifier
  - Format: `"<provider>:<model>"` or `"path/to/file.py:function"`
  - Examples: `"openai:text-embedding-3-large"`, `"./embed.py:embed_fn"`
- `index.fields` (optional): JSON fields to extract for embedding
  - Default: `["$"]` (entire object)
- `ttl.refresh_on_read` (optional): Refresh TTL on GET/SEARCH (default: true)
- `ttl.default_ttl` (optional): Default TTL in minutes for new items
- `ttl.sweep_interval_minutes` (optional): Cleanup interval

**Common embedding models:**
- `openai:text-embedding-3-large` (dims: 3072)
- `openai:text-embedding-3-small` (dims: 1536)
- `openai:text-embedding-ada-002` (dims: 1536)
- `cohere:embed-english-v3.0` (dims: 1024)
- `cohere:embed-multilingual-v3.0` (dims: 1024)

#### checkpointer

Configuration for state checkpointing with TTL.

**Example:**
```json
{
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "default_ttl": 10080,
      "sweep_interval_minutes": 60
    },
    "serde": {
      "allowed_json_modules": [
        ["my_app", "models", "CustomClass"]
      ],
      "pickle_fallback": true
    }
  }
}
```

**Fields:**

- `ttl.strategy`: Deletion strategy (currently only `"delete"`)
- `ttl.default_ttl`: TTL in minutes for checkpoint data
- `ttl.sweep_interval_minutes`: Cleanup interval
- `serde.allowed_json_modules`: Python modules allowed for deserialization
  - Format: `[["module", "submodule", "Class"]]`
  - Use `true` to allow all modules
- `serde.pickle_fallback`: Allow pickle for unknown types (default: true)

#### auth

Custom authentication configuration.

**Example:**
```json
{
  "auth": {
    "path": "./auth.py:my_auth",
    "disable_studio_auth": false,
    "openapi": {
      "securitySchemes": {
        "OAuth2": {
          "type": "oauth2",
          "flows": {
            "password": {
              "tokenUrl": "/token",
              "scopes": {
                "read": "Read access",
                "write": "Write access"
              }
            }
          }
        }
      },
      "security": [{"OAuth2": ["read", "write"]}]
    },
    "cache": {
      "cache_keys": ["user_id"],
      "ttl_seconds": 3600,
      "max_size": 1000
    }
  }
}
```

**Auth implementation example:**

```python
# auth.py
from langgraph_api.auth import Auth, get_current_user
from typing import Optional

class MyAuth(Auth):
    async def authenticate(self, request):
        # Custom authentication logic
        token = request.headers.get("Authorization")
        user = await verify_token(token)
        return user

    async def authorize(self, request, user, resource):
        # Custom authorization logic
        return user.has_permission(resource)

my_auth = MyAuth()
```

#### encryption

Custom at-rest encryption for sensitive data.

**Example:**
```json
{
  "encryption": {
    "path": "./encryption.py:my_encryption"
  }
}
```

**Encryption implementation example:**

```python
# encryption.py
from langgraph_api.encryption import Encryption
from cryptography.fernet import Fernet

class MyEncryption(Encryption):
    def __init__(self):
        self.cipher = Fernet(ENCRYPTION_KEY)

    async def encrypt(self, data: bytes) -> bytes:
        return self.cipher.encrypt(data)

    async def decrypt(self, data: bytes) -> bytes:
        return self.cipher.decrypt(data)

my_encryption = MyEncryption()
```

#### http

HTTP server configuration.

**Example:**
```json
{
  "http": {
    "app": "./custom_app.py:app",
    "disable_assistants": false,
    "disable_threads": false,
    "disable_runs": false,
    "disable_store": false,
    "disable_mcp": false,
    "disable_a2a": false,
    "disable_meta": false,
    "disable_ui": false,
    "disable_webhooks": false,
    "mount_prefix": "/api",
    "enable_custom_route_auth": true,
    "middleware_order": "middleware_first",
    "cors": {
      "allow_origins": ["https://app.example.com"],
      "allow_methods": ["GET", "POST", "PUT", "DELETE"],
      "allow_headers": ["Content-Type", "Authorization"],
      "allow_credentials": true,
      "max_age": 3600
    },
    "configurable_headers": {
      "includes": ["x-user-id", "x-tenant-*"],
      "excludes": ["*key*", "*token*"]
    },
    "logging_headers": {
      "excludes": ["authorization", "x-api-key"]
    }
  }
}
```

**Custom app example:**

```python
# custom_app.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/custom")
async def custom_endpoint():
    return {"message": "Custom endpoint"}

@app.post("/webhook")
async def webhook_handler(data: dict):
    # Handle webhook
    return {"status": "processed"}
```

#### webhooks

Webhook delivery configuration.

**Example:**
```json
{
  "webhooks": {
    "env_prefix": "WEBHOOK_",
    "url": {
      "require_https": true,
      "allowed_domains": ["*.mycompany.com", "hooks.example.com"],
      "allowed_ports": [443, 8443],
      "max_url_length": 2048,
      "disable_loopback": false
    },
    "headers": {
      "X-Webhook-Secret": "${{ env.WEBHOOK_SECRET }}",
      "X-API-Key": "${{ env.WEBHOOK_API_KEY }}"
    }
  }
}
```

**Environment variable template resolution:**
```bash
# .env
WEBHOOK_SECRET=my-secret-123
WEBHOOK_API_KEY=api-key-456

# Results in headers:
# X-Webhook-Secret: my-secret-123
# X-API-Key: api-key-456
```

#### ui

Custom UI component definitions.

**Example:**
```json
{
  "ui": {
    "custom_view": "./ui/CustomView.tsx:default",
    "dashboard": "./ui/Dashboard.tsx:Dashboard"
  }
}
```

**UI component example:**

```typescript
// ui/CustomView.tsx
import React from 'react';

export default function CustomView({ state }) {
  return (
    <div>
      <h1>Custom View</h1>
      <pre>{JSON.stringify(state, null, 2)}</pre>
    </div>
  );
}
```

### Complete Configuration Examples

#### Simple Python Chatbot

```json
{
  "dependencies": [
    ".",
    "langchain_openai"
  ],
  "graphs": {
    "chatbot": "./chatbot.py:graph"
  },
  "env": "./.env"
}
```

#### Production Agent with Authentication

```json
{
  "python_version": "3.12",
  "api_version": "0.2.18",
  "dependencies": [
    ".",
    "langchain_openai>=0.1.0",
    "langchain_community>=0.2.0"
  ],
  "graphs": {
    "agent": "./agent/graph.py:compiled_graph"
  },
  "env": "./.env",
  "auth": {
    "path": "./auth.py:auth_handler",
    "openapi": {
      "securitySchemes": {
        "BearerAuth": {
          "type": "http",
          "scheme": "bearer"
        }
      },
      "security": [{"BearerAuth": []}]
    }
  },
  "store": {
    "index": {
      "dims": 1536,
      "embed": "openai:text-embedding-3-small"
    }
  },
  "http": {
    "cors": {
      "allow_origins": ["https://app.example.com"],
      "allow_credentials": true
    }
  }
}
```

#### Multi-Agent JavaScript System

```json
{
  "node_version": "20",
  "dependencies": ["."],
  "graphs": {
    "coordinator": "./src/coordinator.ts:graph",
    "researcher": "./src/agents/researcher.ts:graph",
    "writer": "./src/agents/writer.ts:graph"
  },
  "env": {
    "OPENAI_API_KEY": "${OPENAI_API_KEY}",
    "LOG_LEVEL": "info"
  },
  "ui": {
    "workflow_view": "./ui/WorkflowView.tsx:default"
  }
}
```

#### Advanced Multi-Service Setup

```json
{
  "python_version": "3.12",
  "api_version": "0.2",
  "image_distro": "wolfi",
  "pip_installer": "uv",
  "dependencies": [
    ".",
    "langchain_openai>=0.1.0",
    "langchain_anthropic>=0.1.0",
    "chromadb>=0.4.0"
  ],
  "graphs": {
    "rag_agent": "./agents/rag.py:graph",
    "summary_agent": "./agents/summary.py:graph"
  },
  "env": "./.env",
  "dockerfile_lines": [
    "RUN apt-get update && apt-get install -y ffmpeg",
    "ENV TZ=America/New_York"
  ],
  "store": {
    "index": {
      "dims": 3072,
      "embed": "openai:text-embedding-3-large",
      "fields": ["title", "content", "metadata.description"]
    },
    "ttl": {
      "default_ttl": 10080,
      "sweep_interval_minutes": 60
    }
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "default_ttl": 10080
    },
    "serde": {
      "allowed_json_modules": [
        ["my_app", "models", "CustomState"]
      ]
    }
  },
  "auth": {
    "path": "./auth.py:custom_auth",
    "cache": {
      "cache_keys": ["user_id"],
      "ttl_seconds": 3600,
      "max_size": 1000
    }
  },
  "encryption": {
    "path": "./encryption.py:field_encryption"
  },
  "http": {
    "mount_prefix": "/api/v1",
    "cors": {
      "allow_origins": ["https://app.example.com"],
      "allow_credentials": true
    },
    "configurable_headers": {
      "includes": ["x-user-*", "x-tenant-id"],
      "excludes": ["*secret*", "*key*"]
    }
  },
  "webhooks": {
    "url": {
      "require_https": true,
      "allowed_domains": ["*.mycompany.com"]
    },
    "headers": {
      "X-Webhook-Secret": "${{ env.WEBHOOK_SECRET }}"
    }
  }
}
```

---

## Docker Integration

The LangGraph CLI deeply integrates with Docker to provide consistent development and deployment experiences.

### Docker Requirements

**Required Software:**
- Docker Engine 20.10+ (or Docker Desktop)
- Docker Compose (plugin or standalone)
  - Plugin: `docker compose` (recommended)
  - Standalone: `docker-compose` binary

**Checking installation:**
```bash
# Docker version
docker --version
# Docker version 24.0.0

# Docker Compose version
docker compose version
# Docker Compose version v2.20.0

# Or standalone
docker-compose --version
# docker-compose version 1.29.2
```

### Capability Detection

The CLI automatically detects Docker capabilities:

**Version Detection:**
```python
{
  "version_docker": (24, 0, 0),
  "version_compose": (2, 20, 0),
  "healthcheck_start_interval": True,  # Docker 25+
  "compose_type": "plugin"  # or "standalone"
}
```

**Feature Availability:**

| Feature | Requirement | Fallback |
|---------|-------------|----------|
| `start_interval` healthcheck | Docker 25+ | Use shorter `interval` |
| BuildKit | Docker 18.09+ | Classic builder |
| Multi-stage builds | Docker 17.05+ | Single-stage |
| Build contexts | BuildKit | Copy to temp directory |

### Image Management

#### Base Images

**Official LangGraph images:**

- `langchain/langgraph-api` - Python runtime
- `langchain/langgraphjs-api` - Node.js runtime
- `langchain/langgraph-server` - Legacy name (Python)
- `langchain/langgraph-debugger` - LangGraph Studio

**Image naming convention:**
```
langchain/langgraph-api:<api_version>-<language><version>-<distro>

Examples:
- langchain/langgraph-api:0.2.18-py3.11
- langchain/langgraph-api:0.2-py3.12-wolfi
- langchain/langgraphjs-api:node20
- langchain/langgraph-api:3.11  (latest API version)
```

**Image layers:**
```dockerfile
FROM langchain/langgraph-api:3.11

# Layer 1: Base OS and Python
# Layer 2: LangGraph API server
# Layer 3: System dependencies
# Layer 4: Python packages
# Layer 5: Application code
```

#### Build Context

**Default context:** Parent directory of `langgraph.json`

**For local dependencies:**
```
project/
├── langgraph.json
├── agent/
│   ├── __init__.py
│   └── graph.py
└── shared/
    └── utils.py
```

Build context: `project/` (includes both `agent/` and `shared/`)

**Additional contexts:**
For dependencies outside the config directory:

```
monorepo/
├── packages/
│   └── shared/  <-- Outside context
├── apps/
│   └── agent/
│       ├── langgraph.json
│       └── src/
```

The CLI automatically adds `--build-context` for parent dependencies.

#### Build Process

**Step-by-step:**

1. **Validation**
   ```bash
   Validating configuration...
   ✅ Configuration valid
   ```

2. **Base Image Pull**
   ```bash
   Pulling langchain/langgraph-api:3.11...
   3.11: Pulling from langchain/langgraph-api
   ✅ Image pulled
   ```

3. **Dockerfile Generation**
   ```bash
   Generating Dockerfile from config...
   - Processing dependencies
   - Configuring environment
   - Setting up graphs
   ```

4. **Docker Build**
   ```bash
   Building Docker image...
   [+] Building 45.2s (12/12) FINISHED
   => [1/7] FROM langchain/langgraph-api:3.11
   => [2/7] COPY requirements.txt /tmp/
   => [3/7] RUN pip install -r /tmp/requirements.txt
   => [4/7] COPY ./agent /deps/agent
   => [5/7] RUN pip install -e /deps/agent
   => [6/7] ENV LANGSERVE_GRAPHS=...
   => [7/7] RUN pip uninstall -y pip setuptools wheel
   => exporting to image
   ```

5. **Image Tagging**
   ```bash
   ✅ Successfully built and tagged: my-agent:latest
   ```

#### Multi-Platform Builds

Build for multiple architectures:

```bash
# AMD64 and ARM64
langgraph build -t my-agent:latest \
  --platform linux/amd64,linux/arm64

# With BuildKit
docker buildx build --platform linux/amd64,linux/arm64 \
  -t my-agent:latest .
```

**Platform-specific considerations:**
- ARM64: Required for Apple Silicon, AWS Graviton
- AMD64: Standard x86_64 servers
- Build time: ~2x longer for multi-platform

### Docker Compose Integration

#### Service Dependencies

**Dependency graph:**
```
langgraph-api
├── depends_on: langgraph-postgres (healthy)
├── depends_on: langgraph-redis (healthy)
└── optional: langgraph-debugger

langgraph-postgres
└── volume: langgraph-data

langgraph-redis
└── (ephemeral)

langgraph-debugger
└── depends_on: langgraph-postgres (healthy)
```

**Startup order:**
1. Create networks and volumes
2. Start Redis (wait for health)
3. Start Postgres (wait for health)
4. Start Debugger (if enabled)
5. Start API server

**Shutdown order:**
1. API server
2. Debugger
3. Redis
4. Postgres (volume persists)

#### Networking

**Default network:** `bridge` (auto-created)

**Service communication:**
```yaml
services:
  langgraph-api:
    environment:
      POSTGRES_URI: postgres://postgres:postgres@langgraph-postgres:5432/postgres
      REDIS_URI: redis://langgraph-redis:6379
```

**Hostname resolution:**
- `langgraph-postgres` → Postgres container IP
- `langgraph-redis` → Redis container IP
- `langgraph-api` → API container IP

**External access:**
```yaml
ports:
  - "8123:8000"  # Host:Container
```

Access from host: http://localhost:8123

#### Volume Management

**Persistent volume:**
```yaml
volumes:
  langgraph-data:
    driver: local
```

**Mount point:**
```yaml
services:
  langgraph-postgres:
    volumes:
      - langgraph-data:/var/lib/postgresql/data
```

**Volume operations:**

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect langgraph-data

# Backup volume
docker run --rm -v langgraph-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/data.tar.gz /data

# Restore volume
docker run --rm -v langgraph-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/data.tar.gz -C /

# Remove volume
docker volume rm langgraph-data
```

#### Compose File Generation

**Inline Dockerfile:**
```yaml
services:
  langgraph-api:
    build:
      context: .
      dockerfile_inline: |
        FROM langchain/langgraph-api:3.11
        RUN pip install langchain_openai
        ADD ./agent /deps/agent
        RUN pip install -e /deps/agent
        ENV LANGSERVE_GRAPHS='{"agent": "./agent/graph.py:graph"}'
```

**Benefits:**
- Single configuration source
- No separate Dockerfile to manage
- Automatic regeneration on config changes

**Watch mode:**
```yaml
services:
  langgraph-api:
    develop:
      watch:
        - path: langgraph.json
          action: rebuild
        - path: ./agent
          action: rebuild
```

**Behavior:**
- File changes trigger rebuild
- Container restarts automatically
- Fast iterative development

### Container Lifecycle

#### Startup Sequence

**Phase 1: Initialization**
```
Creating network "default"
Creating volume "langgraph-data"
```

**Phase 2: Service Startup**
```
Creating langgraph-redis    ... done
Creating langgraph-postgres ... done
Waiting for services to be healthy...
```

**Phase 3: Health Checks**
```
langgraph-redis      | PONG
langgraph-postgres   | ready to accept connections
```

**Phase 4: Application Start**
```
Creating langgraph-api ... done
langgraph-api        | INFO: Application startup complete
```

**Total startup time:** ~10-15 seconds (first run), ~2-3 seconds (subsequent)

#### Runtime Behavior

**Log streaming:**
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f langgraph-api

# With timestamps
docker compose logs -f -t
```

**Service status:**
```bash
docker compose ps

# Output:
NAME                 COMMAND              SERVICE             STATUS
langgraph-api        python /api/main.py  langgraph-api       Up (healthy)
langgraph-postgres   postgres             langgraph-postgres  Up (healthy)
langgraph-redis      redis-server         langgraph-redis     Up (healthy)
```

**Resource usage:**
```bash
docker stats

# Output:
CONTAINER          CPU %     MEM USAGE / LIMIT     NET I/O
langgraph-api      2.5%      512MiB / 8GiB        1.2MB / 800kB
langgraph-postgres 1.0%      128MiB / 8GiB        500kB / 300kB
langgraph-redis    0.5%      32MiB / 8GiB         100kB / 50kB
```

#### Shutdown and Cleanup

**Graceful shutdown:**
```bash
docker compose down

# Output:
Stopping langgraph-api      ... done
Stopping langgraph-debugger ... done
Stopping langgraph-redis    ... done
Stopping langgraph-postgres ... done
Removing langgraph-api      ... done
Removing langgraph-debugger ... done
Removing langgraph-redis    ... done
Removing langgraph-postgres ... done
Removing network default
```

**Full cleanup:**
```bash
docker compose down -v

# Also removes volumes (data loss!)
Removing volume langgraph-data
```

**Selective cleanup:**
```bash
# Remove only stopped containers
docker compose rm

# Remove images
docker compose down --rmi all

# Remove orphaned containers
docker compose down --remove-orphans
```

### Troubleshooting Docker Issues

#### Common Problems

**1. Docker not running**
```
Error: Docker not installed or not running
```

**Solution:**
```bash
# Start Docker Desktop (macOS/Windows)
# or
sudo systemctl start docker  # Linux
```

**2. Permission denied**
```
Error: Permission denied while trying to connect to Docker daemon
```

**Solution:**
```bash
# Add user to docker group (Linux)
sudo usermod -aG docker $USER
newgrp docker

# Or use sudo
sudo langgraph up
```

**3. Port already in use**
```
Error: Bind for 0.0.0.0:8123 failed: port is already allocated
```

**Solution:**
```bash
# Use different port
langgraph up --port 8124

# Or stop conflicting service
lsof -ti:8123 | xargs kill
```

**4. Build context too large**
```
Error: Build context is too large
```

**Solution:**
```bash
# Add .dockerignore
cat > .dockerignore << EOF
node_modules
__pycache__
.git
*.pyc
.env
EOF

# Or use explicit context
docker build -f Dockerfile -t my-agent .
```

**5. Out of disk space**
```
Error: no space left on device
```

**Solution:**
```bash
# Remove unused images
docker image prune -a

# Remove unused volumes
docker volume prune

# Remove all unused data
docker system prune -a --volumes
```

**6. Network issues**
```
Error: failed to pull image: connection timeout
```

**Solution:**
```bash
# Use proxy
docker build --build-arg HTTP_PROXY=http://proxy:8080

# Or configure Docker daemon
# Edit /etc/docker/daemon.json:
{
  "proxies": {
    "default": {
      "httpProxy": "http://proxy:8080",
      "httpsProxy": "http://proxy:8080"
    }
  }
}
```

#### Debugging Techniques

**Inspect containers:**
```bash
# Get container ID
docker ps -a

# Inspect configuration
docker inspect <container_id>

# View logs
docker logs <container_id>

# Execute commands
docker exec -it <container_id> bash
```

**Check connectivity:**
```bash
# From host to container
curl http://localhost:8123/ok

# From container to container
docker exec langgraph-api ping langgraph-postgres

# DNS resolution
docker exec langgraph-api nslookup langgraph-redis
```

**Analyze build:**
```bash
# Show build steps
docker build --progress plain -t my-agent .

# No cache (force rebuild)
docker build --no-cache -t my-agent .

# Stop at specific layer
docker build --target <stage_name> -t my-agent .
```

---

## Complete Examples

### Example 1: Quick Start Chatbot

**Goal:** Create and run a simple chatbot in under 5 minutes.

**Steps:**

1. **Create project:**
```bash
langgraph new ./my-chatbot --template new-langgraph-project-python
cd my-chatbot
```

2. **Set API key:**
```bash
echo "OPENAI_API_KEY=sk-..." > .env
```

3. **Start development server:**
```bash
pip install "langgraph-cli[inmem]"
langgraph dev
```

4. **Test in browser:**
   - Opens automatically at http://localhost:2024
   - Try LangGraph Studio at https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

**Project structure:**
```
my-chatbot/
├── langgraph.json
├── my_agent/
│   ├── __init__.py
│   └── agent.py
├── requirements.txt
├── .env
└── README.md
```

**langgraph.json:**
```json
{
  "dependencies": [
    ".",
    "langchain_openai"
  ],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

**Time:** ~5 minutes

---

### Example 2: Production Deployment Workflow

**Goal:** Build and deploy a production-ready agent with Docker.

**Steps:**

1. **Develop locally:**
```bash
# Start dev server
langgraph dev --port 8000

# Make changes, test with hot reload
# ...
```

2. **Test with Docker:**
```bash
# Build image
langgraph build -t my-agent:dev

# Test locally with full stack
langgraph up --image my-agent:dev --port 8080
```

3. **Build production image:**
```bash
# Pin versions in langgraph.json
{
  "python_version": "3.12",
  "api_version": "0.2.18",
  "dependencies": [
    ".",
    "langchain_openai==0.1.8",
    "langchain_anthropic==0.1.15"
  ]
}

# Build with multi-platform support
langgraph build -t my-agent:v1.0 \
  --platform linux/amd64,linux/arm64 \
  --api-version 0.2.18

# Tag for registry
docker tag my-agent:v1.0 myregistry.com/my-agent:v1.0
```

4. **Push to registry:**
```bash
docker push myregistry.com/my-agent:v1.0
```

5. **Deploy to production:**
```bash
# On production server
docker pull myregistry.com/my-agent:v1.0

# Generate deployment files
langgraph dockerfile ./Dockerfile --add-docker-compose

# Configure environment
cat > .env << EOF
LANGGRAPH_CLOUD_LICENSE_KEY=lgs-...
OPENAI_API_KEY=sk-...
POSTGRES_URI=postgresql://user:pass@prod-db:5432/langgraph
EOF

# Start services
docker compose up -d

# Monitor
docker compose logs -f langgraph-api
```

**Time:** ~30 minutes (including testing)

---

### Example 3: Multi-Agent System with Custom Services

**Goal:** Build a complex system with multiple agents and external services.

**Project structure:**
```
project/
├── langgraph.json
├── .env
├── docker-compose.extra.yml
├── agents/
│   ├── coordinator.py
│   ├── researcher.py
│   └── writer.py
├── shared/
│   ├── tools.py
│   └── prompts.py
└── requirements.txt
```

**langgraph.json:**
```json
{
  "python_version": "3.12",
  "api_version": "0.2.18",
  "dependencies": [
    ".",
    "langchain_openai>=0.1.0",
    "langchain_anthropic>=0.1.0",
    "chromadb>=0.4.0"
  ],
  "graphs": {
    "coordinator": "./agents/coordinator.py:graph",
    "researcher": "./agents/researcher.py:graph",
    "writer": "./agents/writer.py:graph"
  },
  "env": ".env",
  "store": {
    "index": {
      "dims": 1536,
      "embed": "openai:text-embedding-3-small"
    }
  },
  "dockerfile_lines": [
    "RUN apt-get update && apt-get install -y git"
  ]
}
```

**docker-compose.extra.yml:**
```yaml
services:
  chroma:
    image: chromadb/chroma:latest
    ports:
      - "8000:8000"
    volumes:
      - chroma-data:/chroma/chroma
    environment:
      - CHROMA_SERVER_AUTH_CREDENTIALS_PROVIDER=token
      - CHROMA_SERVER_AUTH_CREDENTIALS=test-token

  redis-commander:
    image: rediscommander/redis-commander:latest
    ports:
      - "8081:8081"
    environment:
      - REDIS_HOSTS=local:langgraph-redis:6379

volumes:
  chroma-data:
    driver: local
```

**.env:**
```bash
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
LANGSMITH_API_KEY=ls-...
CHROMA_URL=http://chroma:8000
CHROMA_TOKEN=test-token
```

**Run:**
```bash
# Development
langgraph dev --port 2024

# Production with extra services
langgraph up \
  --docker-compose ./docker-compose.extra.yml \
  --port 8123
```

**Access:**
- API: http://localhost:8123
- Chroma: http://localhost:8000
- Redis UI: http://localhost:8081
- Studio: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:8123

**Time:** ~1 hour (including service setup)

---

### Example 4: Monorepo with Shared Libraries

**Goal:** Manage multiple agents sharing common code in a monorepo.

**Structure:**
```
monorepo/
├── packages/
│   ├── shared/
│   │   ├── __init__.py
│   │   ├── tools.py
│   │   └── prompts.py
│   └── db/
│       ├── __init__.py
│       └── client.py
├── apps/
│   ├── agent-a/
│   │   ├── langgraph.json
│   │   ├── agent.py
│   │   └── .env
│   └── agent-b/
│       ├── langgraph.json
│       ├── agent.py
│       └── .env
└── pyproject.toml
```

**apps/agent-a/langgraph.json:**
```json
{
  "dependencies": [
    "../../packages/shared",
    "../../packages/db",
    ".",
    "langchain_openai"
  ],
  "graphs": {
    "agent": "./agent.py:graph"
  }
}
```

**Build process:**
```bash
cd apps/agent-a

# CLI detects parent dependencies automatically
langgraph build -t agent-a:latest

# Generates additional build contexts
# --build-context outer-shared=../../packages/shared
# --build-context outer-db=../../packages/db
```

**Development:**
```bash
# Each agent can be developed independently
cd apps/agent-a
langgraph dev --port 2024

cd apps/agent-b
langgraph dev --port 2025
```

**Shared CI/CD:**
```yaml
# .github/workflows/build.yml
name: Build Agents
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        agent: [agent-a, agent-b]
    steps:
      - uses: actions/checkout@v3
      - name: Build
        working-directory: apps/${{ matrix.agent }}
        run: |
          pip install langgraph-cli
          langgraph build -t ${{ matrix.agent }}:${{ github.sha }}
```

**Time:** ~2 hours (including monorepo setup)

---

### Example 5: Custom Authentication and Encryption

**Goal:** Implement custom security for enterprise deployment.

**Files:**

**auth.py:**
```python
from langgraph_api.auth import Auth
import jwt
from datetime import datetime

class JWTAuth(Auth):
    def __init__(self, secret_key: str):
        self.secret_key = secret_key

    async def authenticate(self, request):
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if not token:
            return None

        try:
            payload = jwt.decode(token, self.secret_key, algorithms=["HS256"])
            return {
                "user_id": payload["sub"],
                "permissions": payload.get("permissions", [])
            }
        except jwt.InvalidTokenError:
            return None

    async def authorize(self, request, user, resource):
        # Check permissions
        required_permission = f"{resource}:read"
        return required_permission in user.get("permissions", [])

auth = JWTAuth(secret_key="your-secret-key")
```

**encryption.py:**
```python
from langgraph_api.encryption import Encryption
from cryptography.fernet import Fernet
import os

class FieldEncryption(Encryption):
    def __init__(self):
        key = os.environ["ENCRYPTION_KEY"].encode()
        self.cipher = Fernet(key)

    async def encrypt(self, data: bytes) -> bytes:
        return self.cipher.encrypt(data)

    async def decrypt(self, data: bytes) -> bytes:
        return self.cipher.decrypt(data)

encryption = FieldEncryption()
```

**langgraph.json:**
```json
{
  "dependencies": [
    ".",
    "langchain_openai",
    "pyjwt",
    "cryptography"
  ],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "auth": {
    "path": "./auth.py:auth",
    "openapi": {
      "securitySchemes": {
        "BearerAuth": {
          "type": "http",
          "scheme": "bearer",
          "bearerFormat": "JWT"
        }
      },
      "security": [{"BearerAuth": []}]
    },
    "cache": {
      "cache_keys": ["user_id"],
      "ttl_seconds": 3600,
      "max_size": 1000
    }
  },
  "encryption": {
    "path": "./encryption.py:encryption"
  }
}
```

**.env:**
```bash
JWT_SECRET_KEY=your-secret-key-here
ENCRYPTION_KEY=your-fernet-key-here
```

**Test authentication:**
```python
# generate_token.py
import jwt
from datetime import datetime, timedelta

def generate_token(user_id: str, permissions: list[str]):
    payload = {
        "sub": user_id,
        "permissions": permissions,
        "exp": datetime.utcnow() + timedelta(hours=1)
    }
    return jwt.encode(payload, "your-secret-key", algorithm="HS256")

# Generate test token
token = generate_token("user123", ["agent:read", "agent:write"])
print(f"Bearer {token}")
```

**Usage:**
```bash
# Start server
langgraph up

# Test with authentication
curl -H "Authorization: Bearer <token>" \
  http://localhost:8123/assistants

# Without token (should fail)
curl http://localhost:8123/assistants
```

**Time:** ~3 hours (including security implementation)

---

### Example 6: CI/CD Pipeline with GitHub Actions

**Goal:** Automated testing, building, and deployment.

**.github/workflows/deploy.yml:**
```yaml
name: Deploy LangGraph Agent

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          pip install "langgraph-cli[inmem]"
          pip install pytest

      - name: Validate config
        run: |
          langgraph dockerfile ./Dockerfile

      - name: Run tests
        run: pytest tests/

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v3

      - name: Log in to registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Install LangGraph CLI
        run: pip install langgraph-cli

      - name: Build and push
        run: |
          # Build for multiple platforms
          langgraph build \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --platform linux/amd64,linux/arm64 \
            --push

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to production
        run: |
          # SSH to production server
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} << 'EOF'
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} my-agent:latest
            cd /opt/langgraph
            docker compose up -d
          EOF
```

**Time:** ~4 hours (including CI/CD setup)

---

## Best Practices

### Development

1. **Use dev mode for iteration:**
   ```bash
   langgraph dev --server-log-level DEBUG
   ```

2. **Pin dependencies in production:**
   ```json
   {
     "dependencies": [
       "langchain_openai==0.1.8",
       "langchain==0.2.3"
     ]
   }
   ```

3. **Keep .env out of version control:**
   ```bash
   echo ".env" >> .gitignore
   ```

4. **Use .env.example for documentation:**
   ```bash
   cp .env .env.example
   # Remove sensitive values from .env.example
   git add .env.example
   ```

### Docker

1. **Use .dockerignore:**
   ```
   .git
   .env
   __pycache__
   *.pyc
   node_modules
   ```

2. **Multi-stage builds for smaller images:**
   Automatically handled by CLI

3. **Pin base image versions:**
   ```json
   {
     "api_version": "0.2.18",
     "image_distro": "wolfi"
   }
   ```

4. **Clean up regularly:**
   ```bash
   docker system prune -a --volumes
   ```

### Configuration

1. **Validate before deploying:**
   ```bash
   langgraph dockerfile ./Dockerfile
   ```

2. **Use separate configs for environments:**
   ```
   langgraph.dev.json
   langgraph.prod.json
   ```

3. **Document custom configurations:**
   Add comments in README.md

4. **Version control langgraph.json:**
   Always commit configuration changes

### Security

1. **Use secrets management:**
   - Development: `.env` files
   - Production: Kubernetes secrets, AWS Secrets Manager, etc.

2. **Enable authentication:**
   ```json
   {
     "auth": {
       "path": "./auth.py:auth"
     }
   }
   ```

3. **Configure CORS properly:**
   ```json
   {
     "http": {
       "cors": {
         "allow_origins": ["https://yourdomain.com"],
         "allow_credentials": true
       }
     }
   }
   ```

4. **Use encryption for sensitive data:**
   ```json
   {
     "encryption": {
       "path": "./encryption.py:encryption"
     }
   }
   ```

---

## Troubleshooting

### Common Issues

**Issue: `langgraph dev` fails with "module not found"**

Solution:
```bash
# Install inmem extra
pip install "langgraph-cli[inmem]"

# Verify Python version
python --version  # Must be 3.11+
```

**Issue: Docker build fails with "COPY failed"**

Solution:
```bash
# Check file paths in langgraph.json
# Ensure paths are relative to langgraph.json

# Check .dockerignore
# Make sure required files aren't excluded
```

**Issue: Port already in use**

Solution:
```bash
# Use different port
langgraph up --port 8124

# Or kill existing process
lsof -ti:8123 | xargs kill
```

**Issue: Watch mode not triggering**

Solution:
```bash
# Ensure dependencies start with "."
{
  "dependencies": [".", "./shared"]
}

# Try recreating
langgraph up --watch --recreate
```

**Issue: Authentication not working**

Solution:
```bash
# Check auth path
{
  "auth": {
    "path": "./auth.py:auth"  # Must be correct
  }
}

# Verify auth implementation
# Must inherit from langgraph_api.auth.Auth
```

### Getting Help

- **Documentation:** https://langchain-ai.github.io/langgraph/
- **GitHub Issues:** https://github.com/langchain-ai/langgraph/issues
- **Discord:** https://discord.gg/langchain
- **CLI Help:** `langgraph --help`
