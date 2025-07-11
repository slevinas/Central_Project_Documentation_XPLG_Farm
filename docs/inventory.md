# Inventory – xplg-farm-flux-testing-suit

This document lists all hosts in the **xplg-farm-flux-testing-suit** project, grouped by role and with their SSH connection details.

| Hostname       | Role | IP / DNS Name      | SSH User    | SSH Key                            | Description              |
| -------------- | ---- | ------------------ | ----------- | ---------------------------------- | ------------------------ |
| server-01-web  | web  | `<WEB_SERVER_IP>`  | ubuntu      | `~/.ssh/xplg-farm-flux-testing.pem` | Front-end application    |
| server-02-db   | db   | `<DB_SERVER_IP>`   | ubuntu      | `~/.ssh/xplg-farm-flux-testing.pem` | PostgreSQL database      |
| server-03-ci   | ci   | `<CI_SERVER_IP>`   | git-runner  | `~/.ssh/xplg-farm-flux-testing.pem` | GitHub Actions runner    |

---

## Group-wide defaults

Rather than repeating these on every host, we define them once in `ansible/group_vars/all.yml`:

```yaml
ansible_user: ubuntu
ansible_ssh_private_key_file: ~/.ssh/xplg-farm-flux-testing.pem
ansible_become: true
