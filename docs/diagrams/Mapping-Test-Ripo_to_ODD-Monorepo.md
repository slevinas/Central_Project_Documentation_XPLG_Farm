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







```mermaid
flowchart LR
  subgraph Monorepo
    direction LR
    common["src/common\nAllureClient (HTTP wrapper)"]
    flux["src/testing_services/flux_test_service\nuploader.py"]
    results["pytest allure-results/ directory"]
  end

  results -->|reads directory + metadata| flux
  flux -->|instantiates AllureClient| common
  common -->|POST results + metadata| ODD["ODD-Allure Service API"]
````

### Directory Tree

````
## Monorepo Scaffold: `testing_services/flux_test_service`

Below is a proposed scaffold for `flux_test_service` harness within `ODD-X8/src/testing_services/`, now including how the HTTP-wrapper (`AllureClient`) in `common/` and the service-level `uploader.py` interact as part of the call flow diagram.

```mermaid
flowchart LR
  subgraph Monorepo Call Flow
    direction LR
    results["pytest allure-results/ directory"]
    flux["uploader.py
(src/testing_services/flux_test_service)"]
    common["AllureClient (HTTP wrapper)
(src/common)"]
    results -->|reads directory + metadata| flux
    flux -->|instantiates AllureClient| common
    common -->|POST results + metadata| ODD["ODD-Allure Service API"]
  end
````

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

```

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
