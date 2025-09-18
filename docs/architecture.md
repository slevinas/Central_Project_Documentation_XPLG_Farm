##### Nav


- [Ansible playbooks patterns](#ansible-passing-input-parameters-yamljsoncli-and-chaining-playbooks)



----------
# Ansible: passing input parameters (YAML/JSON/CLI) and chaining playbooks

This guide shows:

1) Ways to pass parameters into a playbook (inline, CLI, YAML/JSON files, prompts).  
2) Practical patterns to pass **results from one playbook (or play)** to another at runtime.

---

## 1) Defining inputs for a playbook

### A. Inline `vars` in the playbook
```yaml
# playbook.yml
- name: Example playbook
  hosts: all
  vars:
    app_port: 8080
    db_user: admin
  tasks:
    - name: Show params
      ansible.builtin.debug:
        msg: "App on port {{ app_port }} with user {{ db_user }}"
```

### B. Command‑line overrides with `--extra-vars`
```bash
ansible-playbook playbook.yml \
  --extra-vars "app_port=9090 db_user=haim"
```

### C. Load parameters from a **YAML** file
```yaml
# vars.yml
app_port: 8080
db_user: admin
db_pass: secret
```
```bash
ansible-playbook playbook.yml -e @vars.yml
```

### D. Load parameters from a **JSON** file
```json
// vars.json
{ "app_port": 8080, "db_user": "admin", "db_pass": "secret" }
```
```bash
ansible-playbook playbook.yml -e @vars.json
```

### E. Use `vars_files` inside a playbook
```yaml
- name: Playbook with vars file
  hosts: all
  vars_files:
    - vars.yml
  tasks:
    - ansible.builtin.debug:
        msg: "DB user is {{ db_user }}"
```

### F. Prompt for input at runtime (`vars_prompt`)
```yaml
- name: Prompt example
  hosts: localhost
  gather_facts: false
  vars_prompt:
    - name: db_pass
      prompt: "Enter DB password"
      private: yes
  tasks:
    - ansible.builtin.debug:
        msg: "Password length: {{ db_pass | length }}"
```

> **Tips**
> - Use files (`-e @file.yml`) for structured inputs and version control.  
> - Use CLI `--extra-vars` to quickly override.  
> - Use `vars_prompt` for interactive/sensitive values.

---

## 2) Passing results from one play (or playbook) to another

You have several patterns, from simplest (same playbook run) to cross‑playbook chaining.

### Pattern 1 — Same run, multiple plays: register → share → reuse
This is the most robust approach: orchestrate both steps in a **single parent playbook** composed of multiple plays, and share data between plays.

**How it works**  
1) In Play 1, compute a value and store it (via `set_fact` on a known host, or `set_stats`).  
2) In Play 2, read that value (from `hostvars`, or from aggregated stats) and use it.

**Example using `set_fact` on `localhost` + `hostvars`**
```yaml
# pipeline.yml
- name: Play 1 — compute build info
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Get app version (example)
      ansible.builtin.command: git describe --tags --always
      register: gitver
      args:
        chdir: "/path/to/repo"
    - name: Share version across plays (store on localhost)
      ansible.builtin.set_fact:
        app_version: "{{ gitver.stdout }}"

- name: Play 2 — deploy using version from Play 1
  hosts: webservers
  gather_facts: false
  tasks:
    - name: Use the value from localhost
      ansible.builtin.debug:
        msg: "Deploying version {{ hostvars['localhost'].app_version }} to {{ inventory_hostname }}"

    - name: Example use in a template
      ansible.builtin.template:
        src: app.env.j2
        dest: /opt/app/.env
      vars:
        VERSION: "{{ hostvars['localhost'].app_version }}"
```

**Why this works**  
`set_fact` writes a host‑fact on `localhost`. Subsequent plays can read it via `hostvars['localhost'].app_version`.

> Alternative: `set_stats` (per‑run global data). If you prefer a single global place:
> ```yaml
> - name: Aggregate data for the whole run
>   ansible.builtin.set_stats:
>     data:
>       app_version: "{{ gitver.stdout }}"
> ```
> Later you can read it as `hostvars['localhost']['ansible_stats']['data']['app_version']`.


### Pattern 2 — Include roles/tasks and pass `vars` directly
If you break logic into **roles** or **task files**, you can call them with variables and chain the results without needing separate playbooks.

```yaml
# playbook.yml
- name: Orchestrate two steps with task includes
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Step 1 — run task file that produces a value
      ansible.builtin.include_tasks: tasks/discover.yml
      register: step1

    - name: Step 2 — run another task file, passing results
      ansible.builtin.include_tasks: tasks/deploy.yml
      vars:
        discovered_value: "{{ step1.results_value }}"
```

`tasks/discover.yml` can `set_fact: results_value=...` (or echo a value and the caller registers it).  
`tasks/deploy.yml` can then consume `discovered_value`.

> `include_role` also accepts `vars:` which makes roles a great unit for composable pipelines.


### Pattern 3 — Chain **separate playbooks** across runs (file handoff)
When you truly must run distinct playbooks one after another (e.g., by CI/CD jobs), hand data off via a file.

**Step A — First playbook writes a vars file**
```yaml
# first.yml
- name: Discover values and export to a vars file
  hosts: localhost
  gather_facts: false
  vars:
    export_path: /tmp/next-vars.json
  tasks:
    - name: Compute value (example)
      ansible.builtin.command: date +%s
      register: epoch

    - name: Export as JSON for the next playbook
      ansible.builtin.copy:
        dest: "{{ export_path }}"
        content: "{{ {'build_id': epoch.stdout, 'env': 'staging'} | to_nice_json }}"
      delegate_to: localhost
      run_once: true
```

**Step B — Second playbook consumes that file**
```yaml
# second.yml
- name: Use values from previous run
  hosts: webservers
  gather_facts: false
  vars_files:
    - /tmp/next-vars.json
  tasks:
    - ansible.builtin.debug:
        msg: "Using build {{ build_id }} on {{ env }} for {{ inventory_hostname }}"
```

**Shell (or CI) pipeline**
```bash
ansible-playbook first.yml
ansible-playbook second.yml -e @/tmp/next-vars.json
```

> Variations:
> - Write **YAML** instead of JSON (use `to_nice_yaml`).
> - Put the file in your workspace so your CI job artifacts pass it to the next job.


### Pattern 4 — Persist across runs with **fact caching** (advanced)
If you enable fact caching (e.g., Redis/JSON cache), you can `set_fact: cacheable: true` in Playbook 1 and read it in Playbook 2 without writing files.

```yaml
- name: Set cacheable facts
  hosts: localhost
  gather_facts: false
  tasks:
    - ansible.builtin.set_fact:
        build_id: "12345"
      cacheable: true
```
Then, in a later run, that fact can be available (subject to your cache config/TTL). Requires inventory hostnames to match and controller configured with `fact_caching`.

---

## 3) Putting it all together — a compact example

### Files
```
vars.yml
first.yml
second.yml
pipeline.yml
```

**vars.yml**
```yaml
app_port: 8080
```

**pipeline.yml** (single run, two plays)
```yaml
- name: Build
  hosts: localhost
  gather_facts: false
  tasks:
    - ansible.builtin.command: echo v1.2.3
      register: ver
    - ansible.builtin.set_fact:
        app_version: "{{ ver.stdout }}"

- name: Deploy
  hosts: webservers
  vars_files: [vars.yml]
  tasks:
    - ansible.builtin.debug:
        msg: "Deploy {{ hostvars['localhost'].app_version }} to port {{ app_port }} on {{ inventory_hostname }}"
```

**Run**
```bash
ansible-playbook pipeline.yml -i inventory.ini -e @vars.yml
```

**Separate playbooks** (two runs + file handoff)
```bash
ansible-playbook first.yml
ansible-playbook second.yml -e @/tmp/next-vars.json
```

---

## 4) Quick reference
- `-e key=val` / `-e @file.yml|json` — pass parameters from CLI/files.  
- `vars`, `vars_files`, `vars_prompt` — define inputs inside playbook.  
- `set_fact` + `hostvars` — share values across plays in the same run.  
- `include_tasks` / `include_role vars:` — pass values when composing tasks/roles.  
- File handoff (`copy` + `to_nice_json|yaml`) — chain separate playbooks.  
- Fact caching (`cacheable: true`) — persist across runs without files (needs controller config).

---

### Gotchas
- `import_playbook` does **not** accept `vars:`. Prefer roles/task includes if you need to pass variables into included units.  
- Cross‑playbook variables don’t magically persist. Use a **file handoff** or **fact caching**.

---

**Need help adapting this to your XPLG pipelines (Jenkins/Ansible/Helm)?** Share your current playbook layout and I’ll tailor the chaining pattern + CI snippet for you.



---

## Monorepo Scaffold: `testing_services/flux_test_service`

Below is a proposed scaffold for your `flux_test_service` harness within `ODD-X8/src/testing_services/`, now including how the HTTP-wrapper (`AllureClient`) in `common/` and the service-level `uploader.py` interact as part of the call flow diagram.

```mermaid
flowchart LR
  subgraph Orchestrator Server (TestServiceServer)
    direction TB
    A[Install Docker & GitHub Runner]
    B[Pull XPLG Flux-Sync Image]
    C[Run XPLG Flux-Sync Container<br/>(Instance A)]
  end

  subgraph DockerTester VM (flux-agent)
    direction TB
    D[Install Docker]
    E[Install Python & pytest<br/>(install-python-env.sh)]
    F[Clone `flux_test_service` Repo]
    G[Execute `start-flux-tests.sh` script<br/>• Pulls & runs local XPLG Flux-Sync Container (Instance B)<br/>• Prepares environment ready for account initialization and testing]
    J[Initialize Flux Accounts via API<br/>• Create Source account<br/>• Create Target account]
    H[Run pytest suite<br/>(pytest --alluredir=allure-results)]
    I[Upload results to Allure<br/>(upload-to-allure.sh)]
  end

  A --> B --> C
  C --> G
  D --> E --> F --> G --> J --> H --> I
  I -->|POST results| ODD[ODD-Allure Service API]
```

**Note on Step G:** The `start-flux-tests.sh` script can either pull and run a local XPLG Flux-Sync container (Instance B) if you need an isolated instance for integration tests, or skip that step and directly target the already-running service on the Orchestrator (Instance A). Choose based on your testing isolation requirements.

### Directory Tree Scaffold (with actual filenames)

```
X8.ODD/x8.odd.storm/odd.test/
├── infra/
│   └── flux-test-ci/
│       ├── configure-git-runner.sh      # installer for self-hosted runner
│       └── setup-docker.sh              # Docker install script for orchestrator
│
└── src/
    ├── common/
    │   ├── __init__.py
    │   ├── ssh_helpers.py                # from utils/ssh/ssh_helpers.py
    │   ├── docker_utils.py               # from utils/general/docker_utils.py
    │   ├── allure_client.py              # HTTP wrapper for Allure uploads
    │   └── openapi_generated_client/     # xplg_client API client
    │       └── xplg_client/
    │           ├── api/
    │           ├── models/
    │           └── client.py
    │
    └── testing_services/
        └── flux_test_service/
            ├── src/flux_test_service/
            │   ├── __init__.py
            │   ├── runner.py              # orchestration logic
            │   ├── uploader.py            # uses AllureClient to post results
            │   ├── config.py              # renamed from xconfig.py
            │
            │   ├── api/                   # moved from utils/api
            │   │   ├── api_create_task.py
            │   │   └── api_delete_task.py
            │
            │   ├── flux/                  # moved from utils/flux
            │   │   ├── task_runner.py
            │   │   ├── measure.py
            │   │   └── csv_metrics.py
            │
            │   └── general/               # moved from utils/general
            │       └── record_to_csv.py
            │
            ├── tests/
            │   ├── conftest.py            # pytest fixtures
            │   ├── test_runner.py         # unit tests for runner
            │   └── test_uploader.py       # unit tests for uploader
            │
            ├── scripts/
            │   ├── install-python-env.sh  # venv & deps
            │   ├── start-flux-tests.sh    # pulls/runs XPLG image + init accounts
            │   └── upload-to-allure.sh    # POST allure-results/
            │
            ├── templates/
            │   ├── xml/                   # from TasksXMLs/
            │   │   ├── TaskBasicExampleXML.xml
            │   │   ├── my-pytest-task.xml
            │   │   └── SyncLogsConfiguration.xml
            │   └── jinja/                 # from tasks-templates/
            │       ├── tasks.xml.j2
            │       └── my-task.xml
            │
            └── assets/
                └── sync_metrics.csv       # moved from root
```

## Mapping Existing `XPLG-Flux-Data-Sync-Tests` into Monorepo

One-to-one mapping of files into the structure above.
into Monorepo
`XPLG-Flux-Data-Sync-Tests` into Monorepo

Here’s how your current repo would relocate into the above scaffold:

```
XPLG-Flux-Data-Sync-Tests/                   ODD-X8/src/testing_services/flux_test_service/
─────────────────────────                   ──────────────────────────────────────────────
README.md               →   README.md
scripts/                →   scripts/
  run_flux_tests.sh          start-flux-tests.sh
  run_flux_tests_v2.sh       start-flux-tests-v2.sh
utils/                  →   src/flux_test_service/
  ssh/ssh_helpers.py         common/ssh_helpers.py
  api/*.py                   src/flux_test_service/api/*.py
  flux/*.py                  src/flux_test_service/flux/*.py
openapi_generated_client/ →   src/common/openapi_generated_client/
TasksXMLs/              →   templates/xml/
tasks-templates/        →   templates/jinja/
sync_metrics.csv        →   assets/sync_metrics.csv
pytest.ini              →   pytest.ini
conftest.py             →   tests/conftest.py
requirements.txt        →   requirements-dev.txt
xconfig.py              →   src/flux_test_service/config.py
```

And the call flow from pytest results → uploader → AllureClient → ODD-Allure still applies within this folder mapping. Let me know if you’d like any more refinements!

## Detailed File Mapping & Move Commands

Assuming your original **XPLG-Flux-Data-Sync-Tests** code lives in `~/XPLG-Flux-Data-Sync-Tests` and you’ve cloned **X8.ODD** into `~/X8.ODD/x8.odd.storm/odd.test`, here’s a convenient set of commands to transfer and relocate files into the monorepo structure:

```bash
# 1. Copy your original repo contents into the ODD monorepo folder
cp -r ~/XPLG-Flux-Data-Sync-Tests/* ~/X8.ODD/x8.odd.storm/odd.test/

# 2. Change into the ODD test folder
cd ~/X8.ODD/x8.odd.storm/odd.test

# 3. Move XML templates into the service templates:
mkdir -p src/testing_services/flux_test_service/templates/xml
git mv TasksXMLs/* src/testing_services/flux_test_service/templates/xml/

# 4. Move Jinja task templates:
mkdir -p src/testing_services/flux_test_service/templates/jinja
git mv tasks-templates/* src/testing_services/flux_test_service/templates/jinja/

# 5. Relocate utility modules:
# 5a. SSH helpers
git mv utils/ssh/ssh_helpers.py src/common/ssh_helpers.py

# 5b. API helpers
mkdir -p src/testing_services/flux_test_service/src/flux_test_service/api
git mv utils/api/*.py src/testing_services/flux_test_service/src/flux_test_service/api/

# 5c. Flux utilities
mkdir -p src/testing_services/flux_test_service/src/flux_test_service/flux
git mv utils/flux/*.py src/testing_services/flux_test_service/src/flux_test_service/flux/

# 5d. General utilities
mkdir -p src/testing_services/flux_test_service/src/flux_test_service/general
git mv utils/general/*.py src/testing_services/flux_test_service/src/flux_test_service/general/

# 6. Move the OpenAPI-generated client into common:
mkdir -p src/common/openapi_generated_client
git mv openapi_generated_client/* src/common/openapi_generated_client/

# 7. Move pytest config and fixtures:
git mv conftest.py src/testing_services/flux_test_service/tests/conftest.py
git mv pytest.ini src/testing_services/flux_test_service/pytest.ini

# 8. Rename xconfig to service config:
git mv xconfig.py src/testing_services/flux_test_service/src/flux_test_service/config.py

# 9. Archive sync_metrics.csv into assets:
mkdir -p src/testing_services/flux_test_service/assets
git mv sync_metrics.csv src/testing_services/flux_test_service/assets/sync_metrics.csv

# 10. Move your existing scripts into the new scripts/ folder:
git mv scripts/* src/testing_services/flux_test_service/scripts/

# 11. Clean up now-empty legacy directories:
git rm -r TasksXMLs tasks-templates utils openapi_generated_client

# 12. Inspect and commit:
git status
# If everything looks correct:
git commit -m "chore: import flux-data-sync-tests into flux_test_service scaffold"
```

After running these commands from `odd.test`, your monorepo will contain all the original code mapped into the structured layout under `src/testing_services/flux_test_service`. Then push your feature branch and proceed with testing on the flux-agent VM.

##### ✅ What You've Built
You've correctly modularized the CI as follows:

📁 .github/workflows/orchestrator-ci.yml
The top-level workflow that triggers on push/pull-request (e.g. ci/**), and calls the reusable one.

yaml
Copy
Edit
name: Orchestrator CI

on:
  push:
    branches: [ci/**]  # Adjust as needed

jobs:
  call-flux-sync-tests:
    uses: ./.github/workflows/flux-test-ci.yml
    secrets:
      TESTER_SSH_KEY: ${{ secrets.TESTER_SSH_KEY }}
      TESTER_ENV_FILE_CONTENTS: ${{ secrets.TESTER_ENV_FILE_CONTENTS }}
    with:
      tester-host: ${{ secrets.TESTER_HOST }}
      tester-user: ${{ secrets.TESTER_USER }}
      allure-url:  ${{ secrets.ALLURE_URL }}
📁 .github/workflows/flux-test-ci.yml
This is the reusable workflow (must use workflow_call) and accepts the needed secrets/inputs:

yaml
Copy
Edit
name: Flux Sync CI

on:
  workflow_call:
    inputs:
      tester-host:
        required: true
        type: string
      tester-user:
        required: true
        type: string
      allure-url:
        required: true
        type: string
    secrets:
      TESTER_SSH_KEY:
        required: true
      TESTER_ENV_FILE_CONTENTS:
        required: true

jobs:
  flux-test:
    runs-on: [self-hosted, linux, orchestrator]
    env:
      GITHUB_REF: ${{ github.ref }}
      ALLURE_URL: ${{ inputs.allure-url }}
      TESTER_HOST: ${{ inputs.tester-host }}
      TESTER_USER: ${{ inputs.tester-user }}

    steps:
      - name: Check out code
        uses: actions/checkout@v3

      - name: Start ssh-agent and add tester key
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.TESTER_SSH_KEY }}

      - name: Run orchestration on tester VM
        working-directory: x8.odd.storm/odd.test
        run: |
          chmod +x scripts/orchestrate_flux_tests.sh
          scripts/orchestrate_flux_tests.sh
        env:
          ALLURE_URL: ${{ env.ALLURE_URL }}
          TESTER_HOST: ${{ env.TESTER_HOST }}
          TESTER_USER: ${{ env.TESTER_USER }}
          GITHUB_REF: ${{ env.GITHUB_REF }}
          TESTER_ENV_FILE_CONTENTS: ${{ secrets.TESTER_ENV_FILE_CONTENTS }}

      - name: Upload Allure raw-results artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-raw-results-${{ github.run_id }}
          path: raw-results/
🔐 GitHub Secrets You Must Define
Secret Name	Value
TESTER_SSH_KEY	private key for SSH to tester VM
TESTER_HOST	IP or hostname of your tester VM
TESTER_USER	user (e.g. xplg) on tester VM
ALLURE_URL	Allure server endpoint (e.g. http://10.0.0.138:5050)
TESTER_ENV_FILE_CONTENTS	Multi-line content of .env like you showed

Example TESTER_ENV_FILE_CONTENTS:

env
Copy
Edit
FLUX_DELETE_TASK_SEGMENT=/data-sync/task/delete
FLUX_CREATE_ACC_SEGMENT=/accounts
FLUX_ENABLE_ACC_SEGMENT=/accounts/enable
FLUX_REMOTE_SOURCE_DIR=/home/xplg/flux-source-dir
FLUX_REMOTE_TARGET_DIR=/home/xplg/flux-target-dir
FLUX_REMOTE_LOGS_DIR=/home/xplg/flux-logs-for-sync-tests
FLUX_CSV_METRICS_FILE_NAME=sync_metrics.csv
DOCKER_TESTER_URL=10.0.0.85
DOCKER_TESTER_USER=xplg
DOCKER_TESTER_PASS=arigato$
DOCKER_TESTER_PORT=22
DOCKER_TESTER_API_URL=http://10.0.0.85:30303/logeye/rest
✅ You Can Also Run It Standalone
To run the flux-test-ci.yml directly (without the caller), just use it as a regular workflow file — uncomment the on: push: and on: pull_request: lines like you already had before.

But for modular orchestration via orchestrator-ci.yml, that should stay removed.

Let me know if you'd like:

A dry-run test on a dummy branch

Or to fully switch over and delete the standalone logic

------------

🧾 Summary of What You've Achieved:
✅ Refactored flux-test-ci.yml into a reusable workflow

✅ Created orchestrator-ci.yml as a top-level trigger

✅ Passed GitHub Actions CI using secrets + inputs

✅ Verified .env templating, SSH key injection, and test orchestration works end-to-end

##### ✅ Current Architecture (as you described and as is working now):
You have:

- **Orchestrator VM ** (GitHub self-hosted runner) → runs the CI workflow

- **Tester VM (Docker-enabled)** → is the remote host where:

 - Docker is installed and containers run

 - Tests are executed via SSH

 - .env/configs are synced and logs/artifacts are fetched

------ 


##### ✅ Current Status
You are on branch: "ci/refactor-reusable-workflow"

You are successfully using:

1. flux-test-ci.yml – now modular via workflow_call, accepts secrets and parameters.

2. orchestrator-ci.yml – the main top-level workflow that:

- Triggers on pushes to ci/**

- Calls flux-test-ci.yml as a reusable job using inputs and secrets.

3. "orchestrate_flux_tests.sh" – runs on the orchestrator VM and SSHs into a remote Tester VM to:

 - Clone or update the repo

 - Set up Python and Docker

 - Run the XPLG containers

 - Execute pytest

 -  Push results to Allureorchestrate_flux_tests

4. Secrets and vars are injected correctly via GitHub Actions.
---- 
##### 🎯 Goal / Roadmap
Make your CI workflows clean, modular, scalable, and reusable for different projects or teams. The high-level structure will be:

1. Workflow Tiers (Separation of Concerns)
| Layer                    | Description            | Example                                                       |
| ------------------------ | ---------------------- | ------------------------------------------------------------- |
| 🧠 `orchestrator-ci.yml` | Top-level orchestrator | Defines overall CI flow, triggers others                      |
| ⚙️ `flux-test-ci.yml`    | Reusable job           | Accepts inputs/secrets and runs test orchestration            |
| 🧪 Shell scripts         | Logic runner           | Handles test logic on tester VM (`orchestrate_flux_tests.sh`) |


#### 🛣️ Suggested Roadmap (step-by-step)
Step 1: Modularize Shell Steps
✅ Already done: orchestrate_flux_tests.sh is invoked from GitHub CI

Next steps:

✅ env injection via TESTER_ENV_FILE_CONTENTS

➡️ Modularize shell logic:

Extract docker install into install_docker.sh

Extract clone/update logic into checkout_repo.sh

Extract pytest + allure into run_tests.sh

Step 2: Reusable CI Blocks
Split flux-test-ci.yml into logical reusable steps:

docker-setup.yml

orchestrate-tests.yml

(Optional) allure-upload.yml

Use workflow_call for reuse in other top-level workflows.

Step 3: SSH-less Architecture (Optional Future)
Investigate installing a GitHub runner on the Tester VM dynamically via orchestrator.

That eliminates SSH and allows direct GitHub job scheduling

Adds complexity in secure runner install & cleanup

Step 4: Advanced Matrix Support
Add matrix-based test definitions:

yaml
Copy
Edit
strategy:
  matrix:
    test_case: [
      test_sync_flow_create_execute_api.py,
      test_delete_api.py,
      test_retry_mechanism.py
    ]
Allow test distribution across multiple machines or serially on one.

🧩 What You Should Expect When It’s Done
You push to a branch

orchestrator-ci.yml triggers

It invokes flux-test-ci.yml (or multiple)

.env is templated from GitHub secrets

Orchestration is done fully on the remote machine

Results are uploaded automatically

You get a modular, readable, and reusable CI pipeline

-----

✅ Current State
You’ve already:

- Split CI logic into two GitHub workflows:

 - orchestrator-ci.yml – Top-level entrypoint triggered on push to ci/**.

 - flux-test-ci.yml – Reusable workflow with workflow_call to run tests on the remote Tester VM.

- Parameterized values like TESTER_HOST, TESTER_USER, TESTER_ENV_FILE_CONTENTS through with: and secrets:.

- Confirmed basic orchestration is working end-to-end, from orchestrator to tester.

🧭 Goal
Make your GitHub CI orchestration:

More maintainable and composable.

Easy to trigger from multiple sources (branches, dispatch, schedule).

Cleanly reusable for various test types or environments.


🛣️ Roadmap: CI Modulation Plan
1. Break out script phases into individual workflows
Split the orchestration into these reusable workflows:

bootstrap-orchestrator.yml – setup docker + self-host basics (local VM)

start-orchestrator-instance.yml – launch the XPLG orchestrator (optional)

render-env.yml – render and sync .env to the Tester VM

run-tests-on-tester.yml – SSH + run orchestration + collect results

push-allure-results.yml – publish test results to Allure

These will all be workflow_call compatible.

2. Create a shared composite action (optional)
If multiple jobs are doing similar SSH steps or .env rendering, wrap them as a reusable composite action under .github/actions/.

3. Refactor current shell scripts
Turn orchestrate_flux_tests.sh into smaller scripts:

setup_remote_env.sh

run_flux_tests.sh

collect_results.sh

And invoke them modularly from GH workflows.

4. Add matrix support
Enable matrix testing (e.g. test different datasets or flags in parallel).

5. Optional: Explore GitHub-hosted runners on remote Tester VM
Eventually, replace SSH with GitHub Actions runners on the Tester VM for better reliability and modularity.

Would you like to begin now by extracting one of the stages (e.g. Docker setup or .env handling) into a reusable workflow or composite action?


Great — we'll follow the priority roadmap and begin with Stage 1: Modularizing GitHub Actions for Reusability. Here's how we'll proceed:

✅ Current State
You’re on the ci/refactor-reusable-workflow branch and already have:

A working orchestrate_flux_tests.sh script

.env.template in use

flux-test-ci.yml workflow refactored to use workflow_call (for reuse)

🔄 Stage 1: Make CI Reusable
🔹 1. Extract the reusable logic into a reusable workflow
We'll:

Move the core CI logic from flux-test-ci.yml into .github/workflows/reusable-flux-test.yml

Define inputs: and secrets: blocks

Ensure this workflow can be called by other workflows (and still run standalone for now)

🔹 2. Make orchestrator-ci.yml the top-level entry point
Create .github/workflows/orchestrator-ci.yml

It will use workflow_call to trigger the reusable logic

Will define all necessary secrets + environment in the call


Summary of Current Plan and Roadmap
✅ Current State:
You have two VMs:

Orchestrator VM: Runs the GitHub self-hosted runner, triggers workflows.

Tester VM: Executes the XPLG containers and tests via SSH.

A working branch ci/refactor-reusable-workflow is active and stable.

🎯 Goal:
Modularize and clean up the CI for maintainability, reuse, and future scaling.

🛣️ Roadmap (Based on Your Prioritized Plan):
✅ Phase 1: Templatize & Parameterize via Secrets
.env.template populated dynamically using TESTER_ENV_FILE_CONTENTS.

Secrets (e.g., TESTER_HOST, TESTER_SSH_KEY) injected securely via GitHub Actions.

✅ Phase 2: Modularize into Reusable Workflow
Done: reusable-flux-test.yml (↑) created to expose inputs via workflow_call.

🔜 Phase 3: Call Reusable Workflow from Top-Level orchestrator-ci.yml
Will create a higher-level CI file that does nothing but call the reusable logic.

Allows different branches/projects to reuse the same orchestration logic.

🔜 Phase 4: Refactor Script Steps Further (Optional but Recommended)
Extract Docker setup into setup_docker.yml

Extract test runner call into run_flux_tests.yml

Each can be reused or debugged in isolation.

🔮 Future Improvements:
Parallelize multiple test sets (e.g., via matrix)

Dynamically spin up tester VMs (with installed runner or via SSH)

Slack/email alerts based on --failures or pytest exit code


| Topic                 | Current                                | Recommended Change        |
| --------------------- | -------------------------------------- | ------------------------- |
| `GITHUB_REF` handling | `refs/heads/${{ inputs.branch }}`      | `github.ref` (native var) |
| Reason                | Unnecessary override                   | Use GitHub-native context |
| Fix location          | `flux-test-ci.yml` (reusable workflow) | Same                      |


##### ✅ Current State
You now have a working 2-stage reusable CI pipeline with:

- Top-level workflow .github/workflows/orchestrator-v1-ci.yml

 - Declares with: and secrets: values

 - Triggers the test suite via uses: ./github/workflows/reusable-flux-tests-ci.yml

- Reusable job workflow .github/workflows/reusable-flux-tests-ci.yml

 - Receives inputs/secrets

 - Sets up Docker, runs orchestrator, then calls orchestrate_flux_tests.sh on the remote tester VM

 - Pulls back Allure results

And it passes end-to-end, as shown in your GitHub Actions view.

⏭️ Next Step (as per our agreed roadmap)
We now proceed to split orchestrate_flux_tests.sh into:

prepare_test_dir_and_sync_repo.sh

Ensures test dir exists

Clones or updates repo on remote Tester VM

run_flux_tests.sh

Sets up Python, Docker

Starts containers

Runs pytest

Fetches raw-results

🧩 Modularization Plan
We'll do this by:

Creating two new CI steps in reusable-flux-tests-ci.yml

Replacing the orchestrate_flux_tests.sh block

Ensuring the new scripts are invoked correctly over SSH (or inline)

✅ Confirmed Plan
We will now gradually break this script into discrete executable scripts, each mapped to a CI step:
| Step | Extracted Script Filename           | Purpose                                              |
| ---- | ----------------------------------- | ---------------------------------------------------- |
| 1    | `prepare_test_dir_and_sync_repo.sh` | Creates test dir, clones/updates repo, writes `.env` |
| 2    | `setup_python_env.sh`               | Sets up Python venv and dependencies                 |
| 3    | `setup_docker.sh`                   | Ensures Docker is installed (on Tester VM)           |
| 4    | `start_xplg_via_compose.sh`         | Starts containers using `docker-compose`             |
| 5    | `run_flux_tests.sh`                 | Runs `pytest` on a specific suite                    |
| 6    | `collect_test_results.sh`           | Copies raw-results from remote to CI                 |
| 7    | `push_allure_results.sh`            | Pushes test results to Allure                        |


We'll use SSH options and remote execution where needed — similar to your original code.

🔄 Immediate Next Step
We’ll begin with step 1:
Extract prepare_test_dir_and_sync_repo.sh — which will:

 - Set up SSH known hosts

 - Create the remote dir

 - Seed .env

 - Clone or pull the repo


 📌 Inputs Required (via environment)
Make sure these are passed as env vars in your CI job:

TESTER_USER

TESTER_HOST

TESTER_ENV_FILE_CONTENTS

GITHUB_REF → used to derive BRANCH

You’ll also need to define:

yaml
Copy
Edit
env:
  GITHUB_REF: ${{ github.ref }}
  TESTER_USER: ${{ secrets.TESTER_USER }}
  TESTER_HOST: ${{ secrets.TESTER_HOST }}
  TESTER_ENV_FILE_CONTENTS: ${{ secrets.TESTER_ENV_FILE_CONTENTS }}


---

✅ You’re in sync with the plan. Here's the updated roadmap status:

| Step | Task                                                              | Status                                       |
| ---- | ----------------------------------------------------------------- | -------------------------------------------- |
| 1️⃣  | Make current monolithic CI flow reusable via `workflow_call`      | ✅ DONE                                       |
| 2️⃣  | Extract SSH sync & .env seeding into script → run independently   | ✅ DONE (`prepare_test_dir_and_sync_repo.sh`) |
| 3️⃣  | Extract Python setup on remote                                    | ✅ DONE (`setup_python_env_on_tester_vm.sh`)  |
| 4️⃣  | **Next: Docker install and container startup split into scripts** | ⏳ NOW                                        |


✅ Add CI Step to Run This Script
Add the following step to your reusable-flux-tests-ci.yml, after the "Set up Python" step:

yaml
Copy
Edit
- name: Start Flux containers on Tester VM
  working-directory: x8.odd.storm/odd.test
  run: |
    ssh -o StrictHostKeyChecking=yes -i ~/.ssh/id_rsa "${TESTER_USER}@${TESTER_HOST}" <<'EOF'
      cd ~/X8.ODD/x8.odd.storm/odd.test
      chmod +x scripts/start_xplg_via_compose.sh
      ./scripts/start_xplg_via_compose.sh
    EOF
  env:
    TESTER_HOST: ${{ env.TESTER_HOST }}
    TESTER_USER: ${{ env.TESTER_USER }}
✅ This uses a remote SSH exec — just like you did in orchestrate_flux_tests.sh originally.



🔁 Recap of CI So Far
The current CI flow is now modular and likely looks like this:


| Step                   | Responsibility                       |
| ---------------------- | ------------------------------------ |
| ✅ `checkout`           | fetch repo                           |
| ✅ `ssh-agent`          | add SSH key                          |
| ✅ setup docker         | on orchestrator                      |
| ✅ start orchestrator   | on orchestrator                      |
| ✅ prepare & sync       | tester repo, `.env`, checkout        |
| ✅ setup python         | on tester                            |
| ⏳ **start containers** | via `docker compose up -d` on tester |
| 🔜 run pytest          | and fetch results                    |
| 🔜 push to Allure      | optional in future                   |


#### 🔄 What We Just Achieved


| Task                                    | Status |
| --------------------------------------- | ------ |
| Modularize CI to call reusable workflow | ✅ Done |
| Prepare and sync tester VM              | ✅ Done |
| Set up Python env on tester             | ✅ Done |
| Start Flux containers via SSH           | ✅ Done |



✅ Current State Recap
You're already running a fully modularized and passing CI flow with:

Extracted orchestration into discrete, logical steps.

run_flux_tests.sh doing the pytest run + Allure push.

All CI variables and secrets flowing cleanly via env: blocks and workflow_call.

The relevant CI steps, as confirmed, are:

Setup orchestrator (Docker)

Start orchestrator (XPLG container)

Prepare remote .env and sync repo

Setup Python on tester

Start Flux containers on Tester VM

Run sync tests and push results to Allure

The previously monolithic orchestrate_flux_tests.sh is now fully broken out into modular CI steps — as intended.


| Feature                                      | Status                                            |
| -------------------------------------------- | ------------------------------------------------- |
| Uses `workflow_call` with `inputs`/`secrets` | ✅                                                 |
| Populates `.env` on remote Tester VM         | ✅ via `prepare_test_dir_and_sync_repo.sh`         |
| Starts Orchestrator locally                  | ✅ via `setup_docker.sh` + `start-orchestrator.sh` |
| Runs XPLG containers on Tester VM            | ✅ via SSH block                                   |
| Executes tests on Tester VM                  | ✅ via `run_flux_tests.sh`                         |
| **Merges and Uploads Allure history**        | ✅ (✅ download + ✅ upload added as discussed)      |


🟩 You're Ready To:
Merge this branch into main once stable.

Reuse this CI logic from multiple branches via workflow_call.

Optionally expose parameters like test paths, env config variants, etc.

Let me know when you want to:

Push this to main,

Or start splitting out more steps (e.g., test matrix or log sync),

Or if you'd like help visualizing Allure history in Allure UI.

✅ Summary of What’s Included


| Step                                   | Purpose                                  |
| -------------------------------------- | ---------------------------------------- |
| ✅ `download-artifact` for `history/`   | Ensures Allure retains trends            |
| ✅ `upload-artifact` for `history/`     | Makes history reusable for next run      |
| ✅ `upload-artifact` for `raw-results/` | Helps for debugging or manual inspection |
| ✅ Full orchestration on Tester VM      | Fully broken into reusable steps         |



✅ What is working:
- Tests ran and passed.

- Allure results were generated and pushed correctly.

- run_flux_tests.sh executed fully (your Allure UI contains the latest report).


⚠️ What’s not working:

| Step                       | Issue                                                                                  |
| -------------------------- | -------------------------------------------------------------------------------------- |
| 🔽 Download Allure History | `Error: Artifact not found for name: allure-history` — the artifact doesn’t yet exist. |
| 🔼 Upload Allure History   | `Warning: No files were found` — the history folder didn’t exist or was empty.         |
| 🔼 Upload raw-results      | Same as above — no `raw-results/` folder on the CI runner at that point.               |


| Task                      | Path Used                |
| ------------------------- | ------------------------ |
| `--alluredir=...`         | `raw-results/`           |
| Download artifact         | `raw-results/history/` ✅ |
| Upload artifact (history) | `raw-results/history/` ✅ |
| Upload artifact (all)     | `raw-results/` ✅         |



✅ Key Execution Highlights:

| Step | Operation                                                                             | ✅ Verified |
| ---- | ------------------------------------------------------------------------------------- | ---------- |
| 1️⃣  | SSH into tester VM, activate venv, run `pytest` with `--alluredir=raw-results/`       | ✅          |
| 2️⃣  | Inside tester VM: check for `raw-results/history` and restore it if found             | ✅          |
| 3️⃣  | Inside tester VM: run `allure generate`, then copy new `history/` into `raw-results/` | ✅          |
| 4️⃣  | Back on CI runner: `scp` the full `raw-results/`                                      | ✅          |
| 5️⃣  | On CI runner: generate report again (optional, for debugging or visualization)        | ✅          |
| 6️⃣  | Push results to Allure server                                                         | ✅          |



✅

💡 Final Observations:
You're now properly handling history propagation between runs — which enables Allure’s trend charts and test flakiness tracking.

You're also guarding against missing the CLI (command -v allure) and doing the work conditionally — which is good practice.

The scp now copies the contents of raw-results/, not the folder itself, which avoids nesting problems.

------

#### DAG style architecture

Short answer: yes, the split makes sense—and your current direction already fits it. Here’s how I’d draw the line and what to adjust so it’s clean, predictable, and CI-friendly.

# What lives where

## Repo A — `XpoLog/X8.ODD` (the “orchestrator” / generic automation)

Put **all generic CI + Ansible + Compose logic** here. Nothing suite-specific.

* `.github/workflows/`
  Reusable workflows only (install docker/python, ensure-infra, ensure-compose, run-pytest, push-allure, clone-repo, upload-files, etc.).
* `.ansible/`
  Generic playbooks & roles (deploy\_docker\_compose.yml, load\_\*\_spec.yml, push\_allure\_results.yml, install-docker/python, upload\_from\_orchestrator.yml, etc.), compose templates, inventories.
* `.ci/infra_configs/**`
  Host inventories, clone/upload specs, compose setup lists. These describe *how to prepare a target*, not the tests themselves.
* `.ci/setup_compose/**`
  Per-stack compose configs (your `xplg.compose.yml`, `allure.compose.yml`, etc.).

> Rule of thumb: if it could deploy/prepare **any** service and run **any** pytest suite, it belongs here.

## Repo B — `XpoLog/ODDTests` (the “suite” / project-specific code)

Put **all the test code + suite-specific Python packages and fixtures** here.

* `python/pytest/xplg/flux/src/`

  * `common/` (xconfig, ssh\_helpers, etc.)
  * `suites/flux/src/` (the `flux_test_service` package: `api/`, `infra/`, `flux/`, etc.)
  * `suites/flux/tests/` (pytest tests)
  * `common/openapi_generated_client/` (prefer packaging this cleanly; see below)
  * `pytest.ini`, `.env.sample`, `requirements.txt`
  * `pyproject.toml` (or `setup.cfg`/`setup.py`) for both `flux_test_service` *and* the generated client (or make the client its own package)
* Optional: suite fixtures/templates the tests upload at runtime (e.g., `templates/xml/SyncLogsConfiguration_v2.xml`).

> Rule of thumb: if pytest imports it, or the tests SCP/HTTP it at runtime, it belongs here.

# Pathing/naming nits

* Use a consistent on-VM path: e.g. `/home/xplg/ODDTests/python/pytest/xplg/flux/src` (plural).
  If “ODDTest” is required somewhere, add a symlink on the VM:

  ```
  ln -s /home/xplg/ODDTests /home/xplg/ODDTest
  ```
* Keep imports stable by installing packages **editable** in the venv on the target:

  ```
  pip install -e common/openapi_generated_client
  pip install -e suites/flux
  ```

  (You already validated this flow.)

# The contract between repos (make it explicit)

1. **Infra spec** (in `X8.ODD`): where to deploy, what to clone, what compose to run.
2. **Compose inputs** (in `X8.ODD`): which compose files, env, bind mounts.
3. **Test suite entry** (in `X8.ODD`): which repo to clone (`ODDTests`), where on the VM to place it, and which pytest spec to use.
4. **Runtime env for tests** (in `ODDTests`): `.env.sample` keys and how tests discover hosts, ports, creds. Your `common.xconfig` already handles `.env`.

# CI flow (recommended)

**In `X8.ODD` (your existing orchestrator workflow):**

1. ensure-infra (install python/docker, create dirs)
2. clone repos (clone **X8.ODD** *and* **ODDTests** onto the **remote VM** where tests run)
3. ensure-compose (bring up `flux-src`, `flux-tgt`, etc.)
4. run-pytest (on the VM, in `/home/xplg/ODDTests/python/pytest/xplg/flux/src`, create venv, `pip install -e ...`, then `pytest ...`)
5. push-results-to-allure

> Your earlier question “upload-files passed even without src” — that’s because your `files_json` was empty; the playbook ran with an empty list and succeeded. That’s fine: keep “upload-files” optional (useful for one-off files that aren’t shipped with the suite).

# Packaging the generated client (recommendation)

Right now you editable-install `common/openapi_generated_client` as `xplg-client`. Great. Two options:

* **Best**: make it its own small package (its own repo or subdir) and install with a version (internal index or Git URL). That lets you pin a client version from `ODDTests`.
* **Good**: keep it in `ODDTests/common/openapi_generated_client` with a `pyproject.toml`, and install `-e`. Document how to regenerate it.

Either way, keep the import stable (`import xplg_client` or whatever you choose) and avoid `sys.path` hacks.

# What to move now (checklist)

**Move to `ODDTests`:**

* `common/` python helpers (xconfig, ssh\_helpers)
* `suites/flux/src` python package(s) and `suites/flux/tests`
* `common/openapi_generated_client` (or split as a package)
* test-owned templates/fixtures (XML, small config files) used by the suite’s fixtures to upload via SSH/HTTP.

**Keep in `X8.ODD`:**

* All `.github/workflows/*.yml` reusable workflows (generic)
* All generic `.ansible/playbooks/*.yml` and roles
* `.ansible/compose/**` & compose configs, setup lists
* Infra specs (`.ci/infra_configs/**`) and compose setup lists (`.ci/setup_compose/**`)
* The inputs YAMLs you added for Allure/Compose loaders.

# Import/collection gotchas (learned from your run)

* Always import via installed packages, not `suites.flux.*` paths unless that’s the **installed** package name. You fixed the `No module named 'suites.flux.infra'` by installing `suites/flux` as `flux_test_service`.
* Keep a single `conftest.py` per tests root inside `ODDTests/suites/flux/tests`. Remove stray `conftest.py` in other roots (or make sure they don’t collide).
* `pytest.ini` should live at the **suite root** inside `ODDTests/python/pytest/xplg/flux/src` (not in the orchestrator repo).

# Spec files location

* The **pytest spec** (`.ci/pytest_specs/flux/pytest.tester.yml`) can live in either repo.
  My recommendation: keep **runner/orchestrator specs in `X8.ODD`**, and keep **test behavior/options in `ODDTests`**. The spec references the **on-VM** path of `ODDTests` and the test selection pattern.

# Versioning strategy

* Let `X8.ODD` declare which **branch/ref** of `ODDTests` to clone in the infra spec (or a small JSON file it loads). That gives you a stable cross-repo pin.
* Optionally add a manual dispatch input to override `ODDTests` ref for ad-hoc test runs.

# TL;DR

* ✅ Your split is correct: **generic orchestration in `X8.ODD`**, **suite code in `ODDTests`**.
* ✅ Keep CI/workflows/playbooks/compose in `X8.ODD`.
* ✅ Keep Python tests, helpers, generated client, fixtures in `ODDTests`.
* ✅ Install suite packages editable on the VM; run pytest from the `ODDTests` path.
* ✅ Pin the `ODDTests` ref from `X8.ODD` infra/clone spec.

If you want, I can draft:

* a minimal `ODDTests` tree (with `pyproject.toml` for both packages),
* a `run-pytest-ansi.yml` that assumes **only** a path + marker expression and works against any suite repo,
* a `clone_configs/*.json` that pins `ODDTests` ref cleanly.


##### split the testing service across two repos” plan and make it crisp.

Awesome—glad the playbook path is fixed. Let’s pick up the “split the testing service across two repos” plan and make it crisp.

# ✅ High-level split

**Repo A – `X8.ODD` (producer of reusable CI/Ansible)**

* Reusable GitHub Actions workflows (`workflow_call`)
  `.github/workflows/*.yml` → e.g.:

  * `ensure-infra-compose-orchestrator-ansi.yml`
  * `ensure-compose-ansi.yml`
  * `install-*-on-remote-ansi.yml`
  * `run-pytest-ansi.yml`
  * `push-results-to-allure-ansi.yml`
* **Generic Ansible** (no env/product assumptions)
  `.ansible/playbooks/*.yml` (deploy, upload, load-\*-config, etc.)
* **Generic compose templates** used by tests
  `.ansible/compose/<stack>/*.yml` (e.g., flux docker-compose)
* Optional sample inventory docs (but **avoid** hardcoding your real IPs here)

**Repo B – `ODDTests` (consumer, product/suite specific)**

* The actual **test suite code** and helpers
  `ODDTest/python/pytest/xplg/flux/src/` (what you already moved)
* **Environment/suite configs** consumed by the reusable workflows:

  * `.ci/infra_configs/**` (unified “ensure infra” spec)
  * `.ci/setup_compose/**` (list of compose stacks + per-stack deploy configs)
  * `.ci/pytest_specs/**` (how to run pytest on the target)
  * `.ci/allure/inputs-push-results-to-allure.yml` (your new inputs file)
* Test **data/assets** to upload (e.g. XML templates), declared in the infra spec
* Any **env files** for compose (if you use `compose.env_file` or inline `compose.env`)

This matches what you said: **generic stuff** in `X8.ODD`, **suite/env-specific** in `ODDTests`.

---

# 🔌 How they connect

In `ODDTests` you keep a thin *orchestrator* workflow that **uses** the reusable ones from `X8.ODD`:

```yaml
# ODDTests/.github/workflows/ci.yml
jobs:
  ensure-infra-compose:
    uses: XpoLog/X8.ODD/.github/workflows/ensure-infra-compose-orchestrator-ansi.yml@v0.1.0
    with:
      infra_path: .ci/infra_configs/flux/infra.compose.storm.yml
    secrets:
      ssh-key: ${{ secrets.TESTER_SSH_KEY }}

  run-pytest:
    needs: ensure-infra-compose
    uses: XpoLog/X8.ODD/.github/workflows/run-pytest-ansi.yml@v0.1.0
    with:
      spec_path: .ci/pytest_specs/flux/pytest.tester.yml
    secrets:
      ssh-key: ${{ secrets.TESTER_SSH_KEY }}

  upload-results-to-allure:
    needs: run-pytest
    uses: XpoLog/X8.ODD/.github/workflows/push-results-to-allure-ansi.yml@v0.1.0
    with:
      # Instead of hardcoding, you can now read from a small inputs file:
      # see “Inputs file” below
      inventory_host: storm
      working_dir: /home/xplg/X8.ODD/x8.odd.storm/odd.test
      results_dir: raw-results
      allure_url: "http://10.0.0.138:5050"
      project_id: "flux-sync-ci-reusable-v1"
      force_project_creation: true
      generate_report: true
    secrets:
      ssh-key: ${{ secrets.TESTER_SSH_KEY }}
```

> **Why this works:** when you call a reusable workflow from `X8.ODD`, that job’s workspace is the `X8.ODD` repo, so **its** `.ansible/**` and compose templates are available. Your **configs** (paths under `.ci/**`) remain in `ODDTests`, but those are **passed in** as inputs/extra-vars and only read as files (Ansible `include_vars`) from the `ODDTests` job’s context at the first “load spec” step—already how you’re doing it.

---

# 📦 Where should the docker-compose live?

Two workable patterns:

* **Keep compose files in `X8.ODD`** (current approach).
  Then your per-stack config in `ODDTests` references:

  ```yaml
  compose:
    local_compose_path: .ansible/compose/flux/flux-docker-compose.yml
  ```

  That path is **relative to the X8.ODD workspace** (the reusable workflow repo). This is simple and consistent.

* **Or** move compose files into `ODDTests`, but then the reusable workflow must also check out the consumer repo or accept a second workspace path. That’s extra plumbing. I’d keep compose in `X8.ODD` unless you have a strong reason to couple compose to a suite.

---

# 🧪 Python packages & imports

* Your `common/openapi_generated_client` (installed as `xplg-client`) is broadly useful → consider publishing it to an internal registry (or keep it vendored in `ODDTests` for now).
* Suite code (`suites/flux`) should live in `ODDTests` and be installed in editable mode in the remote VM by the **generic** `run-pytest-ansi.yml` workflow + `pytest.spec` you provide.

---

# 🔐 Secrets and inventory

* Keep secrets in each repo’s settings with **the same names** (e.g., `TESTER_SSH_KEY`) so cross-repo `uses:` calls have access.
* Inventory: you currently point Ansible at `.ansible/inventory/hosts.ini` from `X8.ODD`. That’s fine if your env is stable. If you want env-specific inventories, either:

  * keep **minimal** inventory in `X8.ODD` and pass the **host alias** from `ODDTests` (what you do now), or
  * extend the reusable workflow to accept an **inline inventory** or an inventory file path provided by `ODDTests`.

---

# 🗂️ Inputs file for Allure job (what you just added)

You already created `.ci/allure/inputs-push-results-to-allure.yml` in `ODDTests`. The reusable `push-results-to-allure-ansi.yml` can begin with a small step that loads it (you’ve got the loader playbook in place). That keeps the **values** in the consumer repo and the **mechanics** in the producer repo.

---

# 🚦Migration plan (safe & incremental)

1. **Freeze working branches** in both repos.
2. **In `X8.ODD`:**

   * Move/confirm all generic workflows live under `.github/workflows/` and accept only **generic inputs** (no product strings).
   * Keep generic Ansible and compose under `.ansible/**`.
   * Tag a pre-release, e.g. `v0.1.0`.
3. **In `ODDTests`:**

   * Ensure **all** suite code is under `ODDTest/python/pytest/xplg/flux/src` (as you did).
   * Keep **all configs** under `.ci/**` (infra specs, compose config list, pytest spec, Allure inputs, files to upload).
   * Replace local `uses: ./.github/workflows/...` with cross-repo `uses: XpoLog/X8.ODD/...@v0.1.0`.
4. **Paths sanity:**

   * In `.ci/setup_compose/compose_configs/*.yml`, keep `compose.local_compose_path` pointing to **X8.ODD’s** `.ansible/compose/...`.
   * In infra spec, list `ensure.files` that must be uploaded (e.g., your XML template).
5. **Dry runs:**

   * Run “ensure-infra-compose” only. Confirm docker stack comes up.
   * Run “run-pytest” only. Confirm editable installs & imports work.
   * Run “push-results” only. Confirm Allure receives files.
6. **Cut a real tag** in `X8.ODD` and pin to it in `ODDTests`.

---

# 🧹 Safe cleanup (repo & remote)

* **Repo cleanup:** use ripgrep to see where files are referenced before removing:

  ```bash
  rg -n --hidden -S \
     -e 'uses:\s*\Q./.github/workflows/\E' \
     -e 'with:\s*(infra_path|setup_path|spec_path|files_json|inventory_host|setup_compose_path)' \
     -e '\.ansible/compose/' \
     -e '\.ci/(infra_configs|setup_compose|pytest_specs|allure)/' \
     .
  ```
* **Remote cleanup:** add an optional list of “authoritative files” to your infra spec and a generic Ansible task that **removes everything else** in a target dir (with a feature flag), or keep it manual for now and list stale files in `ensure.files` with `state: absent`.

---

# 🧭 What *not* to do

* Don’t hardcode paths inside generic playbooks to consumer repo files. Always feed them via inputs/extra-vars.
* Don’t rely on accidental artifacts (e.g., “it’s there because a previous job copied it”). Make every needed file explicit in `ensure.files` or derived from a clear config.

---

If you want, I can draft:

* a minimal `ODDTests/.github/workflows/ci.yml` that consumes the three reusable workflows (using your current specs), and
* a tiny README table showing “who owns what” (file → repo).

Say the word and I’ll drop those ready-to-commit snippets.


Yep—that **does** make sense.

* `.ci/infra_configs/flux/infra.compose.storm.yml` is **environment/suite specific**, so it belongs in **`ODDTests`** (the consumer repo).
* The **reusable mechanics** (Ansible playbooks, generic compose templates, reusable Actions) stay in **`X8.ODD`**.

To make this work cleanly when `ODDTests` calls the reusable workflows in `X8.ODD`, do this in the reusable workflows:

## How to wire it up (reusable workflow pattern)

In each reusable workflow in **`X8.ODD`** that needs to read files from the caller (`ODDTests`), perform a **double checkout**:

```yaml
# inside X8.ODD/.github/workflows/ensure-infra-compose-orchestrator-ansi.yml

jobs:
  load:
    runs-on: [self-hosted, linux, orchestrator]
    outputs:
      infra_config: ${{ steps.loadcfg.outputs.infra_config }}
    steps:
      # 1) Checkout the reusable repo (this one) -> has .ansible/** & compose templates
      - name: Checkout X8.ODD (reusable)
        uses: actions/checkout@v4
        with:
          path: x8odd

      # 2) Checkout the caller repo (ODDTests) -> has .ci/** configs
      - name: Checkout caller (ODDTests)
        uses: actions/checkout@v4
        with:
          path: caller
          repository: ${{ github.repository }}
          ref: ${{ github.ref }}

      # 3) Load the infra spec FROM CALLER, run Ansible FROM X8.ODD
      - id: loadcfg
        name: Load & normalize infra spec (v2)
        working-directory: x8odd
        run: |
          ansible-playbook -i localhost, -c local \
            .ansible/playbooks/load_infra_compose_spec.yml \
            --extra-vars "config_path=${{ github.workspace }}/caller/${{ inputs.infra_path }}"
```

Do the same pattern in the other jobs of that reusable workflow:

* Keep `working-directory: x8odd` whenever you run Ansible (so the playbooks and compose templates in X8.ODD are in place).
* When you need a **path from `ODDTests`** (e.g., `.ci/setup_compose/compose_configs/*.yml`, Allure inputs, pytest spec), read it from `${{ github.workspace }}/caller/...`.
* When you need **compose templates or playbooks**, reference them under `x8odd/...`.

> This lets the **config files live in ODDTests** while the **engines live in X8.ODD**.

## About the file in your screenshot

If you want to store a compose file under `ODDTests/topology/xplg/flux/docker/flux-docker-compose-id-1.yml`, that’s fine too. Two options:

1. **Recommended (keep compose generic in X8.ODD):**
   Your per-stack config (in `ODDTests/.ci/setup_compose/compose_configs/xplg.compose.yml`) keeps:

   ```yaml
   compose:
     local_compose_path: .ansible/compose/flux/flux-docker-compose.yml  # in X8.ODD
   ```

   No extra wiring needed.

2. **Compose in ODDTests (caller):**
   Point to the caller path in your per-stack config:

   ```yaml
   compose:
     local_compose_path: ../caller/topology/xplg/flux/docker/flux-docker-compose-id-1.yml
   ```

   Then, when you call the deploy play, set:

   ```yaml
   env:
     GITHUB_WORKSPACE: ${{ github.workspace }}/x8odd
   ```

   and **pass** an extra var like `CALLER_WORKSPACE=${{ github.workspace }}/caller` so your `load_deploy_config.yml` (or the deploy play) can normalize:

   * If `compose.local_compose_path` starts with `../caller/`, rewrite it to an absolute path using `CALLER_WORKSPACE`.

(Keeping compose in `X8.ODD` avoids this special-case logic, so I’d stick with option 1 unless you truly need per-suite compose variants.)

## TL;DR

* **Yes**, moving `infra.compose.storm.yml` (and other `.ci/**` specs) into **`ODDTests`** is the right split.
* Update the reusable workflows in **`X8.ODD`** to **double-checkout** (this repo → `x8odd`, caller repo → `caller`) and:

  * Run Ansible from `x8odd/`.
  * Read suite configs from `caller/`.
  * Keep compose & playbooks in `x8odd/`.

If you want, I can patch one of your reusable workflows (e.g., `ensure-infra-compose-orchestrator-ansi.yml`) with the exact edits so you can copy/paste.

![alt text](image-2.png)

Totally makes sense—and your sketch actually matches the split we’ve been building.

# Big picture

* **`X8.ODD`** = the **engine / services catalog**
  Reusable stuff only: Ansible playbooks, generic Docker/K8s templates, and reusable GitHub Actions.
* **`ODDTests`** = the **payload / test suites**
  All suite- or env-specific code & configs: infra specs, compose selections, pytest specs, test data, and inputs for Allure, etc.

# Correlation to your diagram

**Left (X8.ODD)**

* `topology/docker` (and `k8s`) → canonical templates (e.g., `flux-docker-compose.yml`).
* `.github/workflows` → reusable workflows (`ensure-infra-compose-…`, `run-pytest-…`, `push-results-to-allure-…`).
* (Optional) `x8.oddartifact` → if you later package a tarball of the catalog for offline use; not required for GH reusable workflows.

**Right (ODDTests)**

* `odd.storm/` (infra layer) → the **inputs** describing a target env and services to bring up.

  * e.g. `.ci/infra_configs/flux/infra.compose.storm.yml` (your current file)
  * `topology/` (optional) holds env-specific compose overlays if you don’t want to use the generic ones.
* `odd.test/` (test layer) → pytest specs + assets

  * `.ci/pytest_specs/**` (e.g., `pytest.tester.yml`)
  * `assets/`, `conf/`, `ansible/` as needed
* `odd.codex/`, `odd.scan/` → data/rules or scanning inputs (future).

Two **top-level driver files** in `ODDTests` (names from your sketch):

* `odd.storm.yaml` → your *orchestrator* workflow file (or a small wrapper) that **calls** the reusable workflows from `X8.ODD` using `uses: XpoLog/X8.ODD/.github/workflows/...@<ref>`.
* `odd.test.flux.yaml` → pytest suite inputs/specs (we already have `.ci/pytest_specs/...` doing this).

# How they talk to each other (mechanics)

In each reusable workflow in **X8.ODD**:

1. **Double checkout**

   * `actions/checkout` of **X8.ODD** → path `x8odd/` (contains playbooks & templates).
   * `actions/checkout` of **caller (ODDTests)** → path `caller/` (contains your `.ci/**` inputs).
2. **Run Ansible from `x8odd/`** (so playbooks resolve), but **read inputs from `caller/`**:

   * Example: `config_path=${{ github.workspace }}/caller/.ci/infra_configs/flux/infra.compose.storm.yml`
3. When deploying compose:

   * If `ODDTests` wants to use generic templates, set in the input:
     `compose.local_compose_path: .ansible/compose/flux/flux-docker-compose.yml` (lives in X8.ODD).
   * If `ODDTests` needs a custom compose overlay, point to it in the caller:
     `compose.local_compose_path: topology/xplg/flux/docker/flux-docker-compose-id-1.yml`
     (and the workflow resolves it from `caller/…`).

# Mapping your current files to the model

* `ODDTests/.ci/infra_configs/flux/infra.compose.storm.yml` → lives in **ODDTests** (✅).
* `ODDTests/.ci/setup_compose/compose_configs/*.yml` → lives in **ODDTests** (suite decides which stacks).
* `X8.ODD/.ansible/**` and `.github/workflows/**` → live in **X8.ODD** (shared engine).
* Compose template **default**: `X8.ODD/.ansible/compose/flux/flux-docker-compose.yml`.
* Compose **override** (optional): `ODDTests/topology/xplg/flux/docker/flux-docker-compose-id-1.yml`.

# Minimal repo responsibilities

**X8.ODD**

* Reusable workflows:

  * `ensure-infra-compose-orchestrator-ansi.yml`
  * `run-pytest-ansi.yml`
  * `push-results-to-allure-ansi.yml`
* Playbooks: `.ansible/playbooks/**`
* Templates: `.ansible/compose/**`, `topology/docker/**`, `topology/k8s/**`

**ODDTests**

* Inputs (the “input.spec” idea):

  * Infra: `.ci/infra_configs/**` (e.g., `infra.compose.storm.yml`)
  * Compose set: `.ci/setup_compose/**`
  * Pytest: `.ci/pytest_specs/**`
  * Allure: `.ci/allure/**`
* Test code & assets: `python/`, `src/`, `assets/`, etc.
* Orchestrator workflow(s) that **only** call `uses: XpoLog/X8.ODD/...`.

# Why this makes sense

* Clean separation of **framework** vs **suite**.
* Reusable workflows evolve once (X8.ODD); all suites benefit.
* Each suite (ODDTests) owns its inputs and can override topology per environment without forking the engine.

If you want, I can turn one of your current reusable workflows into the “double-checkout” pattern with exact path fixes so you can drop it in.


Short answer: **yes**. A single “driver” job (e.g. `odd.test.run`) that reads one spec file like

```json
{
  "target": { "inventory_host": "storm" },
  "repo":   { "url": "git@github.com:XpoLog/ODDTests.git", "branch": "Main",
              "dest": "/home/xplg/zigi_testing/clones", "owner": "xplg", "group": "xplg", "mode": "0775" }
}
```

**fits the framework perfectly**:

* **ODDTests** supplies the *spec file(s)* (inputs).
* **X8.ODD** supplies a **single reusable “super-workflow”** that consumes that spec and performs: ensure-infra → clone → (optional) compose → run-pytest → push-to-allure.

Below is a concrete way to wire it.

---

### 1) In **ODDTests** (caller repo): one job only

```yaml
# .github/workflows/odd.test.run.yml (in ODDTests)
name: odd.test.run
on: [push]

jobs:
  odd-test-run:
    uses: XpoLog/X8.ODD/.github/workflows/odd-test-run.yml@<ref>
    with:
      spec_path: .ci/odd.test.run.spec.json   # the JSON you showed
    secrets:
      ssh-key: ${{ secrets.TESTER_SSH_KEY }}
```

Your **spec file** can start minimal (what you posted). Later you can grow it with optional keys:

```json
{
  "target": { "inventory_host": "storm" },
  "repo":   { "url":"git@github.com:XpoLog/ODDTests.git", "branch":"Main", "dest":"/home/xplg/zigi_testing/clones" },
  "ensure": { "python":true, "docker":true, "directories":[...], "files":[...] },
  "compose": { "setup_path": ".ci/setup_compose/setup_compose.yml" },
  "pytest":  { "spec_path": ".ci/pytest_specs/flux/pytest.tester.yml" },
  "allure":  { "url":"http://10.0.0.138:5050", "project_id":"flux-sync-ci", "results_dir":"raw-results",
               "working_dir":"/home/xplg/X8.ODD/x8.odd.storm/odd.test", "generate_report":true, "force_project_creation":true }
}
```

---

### 2) In **X8.ODD** (engine repo): super-workflow

```yaml
# .github/workflows/odd-test-run.yml (reusable)
name: odd-test-run
on:
  workflow_call:
    inputs:
      spec_path: { type: string, required: true }
    secrets:
      ssh-key: { required: true }

jobs:
  # A. Checkout both repos (engine + caller)
  load:
    runs-on: [self-hosted, linux, orchestrator]
    outputs:
      spec_json: ${{ steps.loadspec.outputs.spec_json }}
    steps:
      - uses: actions/checkout@v4           # X8.ODD
        with: { path: engine }
      - uses: actions/checkout@v4           # caller (ODDTests)
        with: { path: caller }

      - id: loadspec
        run: |
          ansible-playbook -i localhost, -c local \
            engine/.ansible/playbooks/load_odd_test_run_spec.yml \
            --extra-vars "config_path=${{ github.workspace }}/caller/${{ inputs.spec_path }}"
        # playbook echoes "spec_json=<JSON>" to GITHUB_OUTPUT

  # B. Ensure infra (reads flags from spec_json)
  ensure:
    needs: load
    uses: XpoLog/X8.ODD/.github/workflows/ensure-infra-compose-orchestrator-ansi.yml@<ref>
    with:
      infra_path: caller/.ci/infra_configs/flux/infra.compose.storm.yml
    secrets:
      ssh-key: ${{ secrets.ssh-key }}

  # C. Run pytest
  run-pytest:
    needs: ensure
    uses: XpoLog/X8.ODD/.github/workflows/run-pytest-ansi.yml@<ref>
    with:
      spec_path: caller/.ci/pytest_specs/flux/pytest.tester.yml
    secrets:
      ssh-key: ${{ secrets.ssh-key }}

  # D. Push results to Allure
  push-allure:
    needs: run-pytest
    uses: XpoLog/X8.ODD/.github/workflows/push-results-to-allure-ansi.yml@<ref>
    with:
      inventory_host: storm
      working_dir: /home/xplg/X8.ODD/x8.odd.storm/odd.test
      results_dir: raw-results
      allure_url: http://10.0.0.138:5050
      project_id: flux-sync-ci
      force_project_creation: true
      generate_report: true
    secrets:
      ssh-key: ${{ secrets.ssh-key }}
```

> From the caller’s point of view this is **ONE job** (`odd-test-run`) even though the reusable workflow has multiple jobs internally.

---

### 3) Loader playbook (engine)

```yaml
# engine/.ansible/playbooks/load_odd_test_run_spec.yml
- hosts: localhost
  gather_facts: false
  vars: { config_path: "" }
  tasks:
    - stat: { path: "{{ config_path }}" }
      register: s
    - fail: { msg: "Spec not found: {{ config_path }}" }
      when: not s.stat.exists
    - include_vars: { file: "{{ config_path }}", name: spec }
    - debug: { msg: "{{ spec | to_nice_json }}" }
    - copy:
        dest: "{{ lookup('env','GITHUB_OUTPUT') }}"
        content: "spec_json={{ spec | to_json }}"
        mode: "0644"
      changed_when: false
```

---

## TL;DR

* Your **single “odd.test.run” job** that starts from a simple JSON spec **100% conforms**.
* Put the spec in **ODDTests**; point your one job to a **reusable super-workflow** in **X8.ODD**.
* The super-workflow parses the spec and runs the same pipeline you already have—just packaged behind one job.


##### how to manual run pytest on VM


```bash
pip install -e common/openapi_generated_client
pip install -e suites/flux
pip install -r requirements.txt