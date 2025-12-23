# EE-Bench YAML Configuration Specifications

This directory contains YAML configuration specifications and the JSON Schema for validating benchmark configurations.

## Files

- **`benchmark-config-schema.json`**: JSON Schema defining the structure and validation rules for YAML configurations
- **`dpaia-jvm.yaml`**: Complete example configuration demonstrating all available options

## YAML Configuration Schema

The `benchmark-config-schema.json` file is the authoritative specification for YAML configuration structure. It:

- Defines all valid configuration properties and their types
- Documents available configurers and evaluators with their options
- Provides validation rules (required fields, enums, constraints)
- Enables IDE autocomplete and validation support

### Using the Schema

#### IDE Integration

Most modern IDEs support JSON Schema validation for YAML files. Configure your IDE to use the schema:

**VS Code:**
Add to your workspace settings (`.vscode/settings.json`):
```json
{
  "yaml.schemas": {
    "core/specs/benchmark-config-schema.json": ["core/specs/*.yaml", "*.yaml"]
  }
}
```

**IntelliJ IDEA / PyCharm:**
1. Open Settings → Languages & Frameworks → Schemas and DTDs → JSON Schema Mappings
2. Add new mapping:
   - Schema file: `core/specs/benchmark-config-schema.json`
   - Schema version: JSON Schema version 7
   - File path pattern: `**/*.yaml`

#### Command-Line Validation

Validate YAML files against the schema using `ajv-cli`:

```bash
# Install ajv-cli (one time)
npm install -g ajv-cli

# Validate a single file
ajv validate -s core/specs/benchmark-config-schema.json -d core/specs/dpaia-jvm.yaml

# Validate all YAML files
ajv validate -s core/specs/benchmark-config-schema.json -d "core/specs/*.yaml"
```

#### Python Validation

Validate programmatically using `jsonschema`:

```python
import json
import yaml
from jsonschema import validate, ValidationError

# Load schema
with open('core/specs/benchmark-config-schema.json', 'r') as f:
    schema = json.load(f)

# Load YAML config
with open('core/specs/dpaia-jvm.yaml', 'r') as f:
    config = yaml.safe_load(f)

# Validate
try:
    validate(instance=config, schema=schema)
    print("✓ Configuration is valid")
except ValidationError as e:
    print(f"✗ Validation error: {e.message}")
```

## Available Configurers

The schema documents the following configurers with their complete options.

### Configurers Directory Structure

After recent refactoring, configurers are organized into dedicated `configurers/` directories within each module for better maintainability:

**Core Module** (`core/src/ee_bench_core/adapters/configurers/`):
```
configurers/
├── bash_command.py          # BashCommandConfigurer + Factory
├── docker_image.py           # DockerImageConfigurer + Factory
└── git_checkout.py           # GitCheckoutConfigurer + Factory
```

**JVM-Spec Module** (`plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurers/`):
```
configurers/
├── base_docker_build_tool.py         # BaseDockerBuildToolConfigurer (shared base)
├── base_local_build_tool.py          # BaseLocalBuildToolConfigurer (shared base)
├── docker_dependencies.py            # DockerDependenciesConfigurer (base)
├── dependencies_factory.py           # DependenciesConfigurerFactory (multi-purpose)
├── jvm_base_image.py                 # JvmBaseImageConfigurer + JdkConfigurerFactory
├── local_jvm.py                      # LocalJvmConfigurer
├── local_task_setup.py               # LocalTaskSetupConfigurer
├── maven/
│   ├── docker_maven.py               # DockerMavenConfigurer
│   ├── docker_maven_dependencies.py  # DockerMavenDependenciesConfigurer + MavenDependenciesFactory
│   ├── local_maven.py                # LocalMavenConfigurer
│   └── local_maven_dependencies.py   # LocalMavenDependenciesConfigurer
└── gradle/
    ├── docker_gradle.py              # DockerGradleConfigurer
    ├── docker_gradle_dependencies.py # DockerGradleDependenciesConfigurer + GradleDependenciesFactory
    ├── local_gradle.py               # LocalGradleConfigurer
    └── local_gradle_dependencies.py  # LocalGradleDependenciesConfigurer
```

**Smart Factories** (still in `configurer_factories.py`):
- `LocalBuildToolConfigurerFactory` - Selects Maven/Gradle for local execution
- `DockerBuildToolConfigurerFactory` - Selects Maven/Gradle for Docker execution
- `MavenConfigurerFactory` - Creates Docker Maven configurer
- `GradleConfigurerFactory` - Creates Docker Gradle configurer

**Agents-Core Module** (`plugins/agents-core/src/ee_bench_agents/configurers/`):
```
configurers/
├── docker_environment.py     # DockerAgentsEnvironmentConfigurer
└── local_environment.py      # LocalAgentEnvironmentConfigurer
```

**Snyk-Validator Module** (`plugins/snyk-validator/src/snyk_plugin/configurers/`):
```
configurers/
└── snyk.py                   # SnykEnvironmentConfigurer + SnykConfigurerFactory
```

**Organization Principles**:
- **1:1 Relationship**: Configurer + Factory in same file (e.g., `git_checkout.py`)
- **Multi-Purpose Factories**: Separate files for factories that create multiple configurers (e.g., `dependencies_factory.py`)
- **Build-Tool Specific**: Subdirectories for Maven/Gradle specific implementations
- **Entry Points**: Updated in `pyproject.toml` to reference new paths

### Generic Docker Configurers

#### `docker_image` - Generic Docker Image Builder

Builds custom Docker images from Dockerfile content or FileSource. This is a generic configurer that can be used to create any Docker image.

**Factory**: `DockerImageConfigurerFactory` (core/src/ee_bench_core/adapters/configurers/docker_image.py)

**Options**:
- `dockerfile` (string, optional): Direct Dockerfile content as multiline string. **Supports template substitution.**
  - Template format: Can use `{instance.property}`, `{instance.property:default}`, `{$ENV_VAR}`, etc.
  - Must provide either `dockerfile` or `dockerfile_source`
- `dockerfile_source` (object, optional): FileSource configuration to load Dockerfile from file or HTTP
  - `type` (string): Source type - `"file"` or `"http"`
  - `path` (string): File path (required if type="file")
  - `uri` (string): HTTP/HTTPS URL (required if type="http")
- `tag` (string): Docker image tag
  - Template format: `{instance.property:default}`
  - Default: `"custom"`
  - Example: `"base:jvm-jdk{instance.jvm_version:24}"`
- `labels` (object): Dictionary of labels to apply to the Docker image
  - All values support template substitution
  - Example: `{"language": "jvm", "version": "{instance.jvm_version:24}"}`
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Use Cases**:
- **Replacing JDK Configurer**: Build custom JDK base images with specific system tools
- **Custom Base Images**: Create specialized base images for any language or framework
- **Multi-stage Builds**: Define complex build pipelines in Dockerfile
- **External Dockerfiles**: Load Dockerfiles from files or URLs

**Example - Custom JDK Base Image (Replaces `jdk` configurer)**:
```yaml
configurers:
  - name: docker_image
    options:
      dockerfile: |
        FROM eclipse-temurin:{instance.jvm_version:24}-jdk

        # Install system deps
        RUN apt-get update && apt-get install -y \
            git wget curl unzip patch jq openssh-client ca-certificates build-essential \
            && rm -rf /var/lib/apt/lists/*

        # Set up SSH for git
        RUN mkdir -p /root/.ssh && ssh-keyscan github.com >> /root/.ssh/known_hosts

        WORKDIR /workspace
      tag: "base:jvm-jdk{instance.jvm_version:24}"
      labels:
        language: "jvm"
        jdk-version: "{instance.jvm_version:24}"
        distribution: "temurin"
```

**Example - Load from File**:
```yaml
configurers:
  - name: docker_image
    options:
      dockerfile_source:
        type: file
        path: "/path/to/Dockerfile"
      tag: "my-custom-image"
      labels:
        version: "1.0"
```

**Example - Load from HTTP**:
```yaml
configurers:
  - name: docker_image
    options:
      dockerfile_source:
        type: http
        uri: "https://raw.githubusercontent.com/org/repo/main/Dockerfile"
      tag: "remote-image"
```

---

### JVM Configurers

#### `jdk` - JDK Base Image Configurer

> **⚠️ Deprecated**: Consider using `docker_image` configurer instead for more flexibility.
> See the `docker_image` section above for an example of how to create custom JDK base images.

Creates Docker base image with specified JDK version.

**Factory**: `JdkConfigurerFactory` (plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurers/jvm_base_image.py)

**Options**:
- `version` (string): JDK version to use. Supports template substitution from instance properties.
  - Template format: `{instance.jvm_version:24}` (uses instance property with fallback)
  - Examples: `"17"`, `"21"`, `"24"`, `"{instance.jvm_version:24}"`
  - Default: `"24"`
- `distribution` (string): JDK distribution name
  - Examples: `"temurin"`, `"openjdk"`
  - Default: `"temurin"`
- `namespace` (string): Docker image namespace for tagging images
  - Default: `"ee-bench"`

**Precedence** (highest to lowest):
1. CLI arguments (`--jvm-version`, `--default-jdk-version`)
2. YAML `options.version` (after template resolution)
3. Default value `"24"`

**Example**:
```yaml
configurers:
  - name: jdk
    options:
      version: "{instance.jvm_version:24}"
      distribution: temurin
      namespace: ee-bench
    env:
      JAVA_TOOL_OPTIONS: "-Xmx2g -Xms512m"
```

---

#### `build_system` - Auto-Detect Build System

Automatically selects Maven or Gradle configurer based on build system detection or configuration.

**Factory**: `DockerBuildToolConfigurerFactory` (plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurer_factories.py:91)

**Options**:
- `build_system` (string, optional): Build system selection. **Supports template resolution.**
  - Template format: `{instance.build_system}`, `{instance.build_system:maven}` (with default)
  - Literal values: `"maven"` or `"gradle"`
  - Examples: `"{instance.build_system}"`, `"maven"`, `"{instance.build_system:maven}"`
  - If not specified, defaults to `"maven"`
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Resolution Order** (highest to lowest):
1. CLI argument `--build-system`
2. YAML `options.build_system` (after template resolution with instance data)
3. Default to `"maven"`

**How It Works**:
The factory resolves the `build_system` template using instance properties, then selects the appropriate configurer:
- `"maven"` → Creates `DockerMavenConfigurer`
- `"gradle"` → Creates `DockerGradleConfigurer`

**Example with Template**:
```yaml
configurers:
  - name: build_system
    options:
      build_system: "{instance.build_system}"  # Resolves from instance.build_system property
      namespace: ee-bench
    env:
      MAVEN_OPTS: "-Xmx1g"
      GRADLE_OPTS: "-Xmx1g"
```

**Example with Literal**:
```yaml
configurers:
  - name: build_system
    options:
      build_system: "maven"  # Explicit Maven
```

---

#### `maven` - Maven Build Tool

Configures Docker image with Maven build tool.

**Factory**: `MavenConfigurerFactory` (plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurer_factories.py:159)

**Options**:
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Example**:
```yaml
configurers:
  - name: maven
    options:
      namespace: ee-bench
```

---

#### `gradle` - Gradle Build Tool

Configures Docker image with Gradle build tool.

**Factory**: `GradleConfigurerFactory` (plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurer_factories.py:205)

**Options**:
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Example**:
```yaml
configurers:
  - name: gradle
    options:
      namespace: ee-bench
```

---

#### `dependencies` - Dependency Resolution (Maven or Gradle)

Automatically selects Maven or Gradle dependencies configurer based on build system and environment type (Docker/Local).

**Factory**: `DependenciesConfigurerFactory` (plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurers/dependencies_factory.py)

**Options**:
- `build_system` (string, optional): Build system selection for dependency resolution. **Supports template resolution.**
  - Template format: `{instance.build_system}`, `{instance.build_system:maven}` (with default)
  - Literal values: `"maven"` or `"gradle"`
  - Examples: `"{instance.build_system}"`, `"maven"`, `"{instance.build_system:maven}"`
  - If not specified, defaults to `"maven"`
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Resolution Order** (highest to lowest):
1. CLI argument `--build-system`
2. YAML `options.build_system` (after template resolution with instance data)
3. Default to `"maven"`

**How It Works**:
The factory resolves the `build_system` template using instance properties, then delegates to specific factories:
- `"maven"` → Delegates to `MavenDependenciesFactory` → Creates `DockerMavenDependenciesConfigurer` or `LocalMavenDependenciesConfigurer`
- `"gradle"` → Delegates to `GradleDependenciesFactory` → Creates `DockerGradleDependenciesConfigurer` or `LocalGradleDependenciesConfigurer`

The factory also supports `sandbox_type` option to choose between Docker and Local implementations:
- `sandbox_type: "docker"` (default) → Creates Docker dependencies configurer
- `sandbox_type: "local"` → Creates Local dependencies configurer

**Configurers Directory Structure**:
Dependencies configurers follow a modular organization:
```
plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurers/
├── docker_dependencies.py           # Base DockerDependenciesConfigurer
├── dependencies_factory.py          # DependenciesConfigurerFactory (selects Maven/Gradle + Docker/Local)
├── maven/
│   └── docker_maven_dependencies.py # DockerMavenDependenciesConfigurer + MavenDependenciesFactory
└── gradle/
    └── docker_gradle_dependencies.py # DockerGradleDependenciesConfigurer + GradleDependenciesFactory
```

This structure co-locates configurers with their factories when there's a 1:1 relationship, and separates multi-purpose factories into dedicated files.

**Example with Template**:
```yaml
configurers:
  - name: dependencies
    options:
      build_system: "{instance.build_system}"  # Resolves from instance.build_system property
      namespace: ee-bench
```

**Example with Literal**:
```yaml
configurers:
  - name: dependencies
    options:
      build_system: "gradle"  # Explicit Gradle
```

---

#### `maven_dependencies` - Maven Dependency Resolution

Resolves Maven project dependencies specifically for Docker environments.

**Factory**: `MavenDependenciesFactory` (plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurers/maven/docker_maven_dependencies.py)

**Options**:
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Example**:
```yaml
configurers:
  - name: maven_dependencies
    options:
      namespace: ee-bench
```

---

#### `gradle_dependencies` - Gradle Dependency Resolution

Resolves Gradle project dependencies specifically for Docker environments.

**Factory**: `GradleDependenciesFactory` (plugins/jvm-spec/src/ee_bench_jvm_spec/jvm/configurers/gradle/docker_gradle_dependencies.py)

**Options**:
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Example**:
```yaml
configurers:
  - name: gradle_dependencies
    options:
      namespace: ee-bench
```

---

#### `git_checkout` - Git Repository Checkout

Clones a Git repository and checks out a specific commit in a Docker image. **Supports template resolution** for all string options.

**Factory**: `GitCheckoutConfigurerFactory` (core/src/ee_bench_core/adapters/configurers/git_checkout.py)

**Options**:
- `url` (string, optional): Repository URL. **Supports templates.**
  - Template format: `{instance.repo}`, `{$ENV_VAR}`, `{cli_arg}`
  - Examples:
    - `"https://github.com/{instance.repo}"`
    - `"https://{$GITHUB_TOKEN}@github.com/{instance.repo}"`
    - `"https://gitlab.com/myorg/{instance.project}.git"`
  - If not specified, tries to build from `instance.repo`
- `base_commit` (string, optional): Commit SHA to checkout. **Supports templates.**
  - Template format: `{instance.base_commit}`, `{instance.commit_hash}`
  - Examples: `"{instance.base_commit}"`, `"abc123def456"`
  - If not specified, uses `instance.base_commit`
- `target_dir` (string, optional): Target directory for cloned repository. **Supports templates.**
  - Examples:
    - `"/workspace/task_project"` (explicit)
    - `"/workspace/{instance.project_name}"` (template)
    - `"{workspace_dir}/project"` (template)
  - **Default resolution order**:
    1. YAML `options.target_dir`
    2. `EnvironmentConfiguration.workspace_dir` + `EnvironmentConfiguration.project_dir`
    3. Default: `"/workspace/task_project"`
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**How It Works**:
1. Resolves all template variables using instance properties and environment variables
2. Clones the repository at the specified URL
3. Checks out the specified commit
4. Creates a Docker image with the checked-out code

**Example with Templates**:
```yaml
environment:
  workspace_dir: "/workspace"
  project_dir: "task_project"  # Used as fallback for target_dir

configurers:
  - name: git_checkout
    options:
      url: "https://github.com/{instance.repo}"
      base_commit: "{instance.base_commit}"
      # target_dir omitted - uses workspace_dir + project_dir
```

**Example with Explicit Values**:
```yaml
configurers:
  - name: git_checkout
    options:
      url: "https://github.com/spring-projects/spring-boot"
      base_commit: "abc123def456"
      target_dir: "/workspace/my_project"
```

**Example with Environment Variables**:
```yaml
configurers:
  - name: git_checkout
    options:
      url: "https://{$GITHUB_TOKEN}@github.com/{instance.repo}"
      base_commit: "{instance.base_commit}"
```

---

#### `bash_command` - Execute Bash Commands

Executes arbitrary bash commands during environment setup. Useful for custom configuration steps, installing tools, or setting up environment.

**Factory**: `BashCommandConfigurerFactory` (core/src/ee_bench_core/adapters/configurers/bash_command.py)

**Options**:
- `command` (string, required): Bash command or script to execute. **Supports template resolution.**
  - Template format: `{instance.property}`, `{$ENV_VAR}`, etc.
  - Can be multi-line script using YAML literal block scalar `|`
  - Examples:
    - Single command: `"apt-get update && apt-get install -y curl"`
    - Multi-line script: See example below
- `working_dir` (string, optional): Working directory for command execution
  - Default: `/workspace` (or current directory)
  - Supports template substitution
- `shell` (string, optional): Shell to use for execution
  - Default: `"/bin/bash"`
  - Examples: `"/bin/bash"`, `"/bin/sh"`, `"/bin/zsh"`
- `timeout` (integer, optional): Command timeout in seconds
  - Default: `300` (5 minutes)
  - Minimum: `1`
- `fail_on_error` (boolean, optional): Whether to fail environment setup if command fails
  - Default: `true`
  - Set to `false` to continue setup even if command fails
- `namespace` (string): Docker image namespace
  - Default: `"ee-bench"`

**Use Cases**:
- Install additional system tools not covered by other configurers
- Run custom setup scripts
- Configure system settings
- Download and setup resources
- Execute initialization commands

**Example - Install System Tools**:
```yaml
configurers:
  - name: bash_command
    options:
      command: |
        #!/bin/bash
        set -e

        # Update package lists
        apt-get update

        # Install development tools
        apt-get install -y \
          curl wget git jq \
          build-essential \
          python3-dev python3-pip

        # Install Node.js
        curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
        apt-get install -y nodejs

        # Verify installations
        echo "Installed versions:"
        git --version
        python3 --version
        node --version
        npm --version
      timeout: 600
      fail_on_error: true
```

**Example - Setup Application Configuration**:
```yaml
configurers:
  - name: bash_command
    options:
      command: |
        # Create application directories
        mkdir -p /opt/app/config /opt/app/logs

        # Generate configuration file
        cat > /opt/app/config/app.conf <<EOF
        {
          "environment": "test",
          "log_level": "debug",
          "database": {
            "host": "localhost",
            "port": 5432
          }
        }
        EOF

        # Set permissions
        chmod 644 /opt/app/config/app.conf
        chmod 755 /opt/app/logs
      working_dir: /opt/app
```

**Example - Download Resources with Template**:
```yaml
configurers:
  - name: bash_command
    options:
      command: |
        # Download dataset specific to this instance
        curl -L "https://example.com/datasets/{instance.dataset_id}.tar.gz" \
          -o /tmp/dataset.tar.gz

        # Extract to workspace
        tar -xzf /tmp/dataset.tar.gz -C /workspace

        # Cleanup
        rm /tmp/dataset.tar.gz
      timeout: 300
```

**Example - Conditional Setup**:
```yaml
configurers:
  - name: bash_command
    options:
      command: |
        # Install dependencies based on project type
        if [ -f "requirements.txt" ]; then
          echo "Python project detected"
          pip3 install -r requirements.txt
        elif [ -f "package.json" ]; then
          echo "Node.js project detected"
          npm install
        elif [ -f "pom.xml" ]; then
          echo "Maven project detected"
          mvn dependency:resolve
        else
          echo "Unknown project type"
        fi
      working_dir: /workspace/project
      fail_on_error: false  # Continue even if no dependencies found
```

---

### Agent Configurers

#### `docker_agents` - Docker Agent Environment Setup

Configures agent environments in Docker containers. Supports using agents declared at the top level or defining new agents inline.

**Factory**: `DockerAgentsEnvironmentConfigurerFactory` (plugins/agents-core/src/ee_bench_agents/configurer_factories.py:157)

**Options**:
- `agents` (list, optional): Agent configurations. Can be:
  - List of strings (agent names): `["claude-code", "gemini"]` - Uses top-level agent definitions
  - List of dicts (agent configs): `[{"name": "gemini", "version": "1.0"}]` - Defines/overrides agents
  - Mixed: `["claude-code", {"name": "gemini", "version": "1.0"}]`
  - If omitted: Uses all agents from `BenchmarkConfiguration.agents`
- `namespace` (string): Docker image namespace for agent images
  - Default: `"agents"`

**How It Works**:
1. If `agents` not specified: Uses all top-level agents from `agents:` section
2. If agent referenced by name (string): Uses properties from top-level agent
3. If agent defined as dict: Overrides top-level properties or defines new agent
4. Creates Docker images with configured agents

**Example - Use All Top-Level Agents**:
```yaml
agents:
  - name: claude-code
    version: "1.0"

configurers:
  - name: docker_agents  # Uses claude-code from top level
```

**Example - Select Specific Agents**:
```yaml
agents:
  - name: claude-code
    version: "1.0"
  - name: gemini
    version: "2.0"

configurers:
  - name: docker_agents
    options:
      agents:
        - claude-code  # Only configure claude-code
```

**Example - Override Agent Properties**:
```yaml
agents:
  - name: claude-code
    version: "1.0"
    timeout: 300

configurers:
  - name: docker_agents
    options:
      agents:
        - name: claude-code
          timeout: 600  # Override timeout
          env:
            CUSTOM_VAR: "value"
```

**Example - Mix Top-Level and Inline Agents**:
```yaml
agents:
  - name: claude-code
    version: "1.0"

configurers:
  - name: docker_agents
    options:
      agents:
        - claude-code  # Use from top level
        - name: gemini  # Define new agent inline
          version: "2.0"
          env:
            GEMINI_API_KEY: "${GEMINI_API_KEY}"
```

---

#### `local_agents` - Local Agent Environment Validation

Validates agent availability on the local system. Does not install agents; they must be pre-installed.

**Factory**: `LocalAgentEnvironmentConfigurerFactory` (plugins/agents-core/src/ee_bench_agents/configurer_factories.py:401)

**Options**:
- `agents` (list, optional): Agent configurations to validate. Same format as `docker_agents`.
  - If omitted: Validates all agents from `BenchmarkConfiguration.agents`

**Note**: Local configurers only validate that agents are available on the host system. Agents must be installed separately.

**Example**:
```yaml
environment:
  sandbox:
    type: local

agents:
  - name: claude-code
    version: "1.0"

configurers:
  - name: local_agents  # Validates claude-code on host
```

---

### Security Configurers

#### `snyk` - Snyk Security Scanner

Installs and configures Snyk security scanning tool.

**Factory**: `SnykConfigurerFactory` (plugins/snyk-validator/src/snyk_plugin/configurers/snyk.py)

**Options**:
- `install_method` (string): Snyk installation method
  - Values: `"npm"`, `"curl"`, or `"binary"`
  - Default: `"npm"`
- `verify_install` (boolean): Whether to verify Snyk installation succeeded
  - Default: `true`
- `token` (string): Snyk API token for authentication
  - **Recommendation**: Use environment variable substitution: `${SNYK_TOKEN}`
  - Required for Snyk operations (scanning, reporting)

**CLI Overrides**:
- `--snyk-token`: Overrides `options.token`

**Example**:
```yaml
configurers:
  - name: snyk
    options:
      install_method: npm
      verify_install: true
      token: "${SNYK_TOKEN:?Snyk token is required}"
```

---

### Common Configurer Options

All configurers support these common mechanisms:

#### Force Rebuild

Control whether to force rebuild of Docker images:

**Precedence** (highest to lowest):
1. CLI arguments: `--force` or `--force-rebuild`
2. Environment configuration: `environment.force_rebuild`
3. Configurer options: `options.force` or `options.force_rebuild`

**Example**:
```yaml
environment:
  force_rebuild: true  # Global force rebuild
  configurers:
    - name: jdk
      options:
        force: true  # Configurer-specific force rebuild
```

#### Environment Variables

All configurers support environment variable configuration:

```yaml
configurers:
  - name: jdk
    options:
      version: "17"
    env:
      # Configurer-specific environment variables
      JAVA_TOOL_OPTIONS: "-Xmx2g"
      JDK_DEBUG: "false"
```

Environment variables are passed to the configurer and can be accessed during configuration.

## Available Evaluators

Evaluators are components that assess code quality, test results, and patches during evaluation. Each evaluator can be configured with specific options and can contribute to the overall evaluation score.

### Evaluator Configuration

All evaluators share common configuration properties:

**Common Properties**:
- `name` (string, required): Evaluator instance name for referencing in pipeline
- `type` (string, required): Evaluator type/factory name (see types below)
- `description` (string, optional): Evaluator description
- `enabled` (boolean, optional): Enable/disable this evaluator (default: `true`)
- `is_terminal` (boolean, optional): Mark as terminal evaluator - stops pipeline if fails (default: `false`)
- `max_score` (number, optional): Maximum score for this evaluator (default varies by type)
- `timeout` (integer, optional): Evaluator timeout in seconds (default varies by type)
- `retry_count` (integer, optional): Number of retries on failure (default: `0`)
- `retry_delay` (integer, optional): Delay between retries in seconds (default: `0`)
- `continue_on_failure` (boolean, optional): Continue evaluation chain if this evaluator fails (default: `true`)
- `options` (object, optional): Evaluator-specific options (see each evaluator below)

### Evaluator Types

---

#### `project_reset` - Reset Project to Original State

Resets the project to its original state before applying patches or predictions. Essential for ensuring clean state between evaluations.

**Options**: None

**Use Cases**:
- Reset git repository to base commit
- Clean build artifacts
- Restore original files before applying predictions
- Prepare clean state for next evaluation

**Example**:
```yaml
evaluations:
  - id: gold-eval
    evaluators:
      - name: reset_before_test
        type: project_reset
        description: "Reset project to base state"
        timeout: 60
```

---

#### `test_runner` - Run Tests with Expectations

Executes tests using the project's build tool (Maven, Gradle, etc.) and evaluates results against expectations.

**Options**:
- `tests` (string or array, optional): Test pattern(s) to run. **Supports templates.**
  - String patterns:
    - `"*"`: Run all tests
    - `"com.example.MyTest"`: Run specific test class
    - `"{instance.fail_to_pass}"`: Use instance property with test names
  - Array of patterns: `["TestClass1", "TestClass2"]`
  - Template format: `{instance.test_names}`, `{instance.fail_to_pass}`
  - Default: `"*"` (all tests)
- `expect_pass` (boolean, optional): Whether tests are expected to pass
  - `true`: Tests should pass (success = all pass, failure = any fail)
  - `false`: Tests should fail (success = tests fail as expected)
  - Default: `true`
- `timeout` (integer, optional): Test execution timeout in seconds
  - Default: `1800` (30 minutes)
- `fail_fast` (boolean, optional): Stop on first test failure
  - Default: `false`
- `verbose` (boolean, optional): Enable verbose test output
  - Default: `false`
- `parallel` (boolean, optional): Run tests in parallel (if supported by build tool)
  - Default: `false`

**Scoring**:
- Score = (passed_tests / total_tests) * max_score
- If `expect_pass=false`, inverted scoring applies

**Example - Run All Tests**:
```yaml
evaluators:
  - name: run_all_tests
    type: test_runner
    description: "Execute all project tests"
    max_score: 100
    timeout: 1800
    options:
      tests: "*"
      expect_pass: true
      verbose: true
```

**Example - Run Specific Tests from Instance**:
```yaml
evaluators:
  - name: run_fail_to_pass
    type: test_runner
    description: "Run tests that should change from fail to pass"
    max_score: 50
    options:
      tests: "{instance.fail_to_pass}"  # Comma-separated test names
      expect_pass: true
      timeout: 600
```

**Example - Run Multiple Test Patterns**:
```yaml
evaluators:
  - name: run_integration_tests
    type: test_runner
    options:
      tests:
        - "com.example.integration.*"
        - "com.example.e2e.*"
      expect_pass: true
      parallel: true
```

**Example - Expect Failure (Negative Testing)**:
```yaml
evaluators:
  - name: verify_broken_tests
    type: test_runner
    description: "Verify tests fail before fix"
    options:
      tests: "{instance.fail_to_pass}"
      expect_pass: false  # Tests should fail
```

---

#### `patch_application` - Apply Patches from Dataset

Applies patches (git diff format) from dataset to the codebase. Used to apply gold patches or test patches.

**Options**:
- `patch` (string, required): Patch content or template. **Supports templates.**
  - Direct patch content: Multi-line git diff format
  - Template reference: `{instance.patch}`, `{instance.test_patch}`
  - Template format: Can reference any instance field
- `patch_type` (string, optional): Type of patch being applied
  - Values: `"gold"`, `"test"`, `"custom"`
  - Default: `"gold"`
  - Used for logging and reporting
- `can_skip_patch` (boolean, optional): Allow skipping if patch is empty
  - `true`: Skip evaluation if patch is empty (no error)
  - `false`: Fail evaluation if patch is empty
  - Default: `false`
- `reverse` (boolean, optional): Apply patch in reverse (undo changes)
  - Default: `false`
- `strip` (integer, optional): Number of leading path components to strip
  - Default: `1` (strip `a/` and `b/` prefixes)
- `working_dir` (string, optional): Directory to apply patch in
  - Default: Project root directory
  - Supports template substitution
- `dry_run` (boolean, optional): Test patch application without actually applying
  - Default: `false`
- `ignore_whitespace` (boolean, optional): Ignore whitespace changes
  - Default: `false`
- `reject_file` (string, optional): Path to save rejected hunks
  - Default: None (rejected hunks not saved)

**Example - Apply Gold Patch**:
```yaml
evaluators:
  - name: apply_gold_patch
    type: patch_application
    description: "Apply gold solution patch"
    is_terminal: true  # Stop if patch fails
    options:
      patch: "{instance.patch}"
      patch_type: "gold"
      can_skip_patch: false
```

**Example - Apply Test Patch**:
```yaml
evaluators:
  - name: apply_test_patch
    type: patch_application
    description: "Apply test cases"
    options:
      patch: "{instance.test_patch}"
      patch_type: "test"
      can_skip_patch: true  # Some instances may not have test patches
```

**Example - Apply Custom Patch with Dry Run**:
```yaml
evaluators:
  - name: verify_patch
    type: patch_application
    description: "Verify patch can be applied"
    options:
      patch: |
        diff --git a/src/Main.java b/src/Main.java
        index abc123..def456 100644
        --- a/src/Main.java
        +++ b/src/Main.java
        @@ -10,7 +10,7 @@ public class Main {
         }
      dry_run: true  # Don't actually apply
      reject_file: /tmp/patch-rejects.txt
```

---

#### `prediction_application` - Apply AI/Agent Predictions

Applies predictions generated by AI agents or from files. Similar to patch application but specifically for predictions.

**Options**:
- `prediction_source` (string, optional): Source of prediction
  - Values: `"file"`, `"agent"`, `"inline"`
  - Default: Inferred from configuration
- `prediction_id` (string, optional): ID of prediction to apply
  - References prediction from `predictions` section
  - Default: Primary prediction
- `format` (string, optional): Prediction format
  - Values: `"patch"`, `"diff"`, `"json"`, `"files"`
  - Default: `"patch"` (git diff format)
- `apply_method` (string, optional): How to apply prediction
  - Values: `"patch"`, `"replace"`, `"merge"`
  - Default: `"patch"`
- `validation` (boolean, optional): Validate prediction before applying
  - Default: `true`
- `backup` (boolean, optional): Create backup of original files
  - Default: `false`
- `can_skip_empty` (boolean, optional): Skip if prediction is empty
  - Default: `true`

**Example - Apply Agent Prediction**:
```yaml
evaluators:
  - name: apply_agent_solution
    type: prediction_application
    description: "Apply agent-generated solution"
    is_terminal: true
    options:
      prediction_id: "agent-prediction"
      format: "patch"
      validation: true
      backup: true
```

**Example - Apply Prediction with Custom Format**:
```yaml
evaluators:
  - name: apply_json_prediction
    type: prediction_application
    options:
      prediction_source: "file"
      format: "json"
      apply_method: "replace"
```

---

#### `regression_detection` - Detect Test Regressions

Compares test results before and after changes to detect regressions (previously passing tests that now fail).

**Options**:
- `baseline_results` (string, optional): Path to baseline test results
  - Template format: `{instance.baseline_tests}`
  - Default: Results from previous evaluator in pipeline
- `comparison_mode` (string, optional): How to compare results
  - Values: `"strict"`, `"lenient"`
  - `"strict"`: Any new failure is a regression
  - `"lenient"`: Only consider tests that were explicitly passing before
  - Default: `"strict"`
- `fail_on_regression` (boolean, optional): Fail evaluation if regression detected
  - Default: `true`
- `allow_new_failures` (boolean, optional): Allow new test failures (not regressions)
  - `true`: New tests can fail without causing regression
  - `false`: Any failure is considered a regression
  - Default: `false`
- `ignore_flaky` (boolean, optional): Ignore known flaky tests
  - Default: `false`
- `flaky_tests` (array, optional): List of known flaky test names to ignore
  - Example: `["FlakyTest1", "FlakyTest2"]`
- `report_path` (string, optional): Path to save regression report
  - Default: None (no report saved)

**Scoring**:
- Score = 100 if no regressions detected
- Score = 0 if regressions detected
- Can be weighted in scoring configuration

**Example - Detect Regressions After Patch**:
```yaml
evaluations:
  - id: regression-check
    evaluators:
      # 1. Run tests before patch (baseline)
      - name: baseline_tests
        type: test_runner
        options:
          tests: "*"
          expect_pass: true

      # 2. Apply patch
      - name: apply_patch
        type: patch_application
        options:
          patch: "{instance.patch}"

      # 3. Run tests after patch
      - name: after_patch_tests
        type: test_runner
        options:
          tests: "*"
          expect_pass: true

      # 4. Detect regressions
      - name: check_regressions
        type: regression_detection
        description: "Ensure no previously passing tests now fail"
        options:
          comparison_mode: "strict"
          fail_on_regression: true
          report_path: "reports/regression-{instance_id}.json"
```

**Example - Lenient Regression Detection**:
```yaml
evaluators:
  - name: lenient_regression_check
    type: regression_detection
    options:
      comparison_mode: "lenient"
      allow_new_failures: true  # New tests can fail
      ignore_flaky: true
      flaky_tests:
        - "com.example.FlakyIntegrationTest"
        - "com.example.TimeDependentTest"
```

---

### Complete Evaluation Pipeline Example

```yaml
evaluations:
  # Evaluation 1: Gold Patch Baseline
  - id: gold-baseline
    name: "Gold Patch Evaluation"
    description: "Evaluate correctness of gold patch"
    evaluators:
      # Reset to clean state
      - name: reset_project
        type: project_reset
        timeout: 60

      # Apply gold patch
      - name: apply_gold
        type: patch_application
        description: "Apply gold solution"
        is_terminal: true  # Stop if patch fails
        options:
          patch: "{instance.patch}"
          patch_type: "gold"

      # Run all tests
      - name: test_gold
        type: test_runner
        description: "Run all tests with gold patch"
        max_score: 100
        timeout: 1800
        options:
          tests: "*"
          expect_pass: true
          verbose: true

    scoring:
      method: "sum"
      evaluators: ["test_gold"]  # Only test_gold contributes to score

    output:
      path: "reports/gold-baseline-{run_id}.json"
      pretty: true

  # Evaluation 2: Agent Prediction with Regression Detection
  - id: agent-eval
    name: "Agent Prediction Evaluation"
    description: "Evaluate agent-generated solution with regression detection"
    evaluators:
      # Reset to clean state
      - name: reset_project
        type: project_reset

      # Baseline: Run tests before agent changes
      - name: baseline_tests
        type: test_runner
        description: "Baseline tests (should mostly pass)"
        options:
          tests: "*"
          expect_pass: true

      # Apply agent prediction
      - name: apply_agent_prediction
        type: prediction_application
        description: "Apply agent-generated solution"
        is_terminal: true
        options:
          prediction_id: "agent"
          format: "patch"
          validation: true
          backup: true

      # Run tests after agent changes
      - name: test_agent_solution
        type: test_runner
        description: "Test agent solution"
        max_score: 100
        timeout: 1800
        options:
          tests: "*"
          expect_pass: true
          verbose: true

      # Check for regressions
      - name: regression_check
        type: regression_detection
        description: "Ensure no regressions introduced"
        max_score: 50
        options:
          comparison_mode: "strict"
          fail_on_regression: true
          report_path: "reports/regressions-{instance_id}.json"

      # Run specific fail-to-pass tests
      - name: test_fail_to_pass
        type: test_runner
        description: "Verify fail-to-pass tests now pass"
        max_score: 50
        options:
          tests: "{instance.fail_to_pass}"
          expect_pass: true

    scoring:
      method: "weighted_sum"
      weights:
        test_agent_solution: 0.5
        regression_check: 0.3
        test_fail_to_pass: 0.2
      normalize: true

    output:
      path: "reports/agent-eval-{run_id}.json"
      pretty: true
      console_output: true

  # Evaluation 3: Test-Driven Development (TDD)
  - id: tdd-eval
    name: "TDD Evaluation"
    description: "Apply test patch first, then solution"
    evaluators:
      # Reset
      - name: reset
        type: project_reset

      # Apply test patch
      - name: apply_tests
        type: patch_application
        description: "Apply test cases"
        options:
          patch: "{instance.test_patch}"
          patch_type: "test"
          can_skip_patch: true

      # Verify tests fail initially
      - name: verify_tests_fail
        type: test_runner
        description: "Tests should fail before solution"
        options:
          tests: "{instance.fail_to_pass}"
          expect_pass: false  # Should fail

      # Apply solution
      - name: apply_solution
        type: patch_application
        description: "Apply solution patch"
        options:
          patch: "{instance.patch}"

      # Verify tests pass after solution
      - name: verify_tests_pass
        type: test_runner
        description: "Tests should pass after solution"
        max_score: 100
        options:
          tests: "{instance.fail_to_pass}"
          expect_pass: true

    scoring:
      method: "sum"
      evaluators: ["verify_tests_pass"]

    output:
      path: "reports/tdd-eval-{run_id}.json"
```

### Evaluator Best Practices

1. **Use `is_terminal` for Critical Evaluators**: Mark patch application as terminal to stop pipeline if patch fails
2. **Reset Between Evaluations**: Always start with `project_reset` for clean state
3. **Baseline Before Changes**: Run tests before applying patches to establish baseline
4. **Regression Detection**: Use `regression_detection` after changes to ensure no breakage
5. **Timeouts**: Set appropriate timeouts based on project size (larger projects need more time)
6. **Retry Failed Tests**: Use `retry_count` for flaky tests, but investigate root cause
7. **Scoring Strategy**: Use weighted scoring for complex evaluations with multiple criteria
8. **Verbose Output**: Enable `verbose: true` during development, disable in production

## Agent Configuration

Agents can be configured with flexible script support for installation, version checking, and execution. Scripts support multiple formats from simple inline commands to complex multi-file specifications.

### Agent Scripts

Agents support three script types:

- **`install_command`**: Script for installing the agent
- **`version_command`**: Script for checking agent version
- **`entry_command`**: Script for running the agent

Each script field supports three formats:

#### 1. Inline String (Simple Shell Command)

```yaml
agents:
  - name: my-agent
    version_command: "my-agent --version"
    entry_command: "my-agent run"
```

#### 2. Type Alias (Language-Specific)

Supported aliases: `shell`, `bash`, `sh`, `python`, `java`, `kotlin`, `javascript`, `ruby`, `go`

```yaml
agents:
  - name: node-agent
    install_command:
      bash: "npm install -g my-agent"
    version_command:
      shell: "my-agent --version"
    entry_command:
      javascript: "node /usr/local/bin/my-agent"

  - name: python-agent
    install_command:
      python: "import subprocess; subprocess.run(['pip', 'install', 'my-agent'])"
    version_command:
      bash: "python -c 'import my_agent; print(my_agent.__version__)'"
```

#### 3. Full Script Specification

For complex scripts with multiple files, custom interpreters, working directories, and environment variables:

```yaml
agents:
  - name: complex-agent
    install_command:
      type: bash
      files:
        - name: install.sh
          path: resources/install.sh
        - name: config.json
          content: |
            {
              "version": "1.0.0",
              "features": ["llm", "code"]
            }
      working_dir: /tmp/install
      env_vars:
        INSTALL_DIR: "/usr/local/bin"
        DEBUG: "true"

    version_command:
      type: python
      files:
        - name: version.py
          content: |
            import json
            with open('/etc/agent/version.json') as f:
                print(json.load(f)['version'])
      interpreter: /usr/bin/python3.11

    entry_command:
      type: bash
      files:
        - name: run.sh
          path: resources/run.sh
      entry_point: run.sh
      args: ["--mode", "interactive"]
      env_vars:
        AGENT_HOME: "/opt/agent"
```

### Agent Environment Variables

Environment variables can be configured at multiple levels with prediction-level overriding agent-level:

#### Agent-Level Environment Variables

Shared across all predictions using this agent:

```yaml
agents:
  - name: claude-code
    version: "1.0"
    env:
      MODEL: "claude-sonnet-4.5"
      TEMPERATURE: "0.7"
      MAX_TOKENS: "4096"
      ANTHROPIC_API_KEY: "${ANTHROPIC_API_KEY}"
```

#### Prediction-Level Environment Variables

Override or extend agent environment variables for specific predictions:

```yaml
agents:
  - name: claude-code
    version: "1.0"
    env:
      MODEL: "claude-sonnet-4.5"
      TEMPERATURE: "0.7"
      ANTHROPIC_API_KEY: "${ANTHROPIC_API_KEY}"

predictions:
  - id: high-temp-prediction
    name: "High Temperature Prediction"
    source: agent
    agent: claude-code
    env:
      # Overrides agent-level TEMPERATURE
      TEMPERATURE: "1.0"
      # Adds new variable
      CONTEXT_WINDOW: "200k"
    output:
      path: predictions/high-temp.json

  - id: low-temp-prediction
    name: "Low Temperature Prediction"
    source: agent
    agent: claude-code
    env:
      # Overrides agent-level TEMPERATURE
      TEMPERATURE: "0.0"
      # Different model for this prediction
      MODEL: "claude-opus-4.5"
    output:
      path: predictions/low-temp.json
```

#### Environment Variable Merge Order

Prediction-level environment variables override agent-level:

1. **Agent-level** (`agents[].env`): Base environment variables
2. **Prediction-level** (`predictions[].env`): Override and extend
3. **CLI arguments**: Final override (e.g., `--agent-opts`)

**Example Merge**:

```yaml
agents:
  - name: my-agent
    env:
      VAR_A: "from-agent"
      VAR_B: "from-agent"
      VAR_C: "from-agent"

predictions:
  - id: pred-1
    source: agent
    agent: my-agent
    env:
      VAR_B: "from-prediction"  # Overrides agent VAR_B
      VAR_D: "new-var"          # Adds new variable

# Final merged environment for pred-1:
# VAR_A: "from-agent"        (from agent)
# VAR_B: "from-prediction"   (overridden by prediction)
# VAR_C: "from-agent"        (from agent)
# VAR_D: "new-var"           (added by prediction)
```

### Complete Agent Configuration Example

```yaml
agents:
  - name: claude-code
    version: "1.0"
    description: "Claude Code AI agent"

    # Installation script
    install_command:
      bash: |
        npm install -g @anthropic-ai/claude-code
        claude-code configure --profile default

    # Version check script
    version_command:
      shell: "claude-code --version"

    # Entry command for running agent
    entry_command:
      type: bash
      files:
        - name: run-agent.sh
          content: |
            #!/bin/bash
            set -e
            echo "Starting Claude Code agent..."
            claude-code "$@"
      args: ["--interactive", "--verbose"]

    # Agent-level environment variables
    env:
      ANTHROPIC_API_KEY: "${ANTHROPIC_API_KEY:?API key required}"
      MODEL: "claude-sonnet-4.5"
      TEMPERATURE: "0.7"
      MAX_TOKENS: "8192"

    # Agent options
    opts:
      timeout: 1800
      retries: 3

    # MCP tools configuration
    features:
      mcp:
        tools_path: "${MCP_CONFIG_PATH:-/etc/claude/mcp-tools.json}"

  - name: gemini
    version: "2.0"
    entry_command: "gemini-agent --mode code"
    env:
      GOOGLE_API_KEY: "${GOOGLE_API_KEY}"
      MODEL: "gemini-2.0-flash-exp"

predictions:
  # Prediction 1: Claude with default temperature
  - id: claude-default
    name: "Claude Code - Default Settings"
    source: agent
    agent: claude-code
    output:
      path: predictions/claude-default.json

  # Prediction 2: Claude with high temperature (creative mode)
  - id: claude-creative
    name: "Claude Code - Creative Mode"
    source: agent
    agent: claude-code
    env:
      TEMPERATURE: "1.0"
      MODEL: "claude-opus-4.5"  # Override to use Opus
    output:
      path: predictions/claude-creative.json

  # Prediction 3: Claude with low temperature (precise mode)
  - id: claude-precise
    name: "Claude Code - Precise Mode"
    source: agent
    agent: claude-code
    env:
      TEMPERATURE: "0.0"
      MAX_TOKENS: "16384"  # Larger output for complex tasks
    output:
      path: predictions/claude-precise.json

  # Prediction 4: Gemini agent
  - id: gemini-default
    name: "Gemini - Default Settings"
    source: agent
    agent: gemini
    output:
      path: predictions/gemini-default.json
```

### Agent Script Examples by Use Case

#### Installing from Package Manager

```yaml
agents:
  - name: node-cli-tool
    install_command:
      bash: "npm install -g my-cli-tool@latest"
    version_command: "my-cli-tool --version"
```

#### Custom Installation with Dependencies

```yaml
agents:
  - name: custom-agent
    install_command:
      type: bash
      files:
        - name: install.sh
          content: |
            #!/bin/bash
            set -e

            # Install system dependencies
            apt-get update
            apt-get install -y python3 python3-pip git

            # Clone and install agent
            git clone https://github.com/org/agent.git /opt/agent
            cd /opt/agent
            pip3 install -r requirements.txt
            pip3 install -e .

            # Verify installation
            agent --version
      working_dir: /tmp
```

#### Multi-File Python Agent

```yaml
agents:
  - name: python-agent
    install_command:
      python: "import subprocess; subprocess.run(['pip', 'install', 'agent-package'])"

    version_command:
      type: python
      files:
        - name: version.py
          content: |
            import agent_package
            print(agent_package.__version__)

    entry_command:
      type: python
      files:
        - name: runner.py
          content: |
            import sys
            from agent_package import main
            sys.exit(main())
        - name: config.py
          content: |
            CONFIG = {
                "mode": "production",
                "verbose": True
            }
      interpreter: /usr/bin/python3.11
      env_vars:
        PYTHONPATH: "/opt/agent/lib"
```

#### Java Agent with Custom Classpath

```yaml
agents:
  - name: java-agent
    version_command:
      java: "java -cp /opt/agent/agent.jar com.example.Agent --version"

    entry_command:
      type: java
      files:
        - name: Agent.java
          path: resources/Agent.java
      interpreter: /usr/bin/java
      args: ["-cp", "/opt/agent/agent.jar", "com.example.Agent"]
      env_vars:
        JAVA_HOME: "/usr/lib/jvm/java-17"
        CLASSPATH: "/opt/agent/lib/*"
```

## Template Support

YAML configurations support template value substitution:

### Instance Property Templates

Reference dataset instance properties with fallback defaults:

```yaml
configurers:
  - name: jdk
    options:
      version: "{instance.jvm_version:24}"  # Use instance.jvm_version, default to "24"
```

### Environment Variable Templates

Substitute environment variables with various operators:

```yaml
# Simple substitution (empty if not set)
token: "${SNYK_TOKEN}"

# Default value if not set
model: "${LLM_MODEL:-claude-sonnet-3-5}"

# Require variable (fail if not set)
api_key: "${API_KEY:?API key is required}"

# Use value only if variable is set
debug_mode: "${DEBUG_MODE:+true}"
```

### CLI Argument Templates

Reference CLI arguments in templates:

```yaml
dataset:
  source:
    path: "{cli.dataset_path}"
```

## Configurer Environment Variables

Configurers support environment variable configuration at multiple levels:

### Configurer-Specific Environment Variables

```yaml
environment:
  configurers:
    - name: jdk
      options:
        version: "17"
      env:
        JAVA_TOOL_OPTIONS: "-Xmx2g -Xms512m"
        JDK_DEBUG: "false"
```

### Sandbox-Level Environment Variables (Shared)

```yaml
environment:
  sandbox:
    type: docker
    # Shared across all configurers
    env:
      PROJECT_NAME: "ee-bench"
      BUILD_ENV: "test"
    docker:
      # Docker-specific env vars
      env_vars:
        DOCKER_HOST: "unix:///var/run/docker.sock"
```

Environment variable merge order (later overrides earlier):
1. `sandbox.env` (shared)
2. `sandbox.docker.env_vars` or `sandbox.local.env_vars` (sandbox-specific)
3. `configurers[].env` (configurer-specific)

## Maintaining the Schema

When adding new configuration properties or options, you MUST update the schema. See **CLAUDE.md** for detailed schema maintenance rules.

### Quick Reference

1. **Adding a configurer:**
   - Add name to `ConfigurerConfiguration.properties.name.enum`
   - Add conditional schema (`allOf` → `if`/`then`) for options
   - Document all supported options

2. **Adding an evaluator:**
   - Add type to `EvaluatorConfiguration.properties.type.enum`
   - Add conditional schema for options
   - Document all supported options

3. **Verify changes:**
   ```bash
   ajv validate -s core/specs/benchmark-config-schema.json -d "core/specs/*.yaml"
   ```

## Example Configuration

See `dpaia-jvm.yaml` for a complete example demonstrating:

- Agent configuration with multiple agents
- Dataset filtering and sampling
- Environment configurers with options
- Configurer-specific environment variables
- Sandbox configuration (Docker)
- Evaluation pipeline with multiple evaluators
- Scoring configuration
- Output configuration
- Template value substitution

## Resources

- **Schema Specification**: `benchmark-config-schema.json`
- **Example Configuration**: `dpaia-jvm.yaml`
- **Feature Documentation**: `.claude/docs/yaml/`
- **Maintenance Guide**: `CLAUDE.md` (Configuration → YAML Configuration Schema section)
- **JSON Schema Documentation**: https://json-schema.org/
- **YAML Specification**: https://yaml.org/spec/
