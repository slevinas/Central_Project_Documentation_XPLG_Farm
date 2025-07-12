1. High-Level Architecture & Flow
mermaid
Copy
Edit
flowchart LR
  subgraph DevOps
    GH[GitHub Repository]
    O[Orchestrator<br/>test_server]
    GH -->|push / PR triggers| O
  end

  subgraph Testing
    O -->|SSH & provision| DT[Docker-tester]
    DT -->|docker pull & run XPLG image| XPLG[XPLG Flux Sync<br/>Docker Image]
    DT -->|execute pytest + allure| R[Allure results]
  end

  subgraph Reports
    R -->|upload via HTTP/API| A[ODD-Allure Service]
  end
Push/PR in GitHub → triggers a workflow on your Orchestrator (self-hosted runner).

Orchestrator connects (SSH/API) to the Docker-tester VM:

Installs/upgrades Docker

Pulls the xplghub/images:xplg.Main-10032 image

Runs the container(s) and pytest harness (with Allure plugin)

Docker-tester generates an Allure results bundle and sends it back to ODD-Allure via its REST API.

This mirrors the “pull-mode” architecture in your Flux Sync Test Plan (Center ↔ Remotes) README, but extended with CI orchestration and a reporting service.

2. High-Level Implementation Steps
Provision Servers

Orchestrator-test_server: install GitHub Actions runner, label it (e.g. orchestrator).

Docker-tester: ensure SSH access from orchestrator; choose an OS (Ubuntu/Debian/CentOS).

Write GitHub Actions Workflow

runs-on: self-hosted && orchestrator

Steps: checkout code, set up SSH key, connect to docker-tester to run a remote script.

Create Remote Scripts on Docker-tester

setup-docker.sh: installs Docker if missing.

run-tests.sh: pulls XPLG image, spins up containers (or docker-compose), runs pytest --alluredir=./results, then invokes allurectl (or curl) to upload to ODD-Allure.

Configure Allure Integration

In run-tests.sh, after test execution, post the ./results directory to the Allure server’s API endpoint.

Verify & Iterate

Test the end-to-end flow by pushing a change, watching the Actions run on Orchestrator, and confirming results land in ODD-Allure.

3. Questions / Assumptions to Verify
Runner choice: Do you intend to use a self-hosted Actions runner for the orchestrator, or GitHub-hosted?

SSH access: Will the orchestrator have passwordless SSH (via key) into Docker-tester?

OS for Docker-tester: Which distribution/version (Ubuntu 22.04, etc.)?

Allure upload: Do you have an existing API endpoint or token for ODD-Allure?

Once you confirm (or adjust) these, I can draft:

A sample .github/workflows/ci.yml

The setup-docker.sh and run-tests.sh scripts

Any necessary SSH/Allure configuration snippets

Does this high-level design look right? Let me know, and we’ll jump into the concrete implementation.



##### Proposal structure

```bash
## Monorepo Scaffold: `testing_services/flux_test_service`

Below is a proposed initial scaffold for your `flux_test_service` harness within `ODD-X8/src/testing_services/`.


ODD-X8/
└── src/
    ├── common/
    │   ├── __init__.py
    │   ├── ssh_helpers.py        # SSH connection utilities
    │   ├── docker_utils.py       # Docker control helpers
    │   └── allure_client.py      # HTTP client to upload to Allure
    │
    └── testing_services/
        └── flux_test_service/
            ├── src/
            │   └── flux_test_service/
            │       ├── __init__.py
            │       ├── runner.py    # starts Flux containers & orchestration logic
            │       └── uploader.py  # wraps `common.allure_client` for uploads
            │
            ├── tests/
            │   ├── conftest.py     # pytest fixtures (e.g. SSHClient fixture)
            │   ├── test_runner.py  # unit tests for runner logic
            │   └── test_uploader.py# unit tests for uploader logic
            │
            ├── scripts/
            │   ├── setup-docker.sh         # installs Docker on Ubuntu 24
            │   ├── install-python-env.sh   # sets up venv & pip installs
            │   ├── start-flux-tests.sh     # your boss’s "start" step + test invocation
            │   └── upload-to-allure.sh     # POST results to ODD-Allure
            │
            ├── pyproject.toml     # poetry or PEP 621 packaging
            ├── requirements-dev.txt # pytest, lint, type-check deps
            ├── tox.ini            # matrix & linting configs
            ├── README.md         # overview & quickstart
            └── .github/
                └── workflows/
                    └── flux-test.yml # CI workflow for GitHub Actions


ODD-X8/
└── src/
    ├── common/                        ← shared helpers (SSH, Docker, Allure clients)
    │   ├── ssh_helpers.py
    │   ├── docker_utils.py
    │   └── allure_client.py
    │
    └── testing_services/              ← all your test harnesses
        ├── flux_test_service/         ← this project
        │   ├── src/
        │   │   └── flux_test_service/
        │   │       ├── __init__.py
        │   │       ├── runner.py       ← starts Flux, spins up containers
        │   │       └── uploader.py     ← talks to common.allure_client
        │   │
        │   ├── tests/                  ← pytest unit/integration
        │   │   ├── conftest.py
        │   │   ├── test_runner.py
        │   │   └── test_uploader.py
        │   │
        │   ├── scripts/                ← shell entrypoints run on Docker-tester
        │   │   ├── setup-docker.sh
        │   │   ├── install-python-env.sh
        │   │   ├── start-flux-tests.sh
        │   │   └── upload-to-allure.sh
        │   │
        │   ├── pyproject.toml
        │   ├── requirements-dev.txt
        │   ├── tox.ini
        │   ├── README.md
        │   └── .github/
        │       └── workflows/flux-test.yml
        │
        └── other_test_service/         ← future harnesses
            └── …


```

- common/ is for utilities you’ll share across all testing services.

- testing_services/flux_test_service/ is this harness: Python code, tests, and the scripts that run on your Ubuntu 24 “Docker-tester.”

- Later you can add testing_services/load_test_service/ or testing_services/security_test_service/ alongside it.

**With this structure, your self-hosted runner job simply checks out the repo, SSH’s into the DO droplet, and runs:**

```bash

cd ~/ODD-X8/src/testing_services/flux_test_service
scripts/setup-docker.sh
scripts/install-python-env.sh
scripts/start-flux-tests.sh
pytest --alluredir=allure-results
scripts/upload-to-allure.sh
```

That keeps everything modular, discoverable, and ready to scale as a “testing services” suite.

