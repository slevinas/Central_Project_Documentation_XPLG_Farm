# Activity Log – xplg-farm

This document is a centralized record of all tasks, requests, and changes performed on the **xplg-farm** infrastructure. It serves as your personal reference to track what you've done, what you’re doing, and what's next.

| Date       | Task / Request                                     | Servers      | Details / Notes                                                                                              | Status    |
| ---------- | -------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------ | --------- |
| 2025-07-10 | GitHub Actions runner installation                 | server-03-ci | Installed runner service under `git-runner`                                                                  | Completed |
| 2025-07-10 | Install GitHub Actions runner & connect repository | server-03-ci | Installed runner service under `git-runner` and connected it to the `xplg-farm-flux-testing-suit` repository | Completed |

## Component Control Reference

Below is a table listing each key component, the server it runs on, its installation or working directory, and the basic commands to start, stop, check status, and view logs.

| Component             | Server        | Directory / Path              | Start Command                      | Status Command                        | Logs Path                           |
| --------------------- | ------------- | ----------------------------- | ---------------------------------- | ------------------------------------- | ----------------------------------- |
| XPLG VPV              | MacOs         | ``                            | [Start VPN](#xplg-start)           | [](#s)                                | ``                                  |
| XPLG Test-Server      | server-flux1  | ``                            | [Connect to Flux1](#connect-flux1) | [](#s)                                | ``                                  |
| GitHub Actions Runner | server-03-ci  | `/opt/runners/actions-runner` | [Start Runner](#runner-start)      | [Check Runner Status](#runner-status) | `/opt/runners/actions-runner/_diag` |
| XPLG Application      | server-01-web | `/opt/xplg/app`               | [Start XPLG App](#xplg-start)      | [Check XPLG Status](#xplg-status)     | `/var/log/xplg/app.log`             |

_(Add more rows for additional components as needed.)_

## Command Reference

For convenience, each command used above is defined here with anchors. Click the links in the table to jump to details below.

---

### MacOs Commands

- \<a name="start-vpn"></a>**Start VPN:**

  ```bash
  # 1 start vpn

  sudo openfortivpn vpn.xplg.com:64334 \
  -u slevinas \
  -p 'C0nnectTmrw1&' \
  --trusted-cert b67e25a79728362799a2950efa705a6ee19658180e4bdf813dab053856e0111b \
  --persistent=10 \
  --set-dns=1 \
  --set-routes=1 \
  -v
  ## 2 enter Macos Pass
  ```

- \<a name="connect-flux1"></a>**Connect to Flux1:**

  ```bash
  ssh xlpg-85
  ```

### GitHub Actions Runner Commands

- \<a name="runner-start"></a>**Start Runner:**

  ```bash
  sudo systemctl start actions.runner.slevinas-XPLG-Flux-Data-Sync-Tests.flux-test-runner-122.service
  ```

- \<a name="runner-status"></a>**Check Runner Status:**

  ```bash
  systemctl status actions.runner.slevinas-XPLG-Flux-Data-Sync-Tests.flux-test-runner-122.service
  ```

- \<a name="runner-logs"></a>**View Runner Logs:**

  ```bash
  ls -l /opt/runners/actions-runner/_diag
  ```

### XPLG Application Commands

- \<a name="xplg-start"></a>**Start XPLG App:**

  ```bash
  sudo systemctl start xplg-app.service
  ```

- \<a name="xplg-status"></a>**Check XPLG Status:**

  ```bash
  systemctl status xplg-app.service
  ```

- \<a name="xplg-logs"></a>**View XPLG Logs:**

  ```bash
  ls -l /var/log/xplg/app.log
  ```

---

#### Quick workflow example (VS Code + Markdown All in One)

1. Paste a duplicate of the last table row below.

2. Change “2025-07-10” → “2025-07-11”, update the Task/Servers/etc.

3. Hit Ctrl+Shift+P → “Markdown: Format Table”.

Done — no manual dashes required.

---
