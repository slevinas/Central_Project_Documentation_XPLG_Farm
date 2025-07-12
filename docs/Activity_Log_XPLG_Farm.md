

#### Table of contents

- [install Docker](#install-docker)











-----
##### install Docker 

1. Verify or install Docker on the DO box
###### 1.1. Check if Docker is already installed

```bash

docker --version
```
If you see something like Docker version 24.x.x, you’re all set.

Otherwise, install Docker CE:

```bash

# Update apt and install prerequisites  
apt update  
apt install -y ca-certificates curl gnupg lsb-release  

# Add Docker’s official GPG key and repo  
mkdir -p /etc/apt/keyrings  
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg  

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null  
```

#### Install Docker Engine  
```bash
apt update  
apt install -y docker-ce docker-ce-cli containerd.io  
```
- Add your user to the docker group (so you can run Docker without sudo):

```bash

usermod -aG docker $USER
# then log out & back in or restart your SSH session
```
#### 2. Run the Allure Docker container
We’ll mount your reports directory into the container and expose the standard Allure port (usually 8080).

Create a stable mount point
Suppose your project lives under /root/flux-data-sync-phase2; we’ll serve reports from there:

```bash

mkdir -p /root/flux-data-sync-phase2/reports
```
Run the Allure container

```bash

docker run -d \
  --name allure-server \
  -p 8080:8080 \
  -v /root/flux-data-sync-phase2/reports:/app/allure-results:ro \
  frankescobar/allure-docker-service
-v …/reports:/app/allure-results:ro
```
mounts your host’s reports/ into the container’s watched results folder.

The container will automatically pick up any new allure-results subfolder inside and make it available in the UI.

Verify it’s running

```bash

docker ps
```
You should see frankescobar/allure-docker-service listening on 0.0.0.0:8080.

Browse to http://<DO-IP>:8080/ in your browser.
You’ll see a list of builds (one per timestamped folder under reports/).

-----





-----
# Activity Log – xplg-farm

This document is a centralized record of all tasks, requests, and changes performed on the **xplg-farm** infrastructure. It serves as your personal reference to track what you've done, what you’re doing, and what's next.

| Date       | Task / Request                                     | Servers      | Details / Notes                                                                                              | Status    |
| ---------- | -------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------ | --------- |
| 2025-07-10 | GitHub Actions runner installation                 | server-03-ci | Installed runner service under `git-runner`                                                                  | Completed |
| 2025-07-10 | Install GitHub Actions runner & connect repository | server-03-ci | Installed runner service under `git-runner` and connected it to the `xplg-farm-flux-testing-suit` repository | Completed |

## Component Control Reference

Below is a table listing each key component, the server it runs on, its installation or working directory, and the basic commands to start, stop, check status, and view logs.

| Component             | Server        | Directory / Path              | Start Command                        | Status Command                        | Logs Path                           |
| --------------------- | ------------- | ----------------------------- | ------------------------------------ | ------------------------------------- | ----------------------------------- |
| XPLG VPN              | MacOs         | ``                            | [Start VPN](#start-vpn)              | [](#s)                                | ``                                  |
| XPLG Test-Server      | server-122    | ``                            | [Connect to Tester](#connect-tester) | [](#s)                                | ``                                  |
| XPLG Test-Server      | server-odd    | ``                            | [Connect to Flux1](#connect-flux1)   | [](#s)                                | ``                                  |
| XPLG Test-Server      | server-odd    | ``                            | [Connect to Flux1](#connect-flux1)   | [](#s)                                | ``                                  |
| GitHub Actions Runner | server-03-ci  | `/opt/runners/actions-runner` | [Start Runner](#runner-start)        | [Check Runner Status](#runner-status) | `/opt/runners/actions-runner/_diag` |
| XPLG Application      | server-01-web | `/opt/xplg/app`               | [Start XPLG App](#xplg-start)        | [Check XPLG Status](#xplg-status)     | `/var/log/xplg/app.log`             |

_(Add more rows for additional components as needed.)_

## Command Reference

For convenience, each command used above is defined with headings that GitHub generates IDs for. Click the links in the table to jump to details below.

### MacOs Commands

##### Start VPN

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

<h3 id="connect-tester">Connect to Tester</h3>

```bash
ssh xlpg-85
```

<h3 id="connect-flux1">Connect to Flux1</h3>

```bash
ssh xlpg-85
```

<h3 id="runner-start">Start Runner</h3>
```bash
sudo systemctl start actions.runner.slevinas-XPLG-Flux-Data-Sync-Tests.flux-test-runner-122.service
```

<h3 id="runner-status">Check Runner Status</h3>
```bash
systemctl status actions.runner.slevinas-XPLG-Flux-Data-Sync-Tests.flux-test-runner-122.service
```

<h3 id="runner-logs">View Runner Logs</h3>
```bash
ls -l /opt/runners/actions-runner/_diag
```

<h3 id="xplg-start">Start XPLG App</h3>
```bash
sudo systemctl start xplg-app.service
```

<h3 id="xplg-status">Check XPLG Status</h3>
```bash
systemctl status xplg-app.service
```

<h3 id="xplg-logs">View XPLG Logs</h3>
```bash
ls -l /var/log/xplg/app.log
```

## Command Reference

For convenience, each command used above is defined here with anchors. Click the links in the table to jump to details below.

---

---

#### Quick workflow example (VS Code + Markdown All in One)

1. Paste a duplicate of the last table row below.

2. Change “2025-07-10” → “2025-07-11”, update the Task/Servers/etc.

3. Hit Ctrl+Shift+P → “Markdown: Format Table”.

Done — no manual dashes required.

---
