# GitHub ARC + Terminal-Bench Setup

This is a working setup guide for proving Terminal-Bench execution through GitHub Actions Runner Controller (ARC) on Kubernetes.

Step 1 installs only the ARC controller. Runner scale sets and Terminal-Bench come later.

## Step 1: Install ARC Controller

### Confirm Target Cluster

Use the Kubernetes context you want ARC installed into:

```bash
kubectl config current-context
kubectl get nodes -o wide
```

### Pick Chart Version

As of 2026-06-01, the latest GHCR chart version for both `gha-runner-scale-set-controller` and `gha-runner-scale-set` is `0.14.2`.

```bash
helm show chart \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

Set the install values:

```bash
export ARC_NAMESPACE="github-arc"
export ARC_RELEASE_NAME="arc"
export ARC_CHART_VERSION="0.14.2"
```

### Install Controller

```bash
helm upgrade --install "${ARC_RELEASE_NAME}" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version "${ARC_CHART_VERSION}" \
  --namespace "${ARC_NAMESPACE}" \
  --create-namespace \
  --wait \
  --timeout 5m
```

### Verify

```bash
helm list -n "${ARC_NAMESPACE}"
kubectl get pods -n "${ARC_NAMESPACE}" -o wide
kubectl get crds | grep actions.github.com
```

```bash
kubectl logs -n "${ARC_NAMESPACE}" "deploy/${ARC_RELEASE_NAME}-gha-rs-controller" --tail=100
```

## Step 2: Install Repo-Scoped Runner Scale Set

This creates a GitHub Actions runner scale set for one repository:

```text
GitHub repo -> ARC listener -> ephemeral runner pod
```

Terminal-Bench is not added yet. This step only registers a runner pool that GitHub Actions can target with `runs-on`.

### Create GitHub App

Create a GitHub App owned by the organization that owns the target repository.

Use this homepage URL:

```text
https://github.com/actions/actions-runner-controller
```

Disable webhooks:

```text
Webhook: inactive
Webhook URL: leave empty
```

Use these permissions:

```text
Repository permissions:
  Administration: Read and write
  Metadata: Read-only

Organization permissions:
  Self-hosted runners: Read and write
```

Generate a private key:

```text
GitHub App -> General -> Private keys -> Generate a private key
```

Save the downloaded `.pem` file and set `GITHUB_APP_PRIVATE_KEY_PATH` to its local path.

Install the app on the target organization or repository.

Get the installation ID from the installation URL:

```text
https://github.com/organizations/<org>/settings/installations/<installation-id>
```

Or get it from the GitHub API:

```bash
gh api "/repos/<owner>/<repo>/installation" --jq .id
```

Copy the numeric app ID, numeric installation ID, and private key path into the variables above.

Use the App ID, not the Client ID:

```text
Correct: App ID is numeric, for example 1234567
Wrong: Client ID starts with Iv, for example Iv23...
```

Validate the IDs before creating the secret:

```bash
case "${GITHUB_APP_ID}" in (*[!0-9]*|"") echo "GITHUB_APP_ID must be numeric. Do not use Client ID." && false;; esac
case "${GITHUB_APP_INSTALLATION_ID}" in (*[!0-9]*|"") echo "GITHUB_APP_INSTALLATION_ID must be numeric." && false;; esac
```

### Set Values

Use a runner scale set name that will later be used in workflow `runs-on`.

```bash
export ARC_NAMESPACE="github-arc"
export ARC_CHART_VERSION="0.14.2"
export RUNNER_SET_NAME="arc-terminal-bench"
export GITHUB_CONFIG_URL="https://github.com/<owner>/<repo>"
export GITHUB_SECRET_NAME="github-arc-app"
export GITHUB_APP_ID="<numeric-app-id>"
export GITHUB_APP_INSTALLATION_ID="<installation-id>"
export GITHUB_APP_PRIVATE_KEY_PATH="./private-key.pem"
```

### Create GitHub App Secret

```bash
kubectl create secret generic "${GITHUB_SECRET_NAME}" \
  --namespace "${ARC_NAMESPACE}" \
  --from-literal=github_app_id="${GITHUB_APP_ID}" \
  --from-literal=github_app_installation_id="${GITHUB_APP_INSTALLATION_ID}" \
  --from-file=github_app_private_key="${GITHUB_APP_PRIVATE_KEY_PATH}" \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

### Install Runner Scale Set

Terminal-Bench runs task containers and verifiers through Docker, so install the runner scale set with Docker-in-Docker enabled.

```bash
helm upgrade --install "${RUNNER_SET_NAME}" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version "${ARC_CHART_VERSION}" \
  --namespace "${ARC_NAMESPACE}" \
  --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
  --set githubConfigSecret="${GITHUB_SECRET_NAME}" \
  --set containerMode.type=dind \
  --set minRunners=0 \
  --set maxRunners=1 \
  --wait \
  --timeout 10m
```

### Verify

```bash
helm list -n "${ARC_NAMESPACE}"
kubectl get autoscalingrunnersets.actions.github.com -n "${ARC_NAMESPACE}"
kubectl get autoscalinglisteners.actions.github.com -n "${ARC_NAMESPACE}"
kubectl get pods -n "${ARC_NAMESPACE}" -o wide
```

Expected:

```text
arc-gha-rs-controller-...          1/1   Running
arc-terminal-bench-...-listener    1/1   Running
```

Check listener logs:

```bash
kubectl logs -n "${ARC_NAMESPACE}" -l "app.kubernetes.io/instance=${RUNNER_SET_NAME}" --tail=100
```

### Confirm GitHub Sees The Runner Scale Set

Open the repository in GitHub:

```text
Settings -> Actions -> Runners
```

The runner scale set should appear as `arc-terminal-bench`. With `minRunners=0`, there may be no idle runner pod until a workflow job is queued.

### Uninstall If Needed

```bash
helm uninstall "${RUNNER_SET_NAME}" -n "${ARC_NAMESPACE}"
kubectl delete secret "${GITHUB_SECRET_NAME}" -n "${ARC_NAMESPACE}"
```

## Step 3: Run One Real Terminal-Bench Task

This runs one real Terminal-Bench task through the local kimchi agent adapter. The repo script builds the current kimchi binary, Harbor runs the agent, and Terminal-Bench runs the verifier. The workflow prints `result.json`; the `reward` field is produced by the verifier.

### Add Kimchi API Key Secret

Create a repository Actions secret:

```text
Settings -> Secrets and variables -> Actions -> New repository secret
Name: KIMCHI_API_KEY
```

### Add Workflow

Create the full workflow:

```bash
mkdir -p .github/workflows

cat > .github/workflows/arc-terminal-bench-poc.yml <<'YAML'
name: Terminal-Bench ARC PoC

on:
  workflow_dispatch:
    inputs:
      task_id:
        description: Terminal-Bench task id
        type: string
        required: true
        default: terminal-bench/fix-git
      model:
        description: Kimchi model
        type: string
        required: true
        default: kimchi-dev/kimi-k2.6

permissions:
  contents: read

concurrency:
  group: arc-terminal-bench-poc
  cancel-in-progress: false

jobs:
  terminal-bench:
    runs-on: arc-terminal-bench
    timeout-minutes: 60
    env:
      KIMCHI_API_KEY: ${{ secrets.KIMCHI_API_KEY }}
      MODEL: ${{ inputs.model }}
      KIMCHI_BINARY_TARGET: linux-x64-baseline
      FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install system build dependencies
        run: |
          set -euo pipefail
          if command -v apt-get >/dev/null 2>&1; then
            export DEBIAN_FRONTEND=noninteractive
            if command -v sudo >/dev/null 2>&1; then
              sudo apt-get update
              sudo apt-get install -y build-essential ca-certificates curl git python3
            else
              apt-get update
              apt-get install -y build-essential ca-certificates curl git python3
            fi
          else
            echo "Unsupported runner image: apt-get not found" >&2
            exit 1
          fi

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Enable corepack
        run: corepack enable

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version-file: tools/proxy-helper/go.mod
          cache-dependency-path: tools/proxy-helper/go.sum

      - name: Setup uv
        uses: astral-sh/setup-uv@v8.1.0

      - name: Install pnpm dependencies
        run: pnpm install --frozen-lockfile
        env:
          PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD: "1"

      - name: Build proxy helper
        run: make -C tools/proxy-helper build
        env:
          GOOS: linux
          GOARCH: amd64

      - name: Install Docker Compose
        run: |
          set -euo pipefail
          case "$(uname -m)" in
            x86_64) compose_arch=x86_64 ;;
            aarch64|arm64) compose_arch=aarch64 ;;
            *) echo "Unsupported architecture: $(uname -m)" >&2; exit 1 ;;
          esac
          compose_version=v5.1.4
          compose_name="docker-compose-linux-${compose_arch}"
          tmp_dir="$(mktemp -d)"
          curl -fsSL "https://github.com/docker/compose/releases/download/${compose_version}/${compose_name}" -o "${tmp_dir}/${compose_name}"
          curl -fsSL "https://github.com/docker/compose/releases/download/${compose_version}/${compose_name}.sha256" -o "${tmp_dir}/${compose_name}.sha256"
          (cd "${tmp_dir}" && sha256sum -c "${compose_name}.sha256")
          plugin_dir="${DOCKER_CONFIG:-$HOME/.docker}/cli-plugins"
          mkdir -p "${plugin_dir}"
          install -m 0755 "${tmp_dir}/${compose_name}" "${plugin_dir}/docker-compose"

      - name: Verify Docker
        run: |
          set -euo pipefail
          docker version
          docker info
          docker compose version

      - name: Run one Terminal-Bench task with kimchi
        run: |
          set -euo pipefail
          echo "MODEL=${MODEL}"
          benchmark/terminal-bench-2/scripts/run-local.sh -i "${{ inputs.task_id }}" -k 1 -n 1

      - name: Show Terminal-Bench result summary
        if: always()
        run: |
          set -euo pipefail
          find benchmark/terminal-bench-2/jobs -name result.json -print -exec cat {} \; || true

      - name: Upload Terminal-Bench jobs
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: terminal-bench-jobs
          path: benchmark/terminal-bench-2/jobs
          if-no-files-found: ignore
YAML
```

Commit and push it:

```bash
git add .github/workflows/arc-terminal-bench-poc.yml
git commit -m "Add ARC Terminal-Bench PoC workflow"
git push
```

### Trigger The Workflow

Set the target repository:

```bash
export GITHUB_REPOSITORY="<owner>/<repo>"
export GITHUB_BRANCH="$(git branch --show-current)"
```

Run the workflow:

```bash
gh workflow run arc-terminal-bench-poc.yml --repo "${GITHUB_REPOSITORY}" --ref "${GITHUB_BRANCH}"
```

Watch the run:

```bash
gh run list --repo "${GITHUB_REPOSITORY}" --workflow arc-terminal-bench-poc.yml --limit 1
```

### Verify Runner Pod

While the workflow is queued or running:

```bash
kubectl get pods -n "${ARC_NAMESPACE}" -l "app.kubernetes.io/instance=${RUNNER_SET_NAME}" -o wide
kubectl get ephemeralrunners -n "${ARC_NAMESPACE}"
```

The listener pod should stay running. The ephemeral runner pod appears while the workflow job is queued or running and is deleted after the job finishes.

Check the latest workflow result:

```bash
gh run list --repo "${GITHUB_REPOSITORY}" --workflow arc-terminal-bench-poc.yml --limit 1
```
