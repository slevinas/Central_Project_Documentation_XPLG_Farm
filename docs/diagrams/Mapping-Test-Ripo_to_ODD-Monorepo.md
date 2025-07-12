Below is a side-by-side sketch showing roughly how the existing XPLG-Flux-Data-Sync-Tests repo would relocate into the new ODD-X8 monorepo under src/testing_services/flux_test_service. I’ve grouped related files and dropped the auto-generated caches (**pycache**, pytest_temp, etc.):

```text

┌────────────────────────────────────────────────┐    ┌─────────────────────────────────────────────────────────────────┐
│  Original:                                    │    │  New Monorepo:                                               │
│  XPLG-Flux-Data-Sync-Tests/                   │    │  ODD-X8/                                                     │
│  ├── README.md                                │    │  └── src/                                                   │
│  ├── TasksXMLs/                               │    │      ├── common/                                            │
│  ├── tasks-templates/                         │    │      │   └── openapi_generated_client/  ← (optional share)    │
│  ├── scripts/                                 │    │      │       └── xplg_client/         ←  OpenAPI client   │
│  │   ├── run_flux_tests.sh                    │    │      └── testing_services/                                  │
│  │   ├── run_flux_tests_v2.sh                 │    │          └── flux_test_service/                            │
│  │   └── copy-only-paths-with.sh              │    │              ├── src/                                     │
│  ├── utils/                                   │    │              │   └── flux_test_service/                     │
│  │   ├── ssh/                                 │    │              │       ├── __init__.py                          │
│  │   │   └── ssh_helpers.py  ←──────── ─────┐ │    │              │       ├── config.py         ← xconfig.py   │
│  │   ├── api/                                │ │    │              │       ├── api/            ← utils/api/     │
│  │   │   ├── api_create_task.py              │ │    │              │       ├── flux/           ← utils/flux/    │
│  │   │   └── api_delete_task.py              │ │    │              │       ├── general/        ← utils/general/ │
│  │   └── flux/…                              │ │    │              │       ├── runner.py                          │
│  ├── openapi_generated_client/               │ │    │              │       └── uploader.py                        │
│  ├── conftest.py            ← pytest fixtures│ │    │              ├── tests/                                   │
│  ├── pytest.ini                              │ │    │              │   ├── conftest.py       ← original             │
│  ├── requirements.in/.txt/-lock              │ │    │              │   ├── test_runner.py      (stubs)             │
│  ├── xconfig.py                              │ │    │              │   └── test_uploader.py                         │
│  ├── sync_metrics.csv                        │ │    │              ├── scripts/                                 │
│  └── …                                       │ │    │              │   ├── setup-docker.sh                         │
└────────────────────────────────────────────────┘ │    │              │   ├── install-python-env.sh                   │
                                                    └───▶│              │   ├── start-flux-tests.sh    ← run_flux…  │
                                                        │              │   ├── start-flux-tests-v2.sh ← …_v2.sh     │
                                                        │              │   └── upload-to-allure.sh                      │
                                                        │              ├── templates/                              │
                                                        │              │   ├── xml/           ← TasksXMLs/             │
                                                        │              │   └── jinja/         ← tasks-templates/       │
                                                        │              ├── assets/                                 │
                                                        │              │   └── sync_metrics.csv                      │
                                                        │              ├── pytest.ini                              │
                                                        │              ├── requirements.txt                        │
                                                        │              ├── requirements-dev.txt                    │
                                                        │              ├── pyproject.toml                          │
                                                        │              ├── README.md                               │
                                                        │              └── .github/                                │
                                                        │                  └── workflows/                          │
                                                        │                      └── flux-test.yml                   │
                                                        └──────────────────────────────────────────────────────────┘

```

````
## Monorepo Scaffold: `testing_services/flux_test_service`

Below is a proposed scaffold for a `flux_test_service` harness within `ODD-X8/src/testing_services/`, now including
 how the HTTP-wrapper (`AllureClient`) in `common/` and the service-level `uploader.py` interact as part of the call flow diagram.








### Directory Tree

````
## Monorepo Scaffold: `testing_services/flux_test_service`

Below is a proposed scaffold for `flux_test_service` harness within `ODD-X8/src/testing_services/`, now including how the HTTP-wrapper (`AllureClient`) in `common/` and the service-level `uploader.py` interact as part of the call flow diagram.



### Directory Tree Scaffold

```
ODD-X8/
└── src/
    ├── common/
    │   ├── __init__.py
    │   ├── ssh_helpers.py        # SSH connection utilities
    │   ├── docker_utils.py       # Docker control helpers
    │   └── allure_client.py      # HTTP client to upload to Allure
    |   └── openapi_generated_client/  ← (optional share)
    |       └── xplg_client/         ← Our OpenAPI client
    │
    └── testing_services/
        └── flux_test_service/
            ├── src/
            │   └── flux_test_service/
            │       ├── __init__.py
            │       ├── runner.py    # starts Flux containers & orchestration logic
            │       └── uploader.py  # packages results + uses AllureClient
            │
            ├── tests/
            │   ├── conftest.py     # pytest fixtures (e.g. SSHClient fixture)
            │   ├── test_runner.py  # unit tests for runner logic
            │   └── test_uploader.py# unit tests for uploader logic
            │
            ├── scripts/
            │   ├── setup-docker.sh         # installs Docker on Ubuntu 24
            │   ├── install-python-env.sh   # sets up venv & pip installs
            │   ├── start-flux-tests.sh     #  "start" step + test invocation
            │   └── upload-to-allure.sh     # POST results to ODD-Allure via uploader
            │
            ├── pyproject.toml     # poetry or PEP 621 packaging
            ├── requirements-dev.txt # pytest, lint, type-check deps
            ├── tox.ini            # matrix & linting configs
            ├── README.md         # overview & quickstart
            └── .github/
                └── workflows/
                    └── flux-test.yml # CI workflow for GitHub Actions
```

---

## Mapping Existing `XPLG-Flux-Data-Sync-Tests` into Monorepo

Here’s how current repo would relocate into the above scaffold:

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

With this updated diagram, you can see how:

1. **pytest** writes results into the `allure-results/` directory.
2. **`uploader.py`** loads those results, attaches any metadata, and uses the shared **`AllureClient`** from `common/`.
3. **`AllureClient`** handles the HTTP POST to **ODD-Allure** server.

Diagram of the Call Flow

```plaintext

flux_test_service.uploader.py
├─ determines:  “Here are my results + metadata”
└─ calls common.allure_client.AllureClient
       └─ handles HTTP, headers, retries, etc.
```

By keeping the raw HTTP logic in common, you avoid duplicating header-construction, error-handling, and retry logic. Meanwhile each service’s uploader.py can remain focused on its own needs (where to find results, what metadata to attach).


Key mappings

utils/ssh/ssh_helpers.py → src/common/ssh_helpers.py

utils/api/* → src/testing_services/flux_test_service/src/flux_test_service/api/

utils/flux/ → src/.../flux_test_service/flux/

conftest.py, pytest.ini → flux_test_service/tests/ & service root

scripts/run_flux_tests*.sh → flux_test_service/scripts/start-flux-tests*.sh

TasksXMLs/, tasks-templates/ → flux_test_service/templates/ (split into xml/ & jinja/)

openapi_generated_client/xplg_client/ → src/common/openapi_generated_client/xplg_client/

xconfig.py → flux_test_service/src/flux_test_service/config.py

sync_metrics.csv → flux_test_service/assets/

With this mapping you keep all Flux-Data-Sync-Tests logic inside the new flux_test_service, share low-level helpers in common/, and standardize CI under .github/workflows/flux-test.yml. Let me know if any specific file needs a different home!


```
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

#### Step-by-Step
1. **Orchestrator Server**

- Install Docker and register the self-hosted GitHub Actions runner.

- Pull & run the XPLG Flux-Sync Docker image so the service under test is live at e.g. http://<orchestrator>:30303.

2. **Workflow Trigger**

- A push or PR in the ODD-X8 repo fires your .github/workflows/flux-test.yml on the Orchestrator runner.

3. **On Docker-Tester (flux-agent)**

- 3.1. Install Docker (if not already)

- 3.2. Install Python + pytest via install-python-env.sh

- 3.3. Clone the flux_test_service folder from your monorepo

- 3.4. start-flux-tests.sh

 - (Optionally) pull/run its own container or just prepare test inputs

 - Wait for the Orchestrator’s API to be healthy

- 3.5. **Run** pytest --alluredir=allure-results against the Orchestrator’s Flux-Sync API

- 3.6. **Upload** results via upload-to-allure.sh, which uses your service’s uploader.py + common.AllureClient

4. **Reporting**

- The shared AllureClient (in src/common/allure_client.py) handles the HTTP POST to your ODD-Allure API, tagging the run with metadata (branch, build number, timestamp).

This keeps a clear separation:

Orchestrator hosts the service under test.

flux-agent (DockerTester) runs the test harness.

common/AllureClient is reused by any future testing service.

---

#### helpers


```bash

 1850  ls -l *log4j*
 1851  cat log4jXpolog.properties 
 1852  ip a
 1853  cat /etc/resolv.conf
 1854  cat /etc/netplan/00-installer-config.yaml 
 1855  vi /etc/netplan/00-installer-config.yaml 
 1856  sudo vi /etc/netplan/00-installer-config.yaml 
 1857  netplan apply
 1858  sudo netplan apply
 1859  ip a
 1860  sudo apt-get remove docker docker-engine docker.io containerd runc
 1861  sudo apt-get update
 1862  sudo apt-get install -y     ca-certificates     curl     gnupg     lsb-release
 1863  sudo mkdir -p /etc/apt/keyrings
 1864  curl -fsSL https://download.docker.com/linux/ubuntu/gpg |   sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
 1865  echo   "deb [arch=$(dpkg --print-architecture) \
 1866    signed-by=/etc/apt/keyrings/docker.gpg] \
 1867    https://download.docker.com/linux/ubuntu \
 1868    $(lsb_release -cs) stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 1869  sudo apt-get update
 1870  sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
 1871  sudo usermod -aG docker $USER
 1872  docker ps
 1873  sudo docker ps

```

* plan to implement Step A (Install Docker & GitHub Runner) on your Orchestrator (TestService) server—and how to verify it works:


#### 1. Install Docker
Create a script scripts/setup-docker.sh in your flux_test_service (or under infrastructure/) with:

```bash

#!/usr/bin/env bash
set -euo pipefail

# 1. Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# 2. Add Docker’s GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 3. Add the Docker APT repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker Engine & tools
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. (Optional) Allow non-root Docker usage
sudo usermod -aG docker "$USER"

echo "Docker installed. You may need to log out and back in for group changes to apply."
```
Make it executable:

```bash

chmod +x scripts/setup-docker.sh
```
**How to test**
1. Run the script

```bash

./scripts/setup-docker.sh
```
2. **Verify Docker version**

```bash

docker --version
# should print something like: Docker version 24.0.2, build…
```
3. **Run the hello-world container**

```bash

docker run --rm hello-world
```
You should see “Hello from Docker!” in the logs.

#### 2. Install & Configure the Self-Hosted GitHub Actions Runner
Create a script scripts/configure-runner.sh:

bash
Copy
Edit
#!/usr/bin/env bash
set -euo pipefail

# 1. Variables – adjust these
ORG_OR_REPO_URL="https://github.com/your-org/ODD-X8"
RUNNER_NAME="orchestrator"
RUNNER_LABELS="self-hosted,orchestrator"

# 2. Create runner directory
mkdir -p actions-runner && cd actions-runner

# 3. Download the latest runner package
#    Check https://github.com/actions/runner/releases for newest version
RUNNER_VERSION="2.326.0"
curl -O -L \
  "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"
tar xzf "actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"

# 4. Fetch a registration token from GitHub UI:
#    GitHub → ODD-X8 → Settings → Actions → Runners → New self-hosted runner → copy token
read -p "Enter GitHub registration token: " REG_TOKEN

# 5. Configure the runner (unattended)
./config.sh --unattended \
  --url "${ORG_OR_REPO_URL}" \
  --token "${REG_TOKEN}" \
  --name "${RUNNER_NAME}" \
  --labels "${RUNNER_LABELS}"

# 6. Install as a systemd service & start
sudo ./svc.sh install
sudo ./svc.sh start

echo "Runner configured and started."
Make it executable:

bash
Copy
Edit
chmod +x scripts/configure-runner.sh
How to test
Run the script on your orchestrator VM:

bash
Copy
Edit
./scripts/configure-runner.sh
Paste in the token when prompted.

Check runner status

bash
Copy
Edit
cd actions-runner
sudo ./svc.sh status
You should see it running.

Verify in GitHub

Go to ODD-X8 → Settings → Actions → Runners

You should see your runner named orchestrator online.

Smoke-test a workflow
Create .github/workflows/runner-test.yml:

yaml
Copy
Edit
name: Runner Smoke Test

on: [workflow_dispatch]

jobs:
  ping:
    runs-on: [self-hosted, orchestrator]
    steps:
      - run: echo "✅ Runner is alive on $(hostname) at $(date)!"
In GitHub, click Run workflow on that file. Confirm the job picks up on your orchestrator and prints the message.

3. Pull & Run the XPLG Flux-Sync Image (Steps B & C)
Once Docker and the runner are in place:

bash
Copy
Edit
# 1. Pull the image
docker pull xplghub/images:xplg.Main-10032

# 2. Run the container (detached, map port)
docker run -d \
  --name flux-sync-orch \
  -p 30303:30303 \
  xplghub/images:xplg.Main-10032
Test that the API is up:

bash
Copy
Edit
curl -I http://localhost:30303/health
# Expect HTTP/1.1 200
With those three pieces in place—Docker, the Actions runner, and your service container—you have fully realized Step A (and B/C) on the orchestrator. The next time you push to your monorepo, your runner can begin orchestrating tests on the DockerTester VM.