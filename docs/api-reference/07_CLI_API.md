# LangGraph CLI API Reference

Complete API reference for the LangGraph Command Line Interface (CLI).

## Table of Contents

- [CLI Commands](#cli-commands)
- [Configuration API](#configuration-api)
- [Schema Classes](#schema-classes)
- [Docker API](#docker-api)
- [Template API](#template-api)
- [Execution API](#execution-api)
- [Progress API](#progress-api)
- [Utility Functions](#utility-functions)

---

## CLI Commands

The LangGraph CLI provides several commands for managing LangGraph applications.

### langgraph up

Launch the LangGraph API server with all required services (Redis, Postgres, etc.).

**Usage:**
```bash
langgraph up [OPTIONS]
```

**Options:**
| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--config`, `-c` | Path | `langgraph.json` | Path to configuration file |
| `--port`, `-p` | int | `8123` | Port to expose |
| `--docker-compose`, `-d` | Path | None | Path to additional docker-compose.yml file |
| `--recreate` / `--no-recreate` | bool | `False` | Recreate containers even if unchanged |
| `--pull` / `--no-pull` | bool | `True` | Pull latest images |
| `--watch` | bool | `False` | Restart on file changes |
| `--wait` | bool | `False` | Wait for services to start before returning |
| `--verbose` | bool | `False` | Show more output from server logs |
| `--debugger-port` | int | None | Serve debugger UI on specified port |
| `--debugger-base-url` | str | None | URL for debugger to access LangGraph API |
| `--postgres-uri` | str | None | Postgres URI (defaults to launching local DB) |
| `--api-version` | str | None | API server version for base image |
| `--image` | str | None | Docker image to use (skips building) |
| `--base-image` | str | None | Base image for LangGraph API server |

**Example:**
```bash
# Basic usage
langgraph up

# With custom port and watch mode
langgraph up --port 8000 --watch

# With debugger
langgraph up --debugger-port 8080

# Skip pulling images (use local builds)
langgraph up --no-pull

# Use specific API version
langgraph up --api-version 0.2.18
```

**Notes:**
- Requires Docker to be installed and running
- Sets up Redis, Postgres (optional), and LangGraph API containers
- For local dev, requires `LANGSMITH_API_KEY` environment variable
- For production, requires `LANGGRAPH_CLOUD_LICENSE_KEY` environment variable

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/cli.py:200`

---

### langgraph build

Build a Docker image for the LangGraph API server.

**Usage:**
```bash
langgraph build [OPTIONS] [DOCKER_BUILD_ARGS]...
```

**Options:**
| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--config`, `-c` | Path | `langgraph.json` | Path to configuration file |
| `--tag`, `-t` | str | **Required** | Tag for the Docker image |
| `--pull` / `--no-pull` | bool | `True` | Pull latest base images |
| `--base-image` | str | None | Base image for LangGraph API server |
| `--api-version` | str | None | API server version to use |
| `--install-command` | str | None | Custom install command (JS projects) |
| `--build-command` | str | None | Custom build command (JS projects) |

**Arguments:**
| Name | Description |
|------|-------------|
| `DOCKER_BUILD_ARGS` | Additional arguments to pass to `docker build` |

**Example:**
```bash
# Basic build
langgraph build -t my-app:latest

# With custom base image
langgraph build -t my-app:v1.0 --base-image langchain/langgraph-api:0.2.18

# With additional Docker build args
langgraph build -t my-app:latest --build-arg MY_VAR=value

# Skip pulling base image
langgraph build -t my-app:latest --no-pull

# JS project with custom commands
langgraph build -t my-app:latest --install-command "pnpm install" --build-command "pnpm build"
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/cli.py:402`

---

### langgraph dockerfile

Generate a Dockerfile for the LangGraph API server.

**Usage:**
```bash
langgraph dockerfile [OPTIONS] SAVE_PATH
```

**Arguments:**
| Name | Required | Description |
|------|----------|-------------|
| `SAVE_PATH` | Yes | Path where Dockerfile will be saved |

**Options:**
| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--config`, `-c` | Path | `langgraph.json` | Path to configuration file |
| `--add-docker-compose` | bool | `False` | Generate docker-compose.yml, .env, and .dockerignore |
| `--base-image` | str | None | Base image for LangGraph API server |
| `--api-version` | str | None | API server version to use |

**Example:**
```bash
# Generate Dockerfile only
langgraph dockerfile ./Dockerfile

# Generate Dockerfile with docker-compose files
langgraph dockerfile ./Dockerfile --add-docker-compose

# With custom base image
langgraph dockerfile ./Dockerfile --base-image langchain/langgraph-api:0.2.18
```

**Output Files:**
When `--add-docker-compose` is used:
- `Dockerfile` - The generated Dockerfile
- `docker-compose.yml` - Docker Compose configuration
- `.dockerignore` - Docker ignore file
- `.env` - Environment variables template (if not exists)

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/cli.py:510`

---

### langgraph dev

Run the LangGraph API server in development mode with hot reloading and debugging support.

**Usage:**
```bash
langgraph dev [OPTIONS]
```

**Options:**
| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--host` | str | `127.0.0.1` | Network interface to bind to |
| `--port` | int | `2024` | Port number to bind to |
| `--no-reload` | bool | `False` | Disable automatic reloading on code changes |
| `--config` | Path | `langgraph.json` | Path to configuration file |
| `--n-jobs-per-worker` | int | `10` | Max concurrent jobs per worker process |
| `--no-browser` | bool | `False` | Skip opening browser on server start |
| `--debug-port` | int | None | Enable remote debugging on specified port |
| `--wait-for-client` | bool | `False` | Wait for debugger client before starting |
| `--studio-url` | str | None | URL of LangGraph Studio instance |
| `--allow-blocking` | bool | `False` | Don't raise errors for blocking I/O |
| `--tunnel` | bool | `False` | Expose via public tunnel (Cloudflare) |
| `--server-log-level` | str | `WARNING` | Log level for API server |

**Example:**
```bash
# Basic development server
langgraph dev

# Custom port and host
langgraph dev --host 0.0.0.0 --port 8000

# With remote debugging
langgraph dev --debug-port 5678

# With public tunnel
langgraph dev --tunnel

# Disable auto-reload
langgraph dev --no-reload
```

**Requirements:**
- Python 3.11 or higher
- Requires `langgraph-cli[inmem]` to be installed:
  ```bash
  pip install -U "langgraph-cli[inmem]"
  ```

**Notes:**
- In-memory server for JS graphs is not supported; use `npx @langchain/langgraph-cli` instead
- Automatically opens LangGraph Studio in browser unless `--no-browser` is specified
- Hot reloading watches for file changes in dependencies listed in config

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/cli.py:685`

---

### langgraph new

Create a new LangGraph project from a template.

**Usage:**
```bash
langgraph new [OPTIONS] [PATH]
```

**Arguments:**
| Name | Required | Description |
|------|----------|-------------|
| `PATH` | No | Path where project will be created (prompts if not provided) |

**Options:**
| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--template` | str | None | Template ID to use (prompts if not provided) |

**Available Templates:**

| Template ID | Description |
|-------------|-------------|
| `new-langgraph-project-python` | A simple, minimal chatbot with memory (Python) |
| `new-langgraph-project-js` | A simple, minimal chatbot with memory (JS/TS) |
| `react-agent-python` | A simple agent that can be flexibly extended to many tools (Python) |
| `react-agent-js` | A simple agent that can be flexibly extended to many tools (JS/TS) |
| `memory-agent-python` | ReAct-style agent with memory storage across threads (Python) |
| `memory-agent-js` | ReAct-style agent with memory storage across threads (JS/TS) |
| `retrieval-agent-python` | Agent with retrieval-based QA system (Python) |
| `retrieval-agent-js` | Agent with retrieval-based QA system (JS/TS) |
| `data-enrichment-agent-python` | Agent that performs web searches and structures findings (Python) |
| `data-enrichment-agent-js` | Agent that performs web searches and structures findings (JS/TS) |

**Example:**
```bash
# Interactive mode (prompts for template and path)
langgraph new

# Specify path
langgraph new ./my-project

# Specify template
langgraph new --template react-agent-python

# Specify both
langgraph new ./my-agent --template memory-agent-python
```

**Notes:**
- Directory must be empty or non-existent
- Template is downloaded from GitHub and extracted to specified path
- After creation, follow the README in the new project for setup instructions

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/cli.py:779`

---

## Configuration API

Functions for validating and processing LangGraph configuration files.

### validate_config

Validate a configuration dictionary.

**Signature:**
```python
def validate_config(config: Config) -> Config:
```

**Parameters:**
- `config` (Config): Configuration dictionary to validate

**Returns:**
- `Config`: Validated and normalized configuration dictionary

**Raises:**
- `click.UsageError`: If configuration is invalid

**Description:**
Validates a configuration dictionary, ensuring all fields are correctly specified:
- Checks Python/Node.js version formats and minimum requirements
- Validates that at least one dependency exists (for Python)
- Validates that at least one graph is defined
- Validates `image_distro` is one of: `debian`, `bullseye`, `bookworm`, `wolfi`
- Validates `pip_installer` is one of: `auto`, `pip`, `uv`
- Validates auth, encryption, and HTTP configuration paths
- Validates `keep_pkg_tools` settings

**Example:**
```python
from langgraph_cli.config import validate_config

config = {
    "python_version": "3.11",
    "dependencies": ["langchain_openai"],
    "graphs": {
        "my_graph": "./graph.py:graph"
    }
}

validated = validate_config(config)
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:111`

---

### validate_config_file

Load and validate a configuration file.

**Signature:**
```python
def validate_config_file(config_path: pathlib.Path) -> Config:
```

**Parameters:**
- `config_path` (pathlib.Path): Path to the configuration JSON file

**Returns:**
- `Config`: Validated configuration dictionary

**Raises:**
- `click.UsageError`: If configuration file is invalid or malformed
- `FileNotFoundError`: If configuration file doesn't exist

**Description:**
Loads a JSON configuration file from disk and validates it. Also checks for Node.js version compatibility if `package.json` exists.

**Example:**
```python
from pathlib import Path
from langgraph_cli.config import validate_config_file

config = validate_config_file(Path("langgraph.json"))
print(config["graphs"])
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:268`

---

### config_to_docker

Generate a Dockerfile from a configuration.

**Signature:**
```python
def config_to_docker(
    config_path: pathlib.Path,
    config: Config,
    *,
    base_image: str | None = None,
    api_version: str | None = None,
    install_command: str | None = None,
    build_command: str | None = None,
    build_context: str | None = None,
    escape_variables: bool = False,
) -> tuple[str, dict[str, str]]:
```

**Parameters:**
- `config_path` (pathlib.Path): Path to configuration file
- `config` (Config): Validated configuration dictionary
- `base_image` (str | None): Base Docker image to use
- `api_version` (str | None): API version for the base image
- `install_command` (str | None): Custom install command (JS projects)
- `build_command` (str | None): Custom build command (JS projects)
- `build_context` (str | None): Build context directory path
- `escape_variables` (bool): Whether to escape shell variables in Dockerfile

**Returns:**
- `tuple[str, dict[str, str]]`:
  - Dockerfile content as string
  - Dictionary of additional build contexts (for BuildKit)

**Description:**
Generates a complete Dockerfile for the LangGraph application based on configuration. Handles:
- Python and Node.js projects
- Local and PyPI dependencies
- Environment variables
- Custom Dockerfile lines
- Build context management for monorepos

**Example:**
```python
from pathlib import Path
from langgraph_cli.config import validate_config_file, config_to_docker

config_path = Path("langgraph.json")
config = validate_config_file(config_path)
dockerfile, contexts = config_to_docker(config_path, config)

# Write to file
with open("Dockerfile", "w") as f:
    f.write(dockerfile)
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:1205`

---

### config_to_compose

Generate docker-compose YAML fragment from configuration.

**Signature:**
```python
def config_to_compose(
    config_path: pathlib.Path,
    config: Config,
    base_image: str | None = None,
    api_version: str | None = None,
    image: str | None = None,
    watch: bool = False,
) -> str:
```

**Parameters:**
- `config_path` (pathlib.Path): Path to configuration file
- `config` (Config): Validated configuration dictionary
- `base_image` (str | None): Base Docker image
- `api_version` (str | None): API version
- `image` (str | None): Pre-built image to use
- `watch` (bool): Enable watch mode for auto-rebuild

**Returns:**
- `str`: Docker Compose YAML fragment for the langgraph-api service

**Description:**
Generates a docker-compose.yml fragment that configures the `langgraph-api` service. Includes:
- Environment variables from config
- Environment file references
- Build configuration
- Watch mode configuration

**Example:**
```python
from pathlib import Path
from langgraph_cli.config import validate_config_file, config_to_compose

config_path = Path("langgraph.json")
config = validate_config_file(config_path)
compose_fragment = config_to_compose(config_path, config, watch=True)
print(compose_fragment)
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:1238`

---

### docker_tag

Generate Docker image tag from configuration.

**Signature:**
```python
def docker_tag(
    config: Config,
    base_image: str | None = None,
    api_version: str | None = None,
) -> str:
```

**Parameters:**
- `config` (Config): Configuration dictionary
- `base_image` (str | None): Base image name
- `api_version` (str | None): API version

**Returns:**
- `str`: Complete Docker image tag (e.g., `langchain/langgraph-api:3.11`)

**Description:**
Generates the appropriate Docker image tag based on configuration, including:
- Language version (Python or Node.js)
- Image distribution (debian, wolfi, etc.)
- API version prefix if specified

**Example:**
```python
from langgraph_cli.config import docker_tag

config = {
    "python_version": "3.11",
    "image_distro": "debian"
}

tag = docker_tag(config)
# Returns: "langchain/langgraph-api:3.11"

tag_with_version = docker_tag(config, api_version="0.2.18")
# Returns: "langchain/langgraph-api:0.2.18-py3.11"
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:1156`

---

### default_base_image

Get the default base image for a configuration.

**Signature:**
```python
def default_base_image(config: Config) -> str:
```

**Parameters:**
- `config` (Config): Configuration dictionary

**Returns:**
- `str`: Default base image name

**Description:**
Determines the appropriate default base image:
- Returns `langchain/langgraphjs-api` for Node.js-only projects
- Returns `langchain/langgraph-api` for Python projects
- Returns custom `base_image` if specified in config

**Example:**
```python
from langgraph_cli.config import default_base_image

config_python = {"python_version": "3.11"}
print(default_base_image(config_python))
# Output: "langchain/langgraph-api"

config_js = {"node_version": "20"}
print(default_base_image(config_js))
# Output: "langchain/langgraphjs-api"
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:1148`

---

### python_config_to_docker

Generate Dockerfile for Python-based LangGraph projects.

**Signature:**
```python
def python_config_to_docker(
    config_path: pathlib.Path,
    config: Config,
    base_image: str,
    api_version: str | None = None,
    *,
    escape_variables: bool = False,
) -> tuple[str, dict[str, str]]:
```

**Parameters:**
- `config_path` (pathlib.Path): Path to configuration file
- `config` (Config): Validated configuration
- `base_image` (str): Base Docker image
- `api_version` (str | None): API version
- `escape_variables` (bool): Escape shell variables in Dockerfile

**Returns:**
- `tuple[str, dict[str, str]]`: Dockerfile content and additional contexts

**Description:**
Generates a Dockerfile specifically for Python projects, handling:
- PyPI and local dependencies
- pip configuration files
- Package installation (pip or uv)
- Virtual environment setup
- Graph path resolution
- Auth/encryption/HTTP path resolution
- Build tool cleanup

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:822`

---

### node_config_to_docker

Generate Dockerfile for Node.js-based LangGraph projects.

**Signature:**
```python
def node_config_to_docker(
    config_path: pathlib.Path,
    config: Config,
    base_image: str,
    api_version: str | None = None,
    install_command: str | None = None,
    build_command: str | None = None,
    build_context: str | None = None,
) -> tuple[str, dict[str, str]]:
```

**Parameters:**
- `config_path` (pathlib.Path): Path to configuration file
- `config` (Config): Validated configuration
- `base_image` (str): Base Docker image
- `api_version` (str | None): API version
- `install_command` (str | None): Custom install command
- `build_command` (str | None): Custom build command
- `build_context` (str | None): Build context path

**Returns:**
- `tuple[str, dict[str, str]]`: Dockerfile content and additional contexts

**Description:**
Generates a Dockerfile for JavaScript/TypeScript projects, with support for:
- npm, yarn, pnpm, and bun package managers
- Custom install and build commands
- Monorepo support via build context
- Graph configuration

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:1054`

---

### get_build_tools_to_uninstall

Determine which build tools to remove from final image.

**Signature:**
```python
def get_build_tools_to_uninstall(config: Config) -> tuple[str]:
```

**Parameters:**
- `config` (Config): Configuration dictionary

**Returns:**
- `tuple[str]`: Tuple of build tools to uninstall (e.g., `("pip", "setuptools", "wheel")`)

**Description:**
Determines which Python build tools should be removed from the final Docker image based on the `keep_pkg_tools` configuration option to reduce image size.

**Example:**
```python
from langgraph_cli.config import get_build_tools_to_uninstall

config = {"keep_pkg_tools": ["pip"]}
tools = get_build_tools_to_uninstall(config)
# Returns: ("setuptools", "wheel")

config = {"keep_pkg_tools": True}
tools = get_build_tools_to_uninstall(config)
# Returns: ()

config = {}
tools = get_build_tools_to_uninstall(config)
# Returns: ("pip", "setuptools", "wheel")
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/config.py:801`

---

## Schema Classes

TypedDict classes that define the structure of LangGraph configuration files.

### Config

Top-level configuration for LangGraph CLI deployments.

**Type:**
```python
class Config(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `python_version` | str | No | Python version in 'major.minor' format (e.g., '3.11'). Must be >= 3.11 |
| `node_version` | str \| None | No | Node.js major version (e.g., '20'). Must be >= 20 |
| `api_version` | str \| None | No | Semantic version of LangGraph API server to use |
| `base_image` | str \| None | No | Base Docker image (defaults to langchain/langgraph-api or langchain/langgraphjs-api) |
| `image_distro` | Distros \| None | No | Linux distribution: 'wolfi', 'debian', 'bullseye', or 'bookworm' |
| `pip_config_file` | str \| None | No | Path to pip config file for custom indices/credentials |
| `pip_installer` | str \| None | No | Package installer: 'auto', 'pip', or 'uv' |
| `dockerfile_lines` | list[str] | No | Additional Dockerfile instructions to append |
| `dependencies` | list[str] | No | List of dependencies (PyPI packages or local paths) |
| `graphs` | dict[str, str] | No | Named graph definitions mapping to Python/JS objects |
| `env` | dict[str, str] \| str | No | Environment variables (dict or path to .env file) |
| `store` | StoreConfig \| None | No | Configuration for long-term memory store |
| `checkpointer` | CheckpointerConfig \| None | No | Configuration for state checkpointing |
| `auth` | AuthConfig \| None | No | Custom authentication configuration |
| `encryption` | EncryptionConfig \| None | No | Custom at-rest encryption configuration |
| `http` | HttpConfig \| None | No | HTTP server configuration |
| `webhooks` | WebhooksConfig \| None | No | Webhooks configuration |
| `ui` | dict[str, str] \| None | No | UI component definitions |
| `ui_config` | dict \| None | No | UI configuration |
| `keep_pkg_tools` | bool \| list[str] \| None | No | Whether to keep packaging tools (pip, setuptools, wheel) |

**Example:**
```json
{
  "python_version": "3.11",
  "dependencies": [
    "langchain_openai",
    "./my_package"
  ],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "env": ".env",
  "store": {
    "index": {
      "dims": 1536,
      "embed": "openai:text-embedding-3-small"
    }
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:528`

---

### StoreConfig

Configuration for the built-in long-term memory store with optional semantic search.

**Type:**
```python
class StoreConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `index` | IndexConfig \| None | No | Vector-based semantic search configuration |
| `ttl` | TTLConfig \| None | No | Time-to-live behavior configuration |

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
      "default_ttl": 60,
      "refresh_on_read": true,
      "sweep_interval_minutes": 5
    }
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:81`

---

### IndexConfig

Configuration for semantic search indexing in the store.

**Type:**
```python
class IndexConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `dims` | int | Yes | Embedding vector dimensionality (must match model output) |
| `embed` | str | Yes | Embedding model identifier (e.g., "openai:text-embedding-3-large") |
| `fields` | list[str] \| None | No | JSON fields to extract for embedding (defaults to ["$"]) |

**Common Embedding Dimensions:**
- `openai:text-embedding-3-large`: 3072
- `openai:text-embedding-3-small`: 1536
- `openai:text-embedding-ada-002`: 1536
- `cohere:embed-english-v3.0`: 1024
- `cohere:embed-multilingual-v3.0`: 1024

**Example:**
```json
{
  "index": {
    "dims": 1536,
    "embed": "openai:text-embedding-3-small",
    "fields": ["title", "abstract"]
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:31`

---

### TTLConfig

Configuration for time-to-live behavior in the store.

**Type:**
```python
class TTLConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `refresh_on_read` | bool | No | Refresh TTL on read operations (defaults to True) |
| `default_ttl` | float \| None | No | Default TTL in minutes for new items |
| `sweep_interval_minutes` | int \| None | No | Interval between automatic cleanup sweeps |

**Example:**
```json
{
  "ttl": {
    "refresh_on_read": true,
    "default_ttl": 1440,
    "sweep_interval_minutes": 60
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:7`

---

### CheckpointerConfig

Configuration for state checkpointing.

**Type:**
```python
class CheckpointerConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ttl` | ThreadTTLConfig \| None | No | TTL configuration for checkpointed data |
| `serde` | SerdeConfig \| None | No | Serialization/deserialization configuration |

**Example:**
```json
{
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "default_ttl": 60,
      "sweep_interval_minutes": 5
    },
    "serde": {
      "allowed_json_modules": [["my_app", "models", "CustomType"]],
      "pickle_fallback": false
    }
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:162`

---

### ThreadTTLConfig

Configure default TTL for checkpointed data within threads.

**Type:**
```python
class ThreadTTLConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `strategy` | Literal["delete"] | No | Deletion strategy for expired data |
| `default_ttl` | float \| None | No | Default TTL in minutes for checkpointed data |
| `sweep_interval_minutes` | int \| None | No | Interval between sweep iterations |

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:107`

---

### SerdeConfig

Configuration for serialization/deserialization of checkpointed state.

**Type:**
```python
class SerdeConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `allowed_json_modules` | list[list[str]] \| bool \| None | No | Allowed Python modules for deserialization |
| `pickle_fallback` | bool | No | Allow pickling as fallback (defaults to True) |

**Example:**
```json
{
  "serde": {
    "allowed_json_modules": [
      ["my_agent", "my_file", "SomeType"]
    ],
    "pickle_fallback": false
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:123`

---

### AuthConfig

Configuration for custom authentication.

**Type:**
```python
class AuthConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | str | Yes | Path to Auth() instance (format: "path/to/file.py:my_auth") |
| `disable_studio_auth` | bool | No | Disable LangSmith API-key auth for Studio requests |
| `openapi` | SecurityConfig | No | Security configuration for OpenAPI spec |
| `cache` | CacheConfig | No | Cache configuration for auth results |

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
              "scopes": {"read": "Read access"}
            }
          }
        }
      },
      "security": [{"OAuth2": ["read"]}]
    }
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:257`

---

### SecurityConfig

OpenAPI security definitions and requirements.

**Type:**
```python
class SecurityConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `securitySchemes` | dict[str, dict[str, Any]] | No | Security scheme definitions |
| `security` | list[dict[str, list[str]]] | No | Global security requirements |
| `paths` | dict[str, dict[str, list[dict[str, list[str]]]]] | No | Path-specific security overrides |

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:184`

---

### CacheConfig

Configuration for authentication result caching.

**Type:**
```python
class CacheConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cache_keys` | list[str] | No | Header keys to use for caching |
| `ttl_seconds` | int | No | Cache TTL in seconds |
| `max_size` | int | No | Maximum cache size |

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:236`

---

### EncryptionConfig

Configuration for custom at-rest encryption.

**Type:**
```python
class EncryptionConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | str | Yes | Path to Encryption() instance (format: "path/to/file.py:my_encryption") |

**Example:**
```json
{
  "encryption": {
    "path": "./encryption.py:my_encryption"
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:305`

---

### HttpConfig

Configuration for the HTTP server.

**Type:**
```python
class HttpConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `app` | str | No | Path to custom Starlette/FastAPI app |
| `disable_assistants` | bool | No | Disable /assistants routes |
| `disable_threads` | bool | No | Disable /threads routes |
| `disable_runs` | bool | No | Disable /runs routes |
| `disable_store` | bool | No | Disable /store routes |
| `disable_mcp` | bool | No | Disable /mcp routes |
| `disable_a2a` | bool | No | Disable /a2a routes |
| `disable_meta` | bool | No | Disable meta endpoints (/openapi.json, /info, /metrics, /docs) |
| `disable_ui` | bool | No | Disable /ui routes |
| `disable_webhooks` | bool | No | Disable webhook delivery |
| `cors` | CorsConfig \| None | No | CORS configuration |
| `configurable_headers` | ConfigurableHeaderConfig \| None | No | Header configuration |
| `logging_headers` | ConfigurableHeaderConfig \| None | No | Logging header configuration |
| `middleware_order` | MiddlewareOrders \| None | No | Middleware execution order |
| `enable_custom_route_auth` | bool | No | Enable auth for custom routes |
| `mount_prefix` | str | No | URL prefix for all routes |

**Example:**
```json
{
  "http": {
    "disable_assistants": false,
    "cors": {
      "allow_origins": ["https://example.com"],
      "allow_methods": ["GET", "POST"],
      "allow_credentials": true
    },
    "mount_prefix": "/api"
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:392`

---

### CorsConfig

Cross-Origin Resource Sharing configuration.

**Type:**
```python
class CorsConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `allow_origins` | list[str] | No | Allowed origins (e.g., ["https://example.com"]) |
| `allow_methods` | list[str] | No | Allowed HTTP methods |
| `allow_headers` | list[str] | No | Allowed headers |
| `allow_credentials` | bool | No | Allow credentials in requests |
| `allow_origin_regex` | str | No | Regex pattern for dynamic origin matching |
| `expose_headers` | list[str] | No | Headers exposed to browser |
| `max_age` | int | No | Preflight cache duration in seconds |

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:326`

---

### ConfigurableHeaderConfig

Configure which headers to include as configurable values.

**Type:**
```python
class ConfigurableHeaderConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `includes` | list[str] \| None | No | Headers to include (supports wildcards) |
| `excludes` | list[str] \| None | No | Headers to exclude (takes precedence) |

**Example:**
```json
{
  "configurable_headers": {
    "includes": ["x-custom-*"],
    "excludes": ["*key*", "*token*"]
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:365`

---

### WebhooksConfig

Configuration for outbound webhook delivery.

**Type:**
```python
class WebhooksConfig(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `env_prefix` | str | No | Required prefix for environment variable references (default: "LG_WEBHOOK_") |
| `url` | WebhookUrlPolicy | No | URL validation policy |
| `headers` | dict[str, str] | No | Static headers (supports `${{ env.VAR }}` templates) |

**Example:**
```json
{
  "webhooks": {
    "env_prefix": "WEBHOOK_",
    "url": {
      "require_https": true,
      "allowed_domains": ["*.example.com"]
    },
    "headers": {
      "Authorization": "Bearer ${{ env.WEBHOOK_TOKEN }}"
    }
  }
}
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:510`

---

### WebhookUrlPolicy

URL validation policy for webhook endpoints.

**Type:**
```python
class WebhookUrlPolicy(TypedDict, total=False):
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `require_https` | bool | No | Enforce HTTPS for absolute URLs |
| `allowed_domains` | list[str] | No | Hostname allowlist (supports wildcards like "*.example.com") |
| `allowed_ports` | list[int] | No | Port allowlist |
| `max_url_length` | int | No | Maximum URL length |
| `disable_loopback` | bool | No | Disallow relative URLs |

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/schemas.py:488`

---

## Docker API

Functions for managing Docker operations and generating Docker Compose configurations.

### check_capabilities

Check Docker environment capabilities.

**Signature:**
```python
def check_capabilities(runner) -> DockerCapabilities:
```

**Parameters:**
- `runner`: Runner instance for executing commands

**Returns:**
- `DockerCapabilities`: Named tuple containing:
  - `version_docker` (Version): Docker version
  - `version_compose` (Version): Docker Compose version
  - `healthcheck_start_interval` (bool): Whether start_interval is supported
  - `compose_type` (DockerComposeType): "plugin" or "standalone"

**Raises:**
- `click.UsageError`: If Docker is not installed or not running

**Description:**
Checks the Docker environment and returns information about available features and versions. Used to determine which Docker Compose syntax to use.

**Example:**
```python
from langgraph_cli.exec import Runner
from langgraph_cli.docker import check_capabilities

with Runner() as runner:
    caps = check_capabilities(runner)
    print(f"Docker version: {caps.version_docker}")
    print(f"Compose version: {caps.version_compose}")
    print(f"Compose type: {caps.compose_type}")
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/docker.py:48`

---

### compose

Generate docker-compose YAML as a string.

**Signature:**
```python
def compose(
    capabilities: DockerCapabilities,
    *,
    port: int,
    debugger_port: int | None = None,
    debugger_base_url: str | None = None,
    postgres_uri: str | None = None,
    image: str | None = None,
    base_image: str | None = None,
    api_version: str | None = None,
) -> str:
```

**Parameters:**
- `capabilities` (DockerCapabilities): Docker environment capabilities
- `port` (int): Port to expose LangGraph API on
- `debugger_port` (int | None): Port for LangGraph Studio debugger
- `debugger_base_url` (str | None): Base URL for debugger to access API
- `postgres_uri` (str | None): External Postgres URI (if not using built-in)
- `image` (str | None): Pre-built image to use
- `base_image` (str | None): Base image for building
- `api_version` (str | None): API version

**Returns:**
- `str`: Complete docker-compose.yml file content

**Description:**
Generates a complete docker-compose.yml configuration for running LangGraph API server with all required services (Redis, Postgres, optional debugger).

**Example:**
```python
from langgraph_cli.exec import Runner
from langgraph_cli.docker import check_capabilities, compose

with Runner() as runner:
    caps = check_capabilities(runner)
    yaml = compose(
        caps,
        port=8123,
        debugger_port=8080
    )
    print(yaml)
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/docker.py:247`

---

### compose_as_dict

Generate docker-compose configuration as a dictionary.

**Signature:**
```python
def compose_as_dict(
    capabilities: DockerCapabilities,
    *,
    port: int,
    debugger_port: int | None = None,
    debugger_base_url: str | None = None,
    postgres_uri: str | None = None,
    image: str | None = None,
    base_image: str | None = None,
    api_version: str | None = None,
) -> dict:
```

**Parameters:**
- Same as `compose()` function

**Returns:**
- `dict`: Docker Compose configuration as a Python dictionary

**Description:**
Generates docker-compose configuration as a dictionary, which can be modified before converting to YAML.

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/docker.py:138`

---

### dict_to_yaml

Convert a dictionary to YAML format.

**Signature:**
```python
def dict_to_yaml(d: dict, *, indent: int = 0) -> str:
```

**Parameters:**
- `d` (dict): Dictionary to convert
- `indent` (int): Indentation level (default: 0)

**Returns:**
- `str`: YAML-formatted string

**Description:**
Simple YAML serializer for docker-compose dictionaries. Handles nested dicts, lists, and basic values.

**Example:**
```python
from langgraph_cli.docker import dict_to_yaml

config = {
    "services": {
        "app": {
            "image": "myapp:latest",
            "ports": ["8000:8000"]
        }
    }
}

yaml = dict_to_yaml(config)
print(yaml)
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/docker.py:117`

---

### DockerCapabilities

Named tuple representing Docker environment capabilities.

**Type:**
```python
class DockerCapabilities(NamedTuple):
    version_docker: Version
    version_compose: Version
    healthcheck_start_interval: bool
    compose_type: DockerComposeType = "plugin"
```

**Fields:**
- `version_docker` (Version): Docker engine version
- `version_compose` (Version): Docker Compose version
- `healthcheck_start_interval` (bool): Whether `start_interval` is supported in healthchecks
- `compose_type` (DockerComposeType): Either "plugin" or "standalone"

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/docker.py:25`

---

### Version

Named tuple representing a semantic version.

**Type:**
```python
class Version(NamedTuple):
    major: int
    minor: int
    patch: int
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/docker.py:16`

---

## Template API

Functions for creating new LangGraph projects from templates.

### create_new

Create a new LangGraph project from a template.

**Signature:**
```python
def create_new(path: str | None, template: str | None) -> None:
```

**Parameters:**
- `path` (str | None): Path where project will be created (prompts if None)
- `template` (str | None): Template ID to use (prompts if None)

**Raises:**
- `SystemExit`: If directory exists and is not empty, or template not found

**Description:**
Downloads and extracts a LangGraph project template to the specified path. If path or template are not provided, prompts user interactively.

**Available Template IDs:**
- `new-langgraph-project-python`
- `new-langgraph-project-js`
- `react-agent-python`
- `react-agent-js`
- `memory-agent-python`
- `memory-agent-js`
- `retrieval-agent-python`
- `retrieval-agent-js`
- `data-enrichment-agent-python`
- `data-enrichment-agent-js`

**Example:**
```python
from langgraph_cli.templates import create_new

# Interactive mode
create_new(None, None)

# Programmatic usage
create_new("./my-agent", "react-agent-python")
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/templates.py:164`

---

### TEMPLATES

Dictionary of available project templates.

**Type:**
```python
TEMPLATES: dict[str, dict[str, str]]
```

**Structure:**
```python
{
    "Template Name": {
        "description": "Template description",
        "python": "https://github.com/...",
        "js": "https://github.com/..."
    }
}
```

**Available Templates:**

| Name | Description |
|------|-------------|
| New LangGraph Project | A simple, minimal chatbot with memory |
| ReAct Agent | A simple agent that can be flexibly extended to many tools |
| Memory Agent | ReAct-style agent with memory storage across conversational threads |
| Retrieval Agent | Agent with retrieval-based question-answering system |
| Data-enrichment Agent | Agent that performs web searches and organizes findings |

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/templates.py:10`

---

### TEMPLATE_IDS

List of all available template IDs.

**Type:**
```python
TEMPLATE_IDS: list[str]
```

**Content:**
```python
[
    "new-langgraph-project-python",
    "new-langgraph-project-js",
    "react-agent-python",
    "react-agent-js",
    "memory-agent-python",
    "memory-agent-js",
    "retrieval-agent-python",
    "retrieval-agent-js",
    "data-enrichment-agent-python",
    "data-enrichment-agent-js"
]
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/templates.py:46`

---

## Execution API

Functions for executing subprocesses and managing async operations.

### Runner

Context manager for running async operations.

**Signature:**
```python
@contextmanager
def Runner():
```

**Yields:**
- Runner instance with a `run()` method

**Description:**
Context manager that provides a consistent interface for running async operations across different Python versions. Uses `asyncio.Runner` on Python 3.11+ and falls back to `asyncio.run()` on older versions.

**Example:**
```python
from langgraph_cli.exec import Runner, subp_exec

with Runner() as runner:
    stdout, stderr = runner.run(
        subp_exec("echo", "hello", collect=True)
    )
    print(stdout)
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/exec.py:12`

---

### subp_exec

Execute a subprocess asynchronously.

**Signature:**
```python
async def subp_exec(
    cmd: str,
    *args: str,
    input: str | None = None,
    wait: float | None = None,
    verbose: bool = False,
    collect: bool = False,
    on_stdout: Callable[[str], bool | None] | None = None,
) -> tuple[str | None, str | None]:
```

**Parameters:**
- `cmd` (str): Command to execute
- `*args` (str): Command arguments
- `input` (str | None): Input to send to stdin
- `wait` (float | None): Seconds to wait before executing
- `verbose` (bool): Print command and output
- `collect` (bool): Collect and return stdout/stderr
- `on_stdout` (Callable): Callback for each stdout line (return True to start displaying all output)

**Returns:**
- `tuple[str | None, str | None]`: (stdout, stderr) if collect=True, otherwise (None, None)

**Raises:**
- `click.exceptions.Exit`: If command returns non-zero exit code (except 130 for user interrupt)

**Description:**
Executes a subprocess asynchronously with support for:
- Stdin input
- Output collection or real-time display
- Signal handling (SIGINT, SIGTERM)
- Custom stdout line processing
- Verbose command display

**Example:**
```python
from langgraph_cli.exec import Runner, subp_exec

with Runner() as runner:
    # Simple execution
    runner.run(subp_exec("docker", "version", verbose=True))

    # Collect output
    stdout, _ = runner.run(
        subp_exec("docker", "images", "--format", "{{.Repository}}", collect=True)
    )

    # With callback
    def on_line(line):
        if "Complete" in line:
            print("Build finished!")
            return True  # Start displaying all output
        return False

    runner.run(subp_exec("docker", "build", ".", on_stdout=on_line))
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/exec.py:31`

---

### monitor_stream

Monitor an async stream and handle output.

**Signature:**
```python
async def monitor_stream(
    stream: asyncio.StreamReader,
    collect: bool = False,
    display: bool = False,
    on_line: Callable[[str], bool | None] | None = None,
) -> bytearray | None:
```

**Parameters:**
- `stream` (asyncio.StreamReader): Stream to monitor
- `collect` (bool): Collect output in buffer
- `display` (bool): Display output to stdout
- `on_line` (Callable): Callback for each line

**Returns:**
- `bytearray | None`: Collected output if collect=True

**Description:**
Internal function for monitoring subprocess output streams. Handles line buffering and limit overrun errors.

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/exec.py:126`

---

## Progress API

Classes for displaying progress spinners in the CLI.

### Progress

Context manager for displaying a progress spinner.

**Signature:**
```python
class Progress:
    def __init__(self, *, message=""):
```

**Parameters:**
- `message` (str): Initial message to display

**Methods:**

#### `__enter__`
Returns a callable that updates the progress message.

**Returns:**
- `Callable[[str], None]`: Function to update progress message

#### `__exit__`
Cleans up the progress spinner.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `delay` | float | Delay between spinner frames (default: 0.1s) |
| `message` | str | Current progress message |

**Description:**
Displays an animated spinner with a message while operations are running. Automatically detects if stdout is a TTY and adjusts behavior accordingly.

**Example:**
```python
from langgraph_cli.progress import Progress
import time

with Progress(message="Loading...") as set_progress:
    time.sleep(1)
    set_progress("Processing...")
    time.sleep(1)
    set_progress("Almost done...")
    time.sleep(1)
    set_progress("")  # Clear spinner

# Spinner automatically stops when context exits
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/progress.py:7`

---

## Utility Functions

Helper functions and constants used throughout the CLI.

### warn_non_wolfi_distro

Display a warning if the image distribution is not Wolfi.

**Signature:**
```python
def warn_non_wolfi_distro(config_json: dict) -> None:
```

**Parameters:**
- `config_json` (dict): Configuration dictionary

**Description:**
Shows a warning message recommending Wolfi Linux for enhanced security if the `image_distro` is not set to "wolfi".

**Example:**
```python
from langgraph_cli.util import warn_non_wolfi_distro

config = {"image_distro": "debian"}
warn_non_wolfi_distro(config)
# Displays warning about Wolfi Linux
```

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/util.py:8`

---

### clean_empty_lines

Remove empty lines from a string.

**Signature:**
```python
def clean_empty_lines(input_str: str) -> str:
```

**Parameters:**
- `input_str` (str): Input string with potential empty lines

**Returns:**
- `str`: String with empty lines removed

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/util.py:4`

---

### log_command

Decorator for logging CLI command usage analytics.

**Signature:**
```python
def log_command(func):
```

**Parameters:**
- `func`: Function to decorate

**Returns:**
- Decorated function that logs usage before execution

**Description:**
Decorator that sends anonymized usage analytics to track CLI command usage. Can be disabled by setting `LANGGRAPH_CLI_NO_ANALYTICS=1` environment variable.

**Logged Data:**
- Operating system and version
- Python version
- CLI version
- Command name
- Anonymized parameters (no sensitive data)

**Source:** `/home/user/langgraph/libs/cli/langgraph_cli/analytics.py:79`

---

### Constants

Important constants used throughout the CLI.

**Module:** `/home/user/langgraph/libs/cli/langgraph_cli/constants.py`

| Constant | Value | Description |
|----------|-------|-------------|
| `DEFAULT_CONFIG` | `"langgraph.json"` | Default configuration file name |
| `DEFAULT_PORT` | `8123` | Default port for LangGraph API server |
| `SUPABASE_URL` | (URL) | Analytics endpoint URL |
| `SUPABASE_PUBLIC_API_KEY` | (Key) | Public API key for analytics |

---

## Environment Variables

Environment variables recognized by the LangGraph CLI:

| Variable | Description |
|----------|-------------|
| `LANGSMITH_API_KEY` | Required for local development with LangSmith |
| `LANGGRAPH_CLOUD_LICENSE_KEY` | Required for production deployments |
| `LANGGRAPH_CLI_NO_ANALYTICS` | Set to `1` to disable usage analytics |

---

## Configuration File Versions

### Minimum Python Version
- **Required:** 3.11
- **Format:** "major.minor" (e.g., "3.11")

### Minimum Node.js Version
- **Required:** 20
- **Format:** "major" only (e.g., "20")

### Supported Image Distributions
- `debian` (default)
- `wolfi` (recommended for security)
- `bullseye`
- `bookworm`

---

## Error Handling

All CLI commands may raise:

- `click.UsageError`: For invalid command-line arguments or configuration errors
- `click.exceptions.Exit`: For subprocess failures
- `FileNotFoundError`: For missing configuration or dependency files
- `ValueError`: For invalid configuration values
- `SystemExit`: For fatal errors requiring program termination

---

## Best Practices

### Configuration
1. Always use absolute paths in configuration files
2. Use `.env` files for sensitive environment variables (never commit to git)
3. Specify exact versions for production deployments using `api_version`
4. Use Wolfi distribution for enhanced container security

### Docker
1. Use `--no-pull` for faster local development iterations
2. Pin to specific `api_version` for production deployments
3. Use `--watch` mode for active development
4. Keep Docker images small by setting `keep_pkg_tools: false`

### Development
1. Use `langgraph dev` for local development instead of `langgraph up`
2. Enable debug ports when troubleshooting
3. Use verbose mode (`--verbose`) to see detailed Docker build output
4. Test configurations with `langgraph dockerfile` before building

### Templates
1. Start with a template using `langgraph new` for new projects
2. Choose the appropriate template based on your use case
3. Customize the generated project to fit your needs

---

## Version Information

To get the CLI version:

```bash
langgraph --version
```

Or programmatically:

```python
from langgraph_cli.version import __version__
print(__version__)
```

---

## See Also

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [LangGraph Server Documentation](https://langchain-ai.github.io/langgraph/cloud/)
- [Docker Documentation](https://docs.docker.com/)
- [LangSmith Documentation](https://docs.smith.langchain.com/)
