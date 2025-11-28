2.  Add the following secrets with the value `true`:
    *   `ACTIONS_RUNNER_DEBUG`: Enables diagnostic logs for the runner itself (useful for self-hosted runner issues).
    *   `ACTIONS_STEP_DEBUG`: Enables debug logs for every step. This is the **most useful** one. It shows inputs, outputs, and expanded command execution.

### Reading Logs Effectively
*   **Expand Groups**: GHA collapses log groups by default. Look for the `>` arrow to expand details.
*   **Timestamps**: Toggle "Show timestamps" in the log viewer to identify slow steps.
*   **Search**: Use `Ctrl+F` (Cmd+F) directly in the browser log view to find error codes or keywords.

---

## 2. Interactive Debugging (SSH)

Sometimes logs aren't enough. You need to poke around the filesystem.

### Using `mxschmitt/action-tmate`
This action creates an SSH session into the runner.

**Usage:**
Add this step **before** the failing step (to inspect state) or **after** (on failure).

```yaml
steps:
  - uses: actions/checkout@v4
  
  - name: Setup Debug Session
    uses: mxschmitt/action-tmate@v3
    if: failure() # Only run if the previous steps failed
```

**How it works:**
1.  When the job runs, it will pause at this step.
2.  Check the "Console" output in GHA. It will print an SSH command (e.g., `ssh session@tmate.io ...`).
3.  Run that command in your local terminal.
4.  You are now inside the runner! You can run commands, check files, and debug environment variables.

> [!CAUTION]
> **SECURITY WARNING**: Never use this on public repositories or workflows that have access to production secrets. Anyone can see the logs and potentially SSH in.

---

## 3. Local Testing & Validation

Reduce the "commit-push-wait" loop by testing locally.

### `act` (Run GHA Locally)
`act` is a command-line tool that runs your workflows in Docker containers locally.

*   **Install**: `brew install act` (macOS) or download binary.
*   **Run**: `act -j build` (Runs the 'build' job).
*   **Simulate Events**: `act pull_request`.
*   **Secrets**: `act -s MY_SECRET=somevalue`.

*Note: `act` is an approximation. It uses Docker, so it won't perfectly match GitHub-hosted runners (especially file permissions or pre-installed tools), but it catches 80% of syntax and logic errors.*

### VS Code Extensions
Install the **GitHub Actions** extension by GitHub.
*   Provides syntax highlighting.
*   Validates YAML structure.
*   Auto-completes fields.

---

## 4. Debugging Logic & Contexts

Unsure why an `if` condition failed or a value is empty? Dump the context.

### Dump Contexts
Create a temporary step to print the JSON structure of GHA contexts.

```yaml
steps:
  - name: Dump GitHub Context
    env:
      GITHUB_CONTEXT: ${{ toJson(github) }}
    run: echo "$GITHUB_CONTEXT"
    
  - name: Dump Inputs
    env:
      INPUTS_CONTEXT: ${{ toJson(inputs) }}
    run: echo "$INPUTS_CONTEXT"
```

> [!WARNING]
> **Secrets Masking**: `toJson(secrets)` will print `***` for masked values, but be careful not to accidentally expose unmasked parts or indirect references.

---

## 5. Common Issues & Fixes

### A. "Command not found" / "File not found"
*   **Cause**: You are in the wrong directory. GHA checks out code to `$GITHUB_WORKSPACE`.
*   **Fix**:
    *   Check `pwd` and `ls -R` in a debug step.
    *   Use `working-directory` keyword if your script is in a subdirectory.
    ```yaml
    - run: ./script.sh
      working-directory: ./scripts
    ```

### B. "Permission denied" (Scripts)
*   **Cause**: Linux/Mac runners respect file permissions.
*   **Fix**: `chmod +x my-script.sh` before running it.

### C. Secrets are Empty
*   **Cause**:
    *   Secret name mismatch (case-sensitive).
    *   Secret not added to the specific **Environment** the job is targeting.
    *   PRs from forks do **not** have access to secrets (security feature).

### D. 403 Forbidden (GITHUB_TOKEN)
*   **Cause**: The default GITHUB_TOKEN has read-only permissions (in modern repos).
*   **Fix**: Explicitly grant permissions at the top of the workflow.
    ```yaml
    - run: env | grep -i proxy
    ```

### B. Self-Hosted Runner Diagnostics
If a self-hosted runner is crashing or behaving oddly:
1.  **Check Service Logs**: On the VM, check `journalctl -u actions.runner.*` or the `_diag` folder in the runner directory.
2.  **Disk Space**: Runners often run out of disk space.
    ```yaml
    - run: df -h
    ```

### C. API Rate Limiting (403/429)
GitHub API has rate limits (1,000 requests/hour per repo for GITHUB_TOKEN).
*   **Symptom**: 403 or 429 errors on API calls.
*   **Fix**: Use a GitHub App token (higher limits) or reduce API calls.

### D. Artifact Inspection
If a build fails but logs are unclear, upload the workspace as an artifact to inspect it locally.
```yaml
- uses: actions/upload-artifact@v4
  if: failure()
  with:
    name: debug-workspace
    path: .
```

---

## 7. Deep Dive: Local Testing with `act`

`act` is powerful but requires configuration to match your real GHA environment.

### Installation
*   **macOS**: `brew install act`
*   **Windows**: `choco install act-cli`
*   **Linux**: `curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash`

### Running Workflows
*   **Default (Push event)**: `act`
*   **Specific Job**: `act -j build`
*   **Specific Event**: `act pull_request`
*   **Dry Run (List jobs)**: `act -l`
*   **Graph View**: `act -g`

### Secrets Management
Do not pass secrets via command line flags if possible (history risk).
1.  Create a file `.secrets` (add to `.gitignore`!).
    ```properties
    MY_TOKEN=12345
    DB_PASSWORD=secret
    ```
2.  Run `act --secret-file .secrets`

### Runner Images (The "Micro" vs "Large" Dilemma)
`act` uses Docker images to simulate runners.
*   **Micro (Default)**: `node:16-buster-slim`. Very small (<200MB). Missing many tools (curl, git, jq might be missing).
*   **Medium**: `catthehacker/ubuntu:act-latest`. ~2GB. Has most common tools.
*   **Large**: `catthehacker/ubuntu:full-latest`. ~20GB+. Matches GitHub runners closely.

**Configuration (`~/.actrc`):**
To use the medium image by default:
```bash
-P ubuntu-latest=catthehacker/ubuntu:act-latest
```

### Limitations
*   **OIDC**: Does not support OpenID Connect (AWS/Azure keyless auth).
*   **Caching**: `actions/cache` does not work locally (it tries to hit GitHub API).
*   **Services**: Service containers (Postgres/Redis sidecars) work, but networking can be tricky on Docker Desktop.

---

## 8. Best Practices for Troubleshooting

1.  **Fail Fast**: By default, if one job in a matrix fails, they all cancel. To debug parallel issues, set:
    ```yaml
    strategy:
      fail-fast: false
    ```
2.  **Isolate**: If a workflow is complex, create a minimal reproduction workflow with just the failing step.
3.  **Check Status**: Check [githubstatus.com](https://www.githubstatus.com/) to see if Actions are having an outage.
