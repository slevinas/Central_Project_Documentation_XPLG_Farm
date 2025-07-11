

```lua

project-root/
├── docs/
│   ├── README.md              ← Overview & nav links
│   ├── architecture.md        ← High-level sketch (network, roles)
│   ├── inventory.md           ← Hostnames, IPs, roles, OS
│   ├── server-01-web.md       ← Web server setup & services
│   ├── server-02-db.md        ← Database server setup
│   ├── server-03-ci.md        ← CI runner setup
│   ├── playbooks/             ← (optional) Ansible playbooks
│   │   └── setup-web.yml
│   └── diagrams/
│       └── xplg-farm-network.mmd  ← Mermaid network diagram
├── ansible/
│   ├── inventory.ini
│   ├── group_vars/
│   └── roles/
└── README.md                  ← Pointer to docs/README.md


```

git add .
git commit -m "chore: add project docs & ansible directory skeleton"