---
name: gitlab-matrix-expressions
description: Use GitLab CI/CD Matrix Expressions to create dynamic 1:1 dependencies between parallel matrix jobs. This simplifies pipeline configuration by avoiding the manual listing of every job combination in `needs`.
---

# GitLab Matrix Expressions

Matrix expressions allow you to define dependencies between jobs that use `parallel:matrix` without manually listing every possible combination. This creates a "1:1 mapping" where a downstream job only waits for the specific upstream job that matches its matrix variables (e.g., `linux:test` waits only for `linux:build`, not `windows:build`).

## Syntax

The syntax uses the `matrix` context to reference the current job's matrix identifiers.

```yaml
'$[[ matrix.IDENTIFIER ]]'

```

> **Note:** The quotes `'` around the expression are required to ensure valid YAML parsing.

## Usage Patterns

### 1. Basic 1:1 Dependency

Use this pattern when you have two jobs (e.g., Build and Test) that run on the same matrix (e.g., multiple providers) and you want them to stay synchronized.

```yaml
linux:build:
  stage: build
  script: echo "Building..."
  parallel:
    matrix:
      - PROVIDER: [aws, gcp]

linux:test:
  stage: test
  script: echo "Testing..."
  parallel:
    matrix:
      - PROVIDER: [aws, gcp]
  needs:
    - job: linux:build
      parallel:
        matrix:
          # This maps the current job's PROVIDER to the upstream job's PROVIDER
          - PROVIDER: ['$[[ matrix.PROVIDER ]]']

```

**Result:** `linux:test [aws]` only waits for `linux:build [aws]`.

### 2. Using YAML Anchors (DRY Principle)

For complex matrices, define the matrix once in a hidden job or anchor to avoid repetition and errors.

```yaml
.build_matrix: &build_matrix
  parallel:
    matrix:
      - OS: ["ubuntu", "alpine"]
        ARCH: ["amd64", "arm64"]

compile_binary:
  <<: *build_matrix
  stage: compile
  script: echo "Compiling..."

integration_test:
  <<: *build_matrix
  stage: test
  needs:
    - job: compile_binary
      parallel:
        matrix:
          - OS: ['$[[ matrix.OS ]]']
            ARCH: ['$[[ matrix.ARCH ]]']

```

### 3. Subset Dependencies

You can mix matrix expressions with static values to depend on a specific subset of upstream jobs.

```yaml
needs:
  - job: prepare_env
    parallel:
      matrix:
        # Match the platform dynamically
        - PLATFORM: ['$[[ matrix.PLATFORM ]]']
          # But only depend on the specific Node version '18'
          VERSION: ["18"] 

```

## Limitations

* **Compile-time only:** Identifiers are resolved when the pipeline is created. You cannot use runtime variables.
* **String replacement only:** You cannot perform arithmetic or complex logic inside `[[ ... ]]`.
* **Scope:** You can only reference identifiers defined in the *current* job's `parallel:matrix` section.
