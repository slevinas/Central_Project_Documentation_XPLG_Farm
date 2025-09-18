##### Table of Contents

- [Got to Bottom](#bottom)


- [Prove the collector can read the file](#1-prove-the-collector-can-read-the-file)

- [Add the cluster’s own Tomcat logs (catalina.*.log)]

----

Absolutely—let’s bake **Option B** right into the compose so each node’s **Tomcat** and **app** logs are bind-mounted to the host for persistence and easy inspection.

# Updated `docker-compose.yml` (with host-persisted logs)

```yaml
version: "3.8"

name: xplg-graal-cluster
networks: { xplgnet: {} }

services:

  xplg-master:
    image: xplghub/images:xplg.RC-Graal-10206
    container_name: xplg-master
    restart: unless-stopped
    networks: [xplgnet]
    expose: ["30303","30443"]
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=master"
      xplg_inception_init_flow: "master"
      xplg_inceptions_post_init_flow_tasks: "clusterExternalConf"
    volumes:
      # shared external config (dev-friendly bind)
      - type: bind
        source: ${HOST_DATA:-./data}
        target: /home/data
      # persist MASTER logs to host
      - ./cluster-logs/master/tomcat:/opt/xplg-service/ServletContainer/logs
      - ./cluster-logs/master/app:/opt/xplg-service/log

  xplg-ui:
    image: xplghub/images:xplg.RC-Graal-10206
    container_name: xplg-ui
    restart: unless-stopped
    networks: [xplgnet]
    ports:
      - "30307:30303"  # HTTP
      - "30447:30443"  # HTTPS
    depends_on: [xplg-master]
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=ui"
      xplg_inception_init_flow: "ui"
      xplg_inceptions_post_init_flow_tasks: "waitForFile"
    volumes:
      - type: bind
        source: ${HOST_DATA:-./data}
        target: /home/data
      # persist UI logs to host
      - ./cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs
      - ./cluster-logs/ui/app:/opt/xplg-service/log

  xplg-processor:
    image: xplghub/images:xplg.RC-Graal-10206
    container_name: xplg-processor
    restart: unless-stopped
    networks: [xplgnet]
    expose: ["30303","30443"]
    depends_on: [xplg-master]
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=processor"
      xplg_inception_init_flow: "processor"
      xplg_inceptions_post_init_flow_tasks: "waitForFile"
    volumes:
      - type: bind
        source: ${HOST_DATA:-./data}
        target: /home/data
      # persist PROCESSOR logs to host
      - ./cluster-logs/processor/tomcat:/opt/xplg-service/ServletContainer/logs
      - ./cluster-logs/processor/app:/opt/xplg-service/log
```

### One-time host prep

```bash
mkdir -p ./data ./cluster-logs/ui/{tomcat,app} \
         ./cluster-logs/master/{tomcat,app} \
         ./cluster-logs/processor/{tomcat,app}
docker compose up -d --force-recreate
```

### What you gain

* Each node writes its **Tomcat logs** to:

  * UI → `./cluster-logs/ui/tomcat/`
  * MASTER → `./cluster-logs/master/tomcat/`
  * PROCESSOR → `./cluster-logs/processor/tomcat/`
* Each node’s **application logs** land in the matching `app/` folder.
* The app still writes to the in-container paths (`/opt/xplg-service/ServletContainer/logs` and `/opt/xplg-service/log`), so nothing changes inside the product—**you just get persistence + host visibility**.

### Adding the cluster’s own logs as sources (in the UI)

1. **Add Data → Source Type → Local**.
2. At the top, set **Server** to the node you want (e.g., `xplg-ui`).
3. **Path** (examples):

   * Tomcat: `/opt/xplg-service/ServletContainer/logs/catalina.*.log`
   * App: `/opt/xplg-service/log/*.log`
4. Save → **Collect Now**.

> Because we bind-mounted those directories, you can also tail them on the host (e.g., `tail -f cluster-logs/ui/tomcat/catalina.YYYY-MM-DD.log`) while the UI ingests them.

### Notes / pitfalls

* Create the host folders **before** starting the stack—bind mounts replace the target directory view for that container.
* Permissions: containers run as root; host files will be root-owned. That’s normal for dev; if you need non-root host editing, use `sudo` or adjust file ownership after creation.
* Don’t mount broader paths like `/opt/xplg-service/ServletContainer`—stick to **logs subfolders** so you don’t shadow app binaries.

If you want, I can also add a tiny “sidecar tail” service that streams those host log folders to your terminal for quick debugging, but the above is enough to make the cluster visible in **System Status** and to persist & collect each node’s logs.









----

Here’s a clean, “batteries-included” compose that brings up a **3-node XPLG cluster** (UI, Master, Processor) with a **shared, persistent config** on your host. It’s set up so the System Status page will show **UI / MASTER / PROCESSOR**, and you can add each node’s own Tomcat (catalina) logs as Local sources.

---

# docker-compose.yml

```yaml
version: "3.8"

name: xplg-graal-cluster

networks:
  xplgnet: {}

services:

  xplg-master:
    image: xplghub/images:xplg.RC-Graal-10206
    container_name: xplg-master
    restart: unless-stopped
    networks: [xplgnet]
    # Master exposes to the internal network only
    expose:
      - "30303"   # HTTP
      - "30443"   # HTTPS
    environment:
      # JVM heap
      ram: "-Xmx2g"
      # IMPORTANT: full property string, not just a path
      sharedstorage: "xpolog.home.root.path=/home/data"
      # Role + inception
      profile: "-Dxpolog.uid.structure=master"
      xplg_inception_init_flow: "master"
      xplg_inceptions_post_init_flow_tasks: "clusterExternalConf"
    volumes:
      # Bind mount so you can poke around easily during dev
      - type: bind
        source: ${HOST_DATA:-./data}
        target: /home/data

  xplg-ui:
    image: xplghub/images:xplg.RC-Graal-10206
    container_name: xplg-ui
    restart: unless-stopped
    networks: [xplgnet]
    # Publish only the UI outward
    ports:
      - "30307:30303"  # HTTP
      - "30447:30443"  # HTTPS
    depends_on:
      - xplg-master
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=ui"
      xplg_inception_init_flow: "ui"
      xplg_inceptions_post_init_flow_tasks: "waitForFile"
    volumes:
      - type: bind
        source: ${HOST_DATA:-./data}
        target: /home/data

  xplg-processor:
    image: xplghub/images:xplg.RC-Graal-10206
    container_name: xplg-processor
    restart: unless-stopped
    networks: [xplgnet]
    expose:
      - "30303"
      - "30443"
    depends_on:
      - xplg-master
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=processor"
      xplg_inception_init_flow: "processor"
      xplg_inceptions_post_init_flow_tasks: "waitForFile"
    volumes:
      - type: bind
        source: ${HOST_DATA:-./data}
        target: /home/data
```

> Optional `.env` (same folder as the compose):
>
> ```
> HOST_DATA=./data
> ```

---

## One-time prep (do this once on the host)

```bash
mkdir -p ./data/plugins ./data/tmp
# Not strictly required, but avoids the plugins.xml warning you saw.
```

Bring it up:

```bash
docker compose up -d
```

Smoke tests:

```bash
# UI answering?
curl -sI http://127.0.0.1:30307/ | head -n1    # HTTP/1.1 200
curl -skI https://127.0.0.1:30447/ | head -n1  # HTTP/1.1 200

# Nodes got the right roles & shared storage?
docker exec -it xplg-master env    | egrep 'profile|inception|shared'
docker exec -it xplg-ui env        | egrep 'profile|inception|shared'
docker exec -it xplg-processor env | egrep 'profile|inception|shared'
```

Open the UI at `https://<host>:30447/` → **System Status**. You should now see separate sections for **UI**, **MASTER**, and **PROCESSOR**. If you don’t, check that all three containers can read the same bind mount:

```bash
# quick cross-check the shared /home/data
docker exec -it xplg-master sh -lc 'echo hello > /home/data/.probe'
docker exec -it xplg-ui     sh -lc 'cat /home/data/.probe'
docker exec -it xplg-processor sh -lc 'cat /home/data/.probe'
```

---

## Add the cluster’s own logs (per node)

In the XPLG UI:

1. **Add Data → Source Type → Local**
2. At the top of the wizard there’s a **Server** selector. Choose the node you want:

   * For UI logs: **Server = xplg-ui**
   * For Master logs: **Server = xplg-master**
   * For Processor logs: **Server = xplg-processor**
3. **Path** (per node):

   ```
   /opt/xplg-service/ServletContainer/logs/catalina.*.log
   ```
4. Save → **Collect Now** (or let the default policy pick them up).

Repeat for each node so you’ll have entries like:

* `Cluster Logs / UI / catalina`
* `Cluster Logs / Master / catalina`
* `Cluster Logs / Processor / catalina`

> Tip: you don’t need to mount these logs to the host; “Local” means “inside that container,” and the Server selector dispatches collection to the right node.

---

## Notes / gotchas

* You **do not** need to use the UI’s “Use external configuration directory” toggle with this compose. The env var
  `sharedstorage="xpolog.home.root.path=/home/data"` tells the product where the shared external config lives.
* If you see `plugins.xml … not a directory` again, just make sure `./data/plugins` exists on the host (already covered).
* Only the **UI** service publishes ports to the host. The **master** and **processor** talk to the UI over the internal `xplgnet` network.

If anything doesn’t show up in System Status after this, send me the short outputs of:

```
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker exec -it xplg-ui sh -lc 'curl -sI http://xplg-master:30303/ | head -n1'
docker logs --tail=200 xplg-ui | egrep -i 'deploy|plugins|waitForFile|ProtocolHandler|ERROR|SEVERE'
docker logs --tail=200 xplg-master | egrep -i 'clusterExternalConf|ProtocolHandler|ERROR|SEVERE'
```





----
Short answer: you’re 95% good. Two things jump out from those logs:

1. **Plugins webapp can’t mount**

```
IllegalArgumentException: The main resource set specified [/home/data/plugins] is not a directory or war file…
```

That means your UI is trying to deploy the **/plugins** context from the external storage, but the folder doesn’t exist yet.

2. **Ports were briefly in use**
   You had a startup where 30303/30443/8009 were already bound. You’ve since restarted and they’re fine (your `curl` shows 200 on both HTTP/HTTPS). Nothing to fix now—just avoid running another container mapping those ports.

3. **“Invalid character in method name …”**
   That’s just a client hitting HTTPS on the HTTP port (or vice-versa). Harmless.

---

## Fix the plugins error (and likely the “bounce back”)

Create the folder the UI expects and restart the UI container.

```bash
# on the host (adjust path if your ${HOST_DATA} is different)
mkdir -p ~/graal/data/plugins
chmod 755 ~/graal/data/plugins

# verify the UI sees it
docker exec -it xplg-ui sh -lc 'ls -ld /home/data /home/data/plugins'

# restart only the UI
docker compose restart ui

# watch deploy messages
docker exec -it xplg-ui sh -lc \
 'tail -n 200 /opt/xplg-service/ServletContainer/logs/catalina.*.log | egrep -i "plugins|SEVERE|ERROR|deploy"'
```

After that, clicking **PortX** should no longer bounce just because the context failed to deploy.
(If it still bounces, it’s probably because the PortX *content* isn’t installed yet—install it from **Marketplace**; the log error you showed would be gone though.)

---

## Sanity checks you can run

* Confirm the UI container really points its external storage to `/home/data`:

```bash
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' xplg-ui | grep -i sharedstorage
# should show: sharedstorage=xpolog.home.root.path=/home/data
```

* Ensure only **ui** publishes host ports:

```yaml
# compose: ui has:
ports: ["30307:30303", "30447:30443"]
# master/processor: only expose, not ports
```

* Verify listeners are answering (you already did):

```bash
curl -sI  http://127.0.0.1:30307/ | head
curl -skI https://127.0.0.1:30447/ | head
```

---

### What’s “okay” in your log, and what isn’t

* `The path attribute with value [/plugins] … has been ignored` — Tomcat 10/11 behavior; not fatal.
* `Jersey … should not consume any entity` — warnings; safe to ignore.
* `Address already in use` — only a problem if it repeats; you got a clean 200 now.
* `… [/home/data/plugins] is not a directory …` — **the one to fix**; create the folder.

Do that `~/graal/data/plugins` step + restart UI and you should be able to launch PortX and keep going with ingesting the Git Runner logs.




----

Good news: the cluster looks healthy. The “size = 0 Bytes” you’re seeing is almost certainly because those two “Worker-utc” assets were added as **Local** paths on the XPLG node, but the files actually live on a *different machine* (the runner VM). When an asset points to a path the collector can’t see, XPLG creates the asset but collects **0 bytes**.

Here’s how to fix that and also add the cluster’s own Tomcat logs as first-class sources.

---

# Fix the 0-byte “Worker-utc” logs

**Why it’s 0:** you added them under the “Local” folder (screenshot shows that). “Local” means “on the collector’s filesystem” (master/processor container). Your runner logs are on the **orchestrator VM**, not inside the containers.

**Do this:**

1. In XPLG UI → **ADD DATA → Get Data via Agent-Less → Remote Servers**
   Add your *runner VM* via SSH (host/ip + credentials). Test & Save.

2. Still under Remote Servers, **Browse** that host to:
   `/opt/runners/actions-runner-2/_diag/` and select files `Worker_*_utc.log`.

3. In **Collection Settings** (top collapsible row):

   * **Scan Method:** first scan = **from beginning** (or “collect last 20 MB on first scan”); subsequent = **tail**.
   * **End Of Line Representation:** **Line Fixed Prefix** and set it to `[`
     (This keeps multi-line “Job message” JSON attached to the previous line.)

4. Patterns (click **Verify** and add):

   * Normal lines:

     ```
     [{date:Date,yyyy-MM-dd HH:mm:ss'Z'} {priority:Level,TRACE;DEBUG;INFO;WARN;ERROR;FATAL} {text:Source}] {string:Message}
     ```
   * Lines that start with “Job message:” (optional second pattern):

     ```
     [{date:Date,yyyy-MM-dd HH:mm:ss'Z'} {priority:Level,TRACE;DEBUG;INFO;WARN;ERROR;FATAL} {text:Source}] Job message:{string:Message}
     ```

5. **Save**. You should now see **Last Collected** update and **Size** start growing.

> If you prefer not to use SSH, you can bind-mount the runner directory into the **master** container and then add a *Local* log that points at that mounted path. SSH is simpler.

---

# Ingest the cluster’s own Tomcat (Catalina) logs

Those logs live **inside the containers** (e.g., `xplg-ui:/opt/xplg-service/ServletContainer/logs`). To make them collectable and persistent, expose them to the **master** in two steps:

### A) Add bind mounts in your compose

Add these to `docker-compose.yml`:

```yaml
services:
  master:
    # …existing config…
    volumes:
      - ${HOST_DATA}:/home/data
      - /var/log/xplg/ui:/logs/ui:ro      # master can read UI logs
      - /var/log/xplg/master:/logs/master # (optional) master’s own logs

  ui:
    # …existing config…
    volumes:
      - ${HOST_DATA}:/home/data
      - /var/log/xplg/ui:/opt/xplg-service/ServletContainer/logs
```

Create the host dirs once:

```bash
sudo mkdir -p /var/log/xplg/ui /var/log/xplg/master
sudo chown -R $USER:$USER /var/log/xplg
docker compose up -d
```

Now **UI** writes its Tomcat logs to `/var/log/xplg/ui` on the host, and the **master** sees them at `/logs/ui`.

### B) Add them in XPLG as Local logs (master sees them)

UI → **ADD DATA → Add Log** (local):

* Path: `/logs/ui/catalina.*.log`
* **Scan Method:** from beginning (first time), then tail
* **End Of Line Representation:** **Automatic** (Tomcat logs are single-line records)

**Pattern** for Tomcat 11 lines like:

```
14-Sep-2025 11:34:10.575 INFO [Thread-0] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory [...]
```

Use:

```
{date:Date,dd-MMM-yyyy HH:mm:ss,SSS} {priority:Level,INFO;WARNING;SEVERE;ERROR} [{text:Thread}] {text:Logger} {string:Message}
```

Verify → Save. You’ll get neat columns: Date, Level, Thread, Logger, Message.

> You can repeat the same trick for **master** or **processor** containers by mounting their logs to host (e.g., `/var/log/xplg/master`) and exposing them to master at `/logs/master`.

---

# Quick checks if Size still shows 0

* **Wrong vantage point:** Local paths must exist inside the *collector’s* container. If the file is on another VM, use **Remote Servers (SSH)** or mount it into the collector.
* **Scan method:** If you picked “tail only,” and the file didn’t grow yet, you’ll see 0. Re-edit the log and set first scan to start from beginning (or last N MB).
* **Permissions:** From the collector container:

  ```bash
  docker exec -it xplg-master sh -lc 'ls -l /logs/ui | sed -n "1,20p"; head -n 3 /logs/ui/catalina.*.log'
  ```

  If that can’t read, fix host perms.
* **File pattern mismatch:** Ensure `Worker_*_utc.log` (underscores) matches your filenames exactly.

---

If you want, we can also add an **alert** (e.g., Level=SEVERE in catalina logs) and a simple dashboard; but first let’s get the sizes moving and fields populating.
l



----
# Add the cluster’s own Tomcat logs (catalina.\*.log)

Because the **master** does the collection in your cluster, give the master access to the UI container’s log files. Easiest dev-friendly way is a bind mount that both services share.

### A) Bind the UI logs to the host and into master

1. On the host:

   ```bash
   sudo mkdir -p /var/xplg/ui-logs
   sudo chown -R $USER:$USER /var/xplg/ui-logs
   ```

2. Edit your `docker-compose.yml`:

   **ui**:

   ```yaml
   services:
     ui:
       # ...existing...
       volumes:
         - ${HOST_DATA}:/home/data
         - /var/xplg/ui-logs:/opt/xplg-service/ServletContainer/logs:ro
   ```

   **master**:

   ```yaml
   services:
     master:
       # ...existing...
       volumes:
         - ${HOST_DATA}:/home/data
         - /var/xplg/ui-logs:/ui-logs:ro
   ```

3. Apply:

   ```bash
   docker compose up -d
   ```

4. Sanity check:

   ```bash
   docker exec -it xplg-master sh -lc 'ls -lh /ui-logs; head -n 3 /ui-logs/catalina.*.log'
   ```

5. In the UI (Folders & Logs → **Add Log**), point the path to:

   ```
   /ui-logs/catalina.*.log
   ```

   * **First run**: from beginning
   * **EOL**: Line Fixed Prefix `dd-MMM-` OR just leave automatic (Tomcat lines are single-line)
   * **Pattern** (works with your Tomcat 11 lines):

     ```
     {date:Date,dd-MMM-yyyy HH:mm:ss,SSS} {priority:Level,INFO;WARNING;SEVERE;ERROR} [{text:Thread}] {text:Logger} {string:Message}
     ```
   * **Verify** → **Save** → **Collect Now**.

### B) (Alternative) Ship logs without mounts

If you can’t mount:

* Use the built-in **HTTP/TCP listener** (right panel on *Add Data*) and stream from the UI container:

  ```bash
  docker exec -it xplg-ui sh -lc \
    'apk add -q busybox-extras || true; tail -F /opt/xplg-service/ServletContainer/logs/catalina.*.log | nc -w 1 <MASTER-IP> 1469'
  ```

  Then add a listener source in the UI (HTTP/TCP) and map fields with a Tomcat pattern.

---

## Quick “did it work?” signals

* Asset grid shows **Last Collected** updating and **Size > 0**.
* Log Viewer shows rows when range = **Last 1 hour**.
* `catalina.*.log` shows INFO “Starting ProtocolHandler…” once on restart, not repeated errors.

If you want, paste the result of **import sample from source path** for `Worker-utc` (what you actually see in that box), and I’ll give you the exact one-line pattern/EOL choices for that specific file so you’re not stuck on “Processing…”.



---

## 1) Prove the collector can read the file

* Open the **Worker-utc** asset → **… → Edit**.
* In **Raw Log Sample**, click **import sample from the source path**.

  * If you see lines: the file is reachable; go to step 2.
  * If you get an error: the path/perm is wrong—that alone explains “Processing…”.

## 2) Make the first run actually collect data

* **Collection Settings → Scan Method**: set first run to **from beginning** (or “last 50 MB”).
* **End Of Line Representation**: set **Line Fixed Prefix** to `[` (your records all start with `[`).
* Click **Verify** at the bottom—ensure the sample turns green with columns.
* **Save** → in the assets grid, **Actions → Collect Now**.
* In the Log Viewer, change time range from **All Time** → **Last 1 hour** and hit **FILTER**.

## 3) If the viewer still spins

* Click **Reset Filters** (top right of the grid header).
* Open your browser **DevTools → Network**, click the log again, and look for an XHR that’s pending or 500’ing. That tells us if the UI call fails.
* Server side quick check (UI node):

  ```bash
  docker exec -it xplg-ui sh -lc \
    'tail -n 200 /opt/xplg-service/ServletContainer/logs/catalina.*.log | egrep -i "SEVERE|ERROR|Exception" || true'
  ```

  If anything fires at the exact click time, we’ll know.

---





---
#### How to prove which node is collecting (no guesswork)

Great questions — and thanks for calling out the assumptions. Here’s a straight, verifiable answer.

## Do we “know” the **master** does the collection?

I don’t **know** it a-priori. I inferred it from the way you boot the cluster (the `profile=-Dxpolog.uid.structure=master|ui|processor` envs and the “inception” flags you pasted). Different XPLG builds can wire those roles differently.

### How to **prove** which node is collecting (no guesswork)

Pick a path that exists in only one container and try to add it as a **Local** source. If that source ingests, the node that can see that path is the collector.

Example:

```bash
# create a tiny test file only in the UI container
docker exec -it xplg-ui sh -lc 'echo "$(date) from UI" >> /tmp/who-collects.log'
```

Now in the UI: **Add Data → Source Type → Local → Path** `/tmp/who-collects.log` → first run: *from beginning* → Save → Collect Now.

* If data flows: whichever node performed the collection is the node that can see `/tmp/who-collects.log` (the UI container in this test).
* If it fails (file not found): the actual collector is a different container. Create that file in the **master** and try again:

```bash
docker exec -it xplg-master sh -lc 'echo "$(date) from MASTER" >> /tmp/who-collects.log'
```

You can double-check at OS level by seeing which container has the file open:

```bash
docker exec -it xplg-master sh -lc 'for f in /proc/*/fd/*; do readlink "$f" 2>/dev/null; done | grep who-collects.log || true'
docker exec -it xplg-ui     sh -lc 'for f in /proc/*/fd/*; do readlink "$f" 2>/dev/null; done | grep who-collects.log || true'
```

The one that prints the path is the process actually reading it.

> TL;DR: the collector is the node/container that physically reads the file. Prove it with a path that exists only in one container.

---

## What does **Add Data → Source Type → Local** mean in Docker?

“Local” always means **the local filesystem as seen by the process that reads the log**.

* In a single-node, bare-metal install, that’s the machine’s filesystem.
* In a container, that’s the **container’s** filesystem namespace.
  If you want to read host files, you must **mount** them into that container and use the **in-container** path in the wizard.

### Practical implications for your cluster

* If the **master** is your collector: mount host paths into **xplg-master**, then add the log as **Local** using the container path (e.g., host `/var/log/app.log` → mount to `/hostlogs/app.log` → put `/hostlogs/app.log` in the wizard).
* If the **UI** is the collector, do the same but mount into **xplg-ui**.
* If you want the UI to browse & configure but the master to read, mount the same host path into **both** containers and point the source at a path both can see (or assign the collector explicitly if your build exposes that setting).

### Alternatives when you can’t mount

Use a **listener** (TCP/HTTP/Syslog) or a **remote** method (SSH/WinRM). Then the path lives on the sender’s side, and the collector just receives the stream. This avoids any container filesystem concerns.

---

## Quick sanity checklist you can run now

1. Create `/tmp/who-collects.log` only in **one** container and add it as **Local** → observe which one ingests.
2. For any real host file you want to collect, mount it into that collector container and use that **container path** in the wizard.
3. After saving a source, always **Collect Now** and set time range to **Last 1 hour** to avoid a slow “All Time” query.

If you tell me which container succeeds in step (1), I’ll tailor the mounting and pathing for your exact compose so “Local” behaves exactly as you expect.


---

Totally makes sense. Here’s how to finish the probe and then wire **self-monitoring** cleanly.

## What I meant by “first run: from beginning → Save → Collect Now”

That’s just 3 UI clicks:

1. In **Add Data → Source Type → Local**, after you pick the path (e.g. `/tmp/who-collects.log`), click **COLLECTION SETTINGS** (the collapsible panel right under the file browser) → **Scan Method** → choose **Read from beginning (first time)**.
2. Click **Save** (top-right).
3. Go to **Folders & Logs**, find the new log, click its **⋯** menu → **Collect Now**.
   (This triggers an immediate read instead of waiting for the next poll.)

Tip: after that, open **Log Viewer**, set time range to **Last 1 hour** and refresh.

## Quick self-check with your probe file

Inside the container you used to add the log (sounds like `xplg-ui`):

```bash
docker exec -it xplg-ui sh -lc 'echo "$(date) UI proves it" >> /tmp/who-collects.log'
```

Then in the UI, open that log → you should see the new line. If yes, the node that can see `/tmp/who-collects.log` is the actual collector for **Local** sources you add from this screen.

---

## Let XPLG watch its **own** logs (self-monitoring)

Yes, it’s a good idea—part of the product’s value is monitoring itself.

### Option A (fast: read inside the UI container)

Add these as **Local** sources (same screen you’re on):

* Tomcat (web tier):

  * `/opt/xplg-service/ServletContainer/logs/catalina.*.log`
  * (optional) `/opt/xplg-service/ServletContainer/logs/localhost_access_log.*.txt`
* App/runtime:

  * `/opt/xplg-service/log/*.log`

For **Patterns**:

* Click **Use System Pattern** and search/select a Tomcat/Java pattern if offered.
* Or start simple: `{date,dd-MMM-yyyy HH:mm:ss.SSS} {priority,TRACE;DEBUG;INFO;WARN;WARNING;ERROR;SEVERE;FATAL} [{text:Thread}] {string:Logger} {string} {string}` and refine later.

Then **Save → Collect Now** for each.

> Caveat: those files live **inside** the container. They will disappear if you recreate the container unless you mount them (see Option B).

### Option B (better for persistence & multi-node)

Mount the log directories to the host in **docker-compose**, so logs survive restarts and can be read by whichever node you choose to be the collector.

Example for `xplg-ui` (do similar for `xplg-master` / `xplg-processor`):

```yaml
services:
  ui:
    volumes:
      # Persist Tomcat logs
      - ./cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs
      # Persist app logs
      - ./cluster-logs/ui/app:/opt/xplg-service/log
```

Bring the stack up, then add **Local** sources pointing to the same in-container paths as above (the app still writes to `/opt/...`, but those paths now map to host folders). Now you get persistence **and** easy access on the host (`./cluster-logs/...`).

---

## Why your “Local” tree says “Local” in the left nav

That “Local” you see under your **Folders & Logs** is just the **virtual folder** in XPLG’s repository (where entries appear), not the OS location. The **Source Type → Local** choice determines *how* the data is read (from the collector’s filesystem). So:

* **Local (source type)** = “read from the collector’s container filesystem (or its mounts)”.
* **Local (folder in the tree)** = just a folder name in XPLG’s UI.

---

## If a log opens and the viewer hangs on “Processing…”

Do a quick triage:

1. In the log’s **⋯** menu → **Properties** → check **Last Collected** and **Size**.
2. Click **Collect Now** once.
3. Append a line to the source file and verify it bumps **Last Collected**.
4. If size stays 0: re-open **Add Data → your log path** and set **Scan Method → Read from beginning (first time)**, save, collect now.
5. If it still hangs, reduce the viewer time range (e.g., **Last 15 min**) and refresh.

---

If you tell me which node you want as the long-term **collector**, I’ll give you the exact compose mounts + the minimal set of “Local” entries to monitor **all** node logs (UI, master, processor) with persistence.

----

##### Running the cluster manually


Totally—here’s a clean, copy-paste friendly flow to run the **three nodes manually with `docker run`**, keep each node’s logs **persisted on the host**, and then **verify** the cluster is healthy.

---

# 0) Host prep (dirs & network)

```bash
# where you'll run everything
mkdir -p ~/xplg-cluster
cd ~/xplg-cluster

# shared external config used by all nodes
mkdir -p ./data/plugins

# host-persisted logs per node (Tomcat + app)
mkdir -p ./cluster-logs/ui/{tomcat,app} \
         ./cluster-logs/master/{tomcat,app} \
         ./cluster-logs/processor/{tomcat,app}

# private docker network for the cluster
docker network create xplgnet
```

> Creating `./data/plugins` avoids the “plugins.xml main resource set … not a directory” warning.

---

# 1) Run MASTER

```bash
docker run -d --name xplg-master \
  --network xplgnet \
  --restart unless-stopped \
  -e ram="-Xmx2g" \
  -e sharedstorage="xpolog.home.root.path=/home/data" \
  -e profile="-Dxpolog.uid.structure=master" \
  -e xplg_inception_init_flow="master" \
  -e xplg_inceptions_post_init_flow_tasks="clusterExternalConf" \
  -v "$(pwd)/data:/home/data" \
  -v "$(pwd)/cluster-logs/master/tomcat:/opt/xplg-service/ServletContainer/logs" \
  -v "$(pwd)/cluster-logs/master/app:/opt/xplg-service/log" \
  xplghub/images:xplg.RC-Graal-10206



docker rm -f xplg-ui
docker run -d --name xplg-ui \
  --network xplgnet \
  -p 30307:30303 -p 30447:30443 \
  -e ram="-Xmx2g" \
  -e sharedstorage="xpolog.home.root.path=/home/data" \
  -e profile="-Dxpolog.uid.structure=ui" \
  -e xplg_inception_init_flow="ui" \
  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \
  -v "$(pwd)/data:/home/data" \
  -v "$(pwd)/cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs" \
  -v "$(pwd)/cluster-logs/ui/app:/opt/xplg-service/log" \
  --health-cmd 'wget -q --spider http://127.0.0.1:30303/ || curl -fsS http://127.0.0.1:30303/ >/dev/null' \
  --health-interval 15s --health-timeout 3s --health-retries 5 --health-start-period 60s \
  xplghub/images:xplg.RC-Graal-10206


docker run -d --name xplg-master \
  --network xplgnet \
  --restart unless-stopped \
  -e ram="-Xmx2g" \
  -e sharedstorage="xpolog.home.root.path=/home/data" \
  -e profile="-Dxpolog.uid.structure=master" \
  -e xplg_inception_init_flow="master" \
  -e xplg_inceptions_post_init_flow_tasks="clusterExternalConf" \
  -v "$(pwd)/data:/home/data" \
  -v "$(pwd)/cluster-logs/master/tomcat:/opt/xplg-service/ServletContainer/logs" \
  -v "$(pwd)/cluster-logs/master/app:/opt/xplg-service/log" \
  xplghub/images:xplg.RC-Graal-10206


```

> No ports published on MASTER (internal only).

---

# 2) Run UI

```bash
docker run -d --name xplg-ui \
  --network xplgnet \
  --restart unless-stopped \
  -p 30307:30303 -p 30447:30443 \
  -e ram="-Xmx2g" \
  -e sharedstorage="xpolog.home.root.path=/home/data" \
  -e profile="-Dxpolog.uid.structure=ui" \
  -e xplg_inception_init_flow="ui" \
  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \
  -v "$(pwd)/data:/home/data" \
  -v "$(pwd)/cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs" \
  -v "$(pwd)/cluster-logs/ui/app:/opt/xplg-service/log" \
  xplghub/images:xplg.RC-Graal-10206
```

> UI is the one you browse:
> `http://<host>:30307/` or `https://<host>:30447/`

---

# 3) Run PROCESSOR

```bash
docker run -d --name xplg-processor \
  --network xplgnet \
  --restart unless-stopped \
  -e ram="-Xmx2g" \
  -e sharedstorage="xpolog.home.root.path=/home/data" \
  -e profile="-Dxpolog.uid.structure=processor" \
  -e xplg_inception_init_flow="processor" \
  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \
  -v "$(pwd)/data:/home/data" \
  -v "$(pwd)/cluster-logs/processor/tomcat:/opt/xplg-service/ServletContainer/logs" \
  -v "$(pwd)/cluster-logs/processor/app:/opt/xplg-service/log" \
  xplghub/images:xplg.RC-Graal-10206
```

---

## 4) Quick sanity checks

**Containers & ports**

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

**UI responds**

```bash
curl -I  http://127.0.0.1:30307/ | sed -n '1,8p'
curl -kI https://127.0.0.1:30447/ | sed -n '1,8p'
# Expect HTTP/1.1 200 and the small bootstrap page that redirects to /logeye/root.html
```

**Tomcat actually started (per node)**

```bash
# UI
tail -n 120 ./cluster-logs/ui/tomcat/catalina.*.log | egrep -i \
'Starting ProtocolHandler|deploy(ing|ment of)|Server startup|SEVERE|ERROR'

# MASTER
tail -n 120 ./cluster-logs/master/tomcat/catalina.*.log | egrep -i \
'Starting ProtocolHandler|deploy(ing|ment of)|Server startup|SEVERE|ERROR'

# PROCESSOR
tail -n 120 ./cluster-logs/processor/tomcat/catalina.*.log | egrep -i \
'Starting ProtocolHandler|deploy(ing|ment of)|Server startup|SEVERE|ERROR'
```

**Shared config present**

```bash
ls -al ./data
# You should see files/dirs created by the master after a minute or so.
```

**Inside-container spot check (optional)**

```bash
docker exec -it xplg-ui sh -lc 'ls -l /home/data; ls -l /opt/xplg-service/ServletContainer/logs | head'
```

---

## 5) Verify the cluster in the product UI

* Open the UI: `http://<host>:30307/`
* Go to **System Status** (gear icon → System Status).
* After 30–90 seconds you should see **UI**, **MASTER**, and **PROCESSOR** sections.

  * If you only see one of them, give it another minute; then check the other nodes’ catalina logs for connection errors.

---

## 6) Add the cluster’s own logs as sources (one-time)

Because we bind-mounted logs, you can add them as **Local** on the node that owns them:

1. **ADD DATA → Source Type → Local**
2. Top-right “Server” dropdown → pick the node (e.g., **UI**).
3. **Path** examples:

   * Tomcat: `/opt/xplg-service/ServletContainer/logs/catalina.*.log`
   * App: `/opt/xplg-service/log/*.log`
4. In **Collection Settings**, choose:

   * **File rotation policy**: *Cycling* (typical for catalina logs)
   * **From the last**: e.g., *7 days* (or what you need)
5. **Save** → then **Collect Now** on the log to pull the current files.
6. Repeat for **MASTER** and **PROCESSOR** nodes (select each server, same paths).

> Since those in-container paths are bind-mounted, they also live on your host under `./cluster-logs/<node>/...`, so you can tail them locally while XpoLog ingests them.

---

## 7) Common issues & fixes

* **“plugins.xml … not a directory”**: ensure `./data/plugins` exists before starting (you did).
* **BindException: Address already in use** on 30303/30443:
  Only the **UI** publishes ports. Stop any stray containers using those ports:

  ```bash
  sudo ss -ltnp | egrep '30307|30447'
  docker ps | egrep 'xplg-ui|30307|30447'
  ```
* **Shutdown socket 8095 in use**: rare in separate containers. If you see it, just restart that container:

  ```bash
  docker restart xplg-ui   # or xplg-master / xplg-processor
  ```

---

## 8) Clean stop / start

```bash
# stop
docker stop xplg-ui xplg-master xplg-processor

# start again (preserves data/logs)
docker start xplg-master xplg-ui xplg-processor
```

That’s it—this mirrors the compose setup you have, but with explicit `docker run`s, and it sets you up to (a) see **UI/MASTER/PROCESSOR** on System Status and (b) ingest each node’s own logs while keeping them persisted on the host.




---
##### Bottom


----

```bash
docker run -d --name xplg-graal \\n  -p 30306:30303 \\n  -p 30446:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n  --restart unless-stopped \\n  xplghub/images:xplg.RC-Graal-10206
 3801  docker ps
 3802  docker run  --platform linux/amd64 \\n  -d  --name xplg-graal \\n  -p 30307:30303 \\n  -p 30447:30443 \\n  -p 38447:8443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v $(pwd)$:/home/xplg \\n  --restart unless-stopped \\n  "$IMG" 
 3803  docker run  -d --name xplg-graal \\n  -p 30307:30303 \\n  -p 30447:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n  --restart unless-stopped \\n  xplghub/images:xplg.RC-Graal-10206 
 3804  docker rm -f /xplg-graal  2>/dev/null || true 
 3805  docker run  -d --name xplg-graal \\n  -p 30307:30303 \\n  -p 30447:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n  --restart unless-stopped \\n  xplghub/images:xplg.RC-Graal-10206 
 3806  docker ps
 3807  docker restart xplg-graal
 3808  docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"\n 
 3809  docker exec -it xplg-graal sh -lc 'sed -n "1,10p" /opt/xplg-service/xpologInit.prop' 
 3810  docker exec -it xplg-graal sh -lc '\n     set -e\n     ls -ld /home/xplg/targetExtConf\n     touch /home/xplg/targetExtConf/_write_probe\n     ls -al /home/xplg/targetExtConf | head -n 30\n   ' 
 3811  docker logs --since=5m xplg-graal | tail -n +1 | sed -n '1,200p'\n 
 3812  docker logs --since=10m xplg-graal | tail -n +1 | sed -n '1,200p'\n 
 3813  docker logs --since=10m xplg-graal \n 
 3814  docker logs xplg-graal\n 




 ####

 docker run  -d --name xplg-graal \
   -p 30307:30303 \
   -p 30447:30443 \
   -e sharedstorage=/home/xplg/targetExtConf \
   -e ram='-Xmx4g' \
   -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \
   --restart unless-stopped \
    xplghub/images:xplg.RC-Graal-10206
```



#### Tusday Sep 16 

This looks good! From your screenshots and tails:

* The **UI node’s Tomcat logs** are being ingested and indexed (49.5 KB, “Tomcat 5.0.28” AppTag, and you can see the `Starting ProtocolHandler` / `Server startup` lines).
* Your “Zigi Testing” sample log is also parsed and displayed with fields—so collection + indexing are working.

A couple of quick follow-ups to finish the setup and make it dependable:

### 1) Collect Master & Processor logs now

You already created the folders (“Master”, “Processor”). To force an initial backfill:

1. Open each log (Master → `catalina.*.log`, Processor → `catalina.*.log`).
2. Use the kebab menu (…) → **Collect Now** (or **Recollect from beginning** if you want full history).
3. Verify size grows and “Last Collected” updates.

### 2) Health/port checks without `wget`

Alpine images typically don’t have `wget`. Use `curl`:

```bash
# UI container
docker exec -it xplg-ui sh -lc '
  set -e
  echo -n "HTTP : "; curl -fsS http://127.0.0.1:30303/    > /dev/null && echo OK || echo FAIL
  echo -n "HTTPS: "; curl -kfsS https://127.0.0.1:30443/  > /dev/null && echo OK || echo FAIL
'
```

(You can run the same pattern inside `xplg-master` and `xplg-processor` to confirm their embedded Tomcat ports are up.)

### 3) Make multi-line application logs readable

For logs like your worker file that have big JSON-ish blocks (`"fileTable":[ ... ]`):

* When adding the log, expand **Collection Settings → End of line representation** and set **Line Fixed Prefix** to a token that starts each new record.
  For your runner lines that begin with a bracketed timestamp, using just `[` works well. XpoLog will treat lines that **don’t** start with `[` as continuations of the previous record, which stitches multi-line messages correctly.

### 4) Persist the cluster’s own logs (you already mounted them)

You bound:

```
./cluster-logs/ui/tomcat       -> /opt/xplg-service/ServletContainer/logs
./cluster-logs/master/tomcat   -> /opt/xplg-service/ServletContainer/logs
./cluster-logs/processor/tomcat-> /opt/xplg-service/ServletContainer/logs
```

That’s perfect. Continue to add **Local** sources pointing to the **in-container** paths (e.g., `/opt/xplg-service/ServletContainer/logs/catalina.*.log`). Because of your bind mounts, the data persists on the host and survives restarts.

### 5) Nice-to-have monitors for the cluster

Create a couple of searches/monitors against those Tomcat logs:

* `Level:SEVERE OR ERROR`
* `message:"Address already in use"` (you hit this earlier)
* `message:"Failed to create server shutdown socket"`
* `message:"Protocol handler initialization failed"`
* `message:"OutOfMemoryError"`

Attach email/webhook alerts and you’ll know quickly if a node gets into a bad state.

### 6) Quick sanity checklist

* **System Status** page now shows 3 nodes working? (UI, Master, Processor).
* Each node’s **catalina log** shows the three “Starting ProtocolHandler” lines and **Server startup in … ms** (your tails confirm this).
* Your **Local → xplg-cluster** folders show **non-zero sizes** and update when you hit **Collect Now**.

If you want, I can sketch a minimal “Tomcat-catalina” parse pattern to split fields (date/level/thread/class/message) for easier filtering—just say the word and I’ll drop it in.


----

Got it—thanks for the screenshot. On your build, the **Local** page doesn’t have a node/server picker. “Local” always means “the filesystem inside the **UI** container.” That’s why you can browse only the UI node’s own `/opt/xplg-service/...` paths.

You have two clean ways to add **Master/Processor** logs:

## Option A — Mount those log folders into the UI container (then use “Local”)

Add read-only mounts so the UI container can see the other nodes’ bind-mounted logs.

### If you’re on docker-compose

In the **ui** service, add mounts like these (keep your existing ones too):

```yaml
services:
  ui:
    volumes:
      # UI node (already have this)
      - ./cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs
      - ./cluster-logs/ui/app:/opt/xplg-service/log

      # Make other nodes' logs visible inside the UI container (read-only)
      - ./cluster-logs/master/tomcat:/mnt/cluster/master/tomcat:ro
      - ./cluster-logs/master/app:/mnt/cluster/master/app:ro
      - ./cluster-logs/processor/tomcat:/mnt/cluster/processor/tomcat:ro
      - ./cluster-logs/processor/app:/mnt/cluster/processor/app:ro
```

Then:

```bash
docker compose up -d --force-recreate ui
```

In the UI:

* **Add Data → Local → Browse** to:

  * `/mnt/cluster/master/tomcat/` (e.g., `catalina.*.log`)
  * `/mnt/cluster/processor/tomcat/`
  * (and the optional `/app` folders)
* In **Collection Settings**: set rotation to **Cycling**, “From the last: All time” (or your window), then **Save** → **Collect Now**.

### If you started containers manually

Recreate **only the UI** container with extra `-v` flags:

```bash
docker rm -f xplg-ui
docker run -d --name xplg-ui \
  --network xplgnet \
  -p 30307:30303 -p 30447:30443 \
  -e ram='-Xmx2g' \
  -e profile='-Dxpolog.uid.structure=ui' \
  -e XPLG_INCEPTION_PARAMS='xplg_inception_init_flow=ui
xplg_inceptions_post_init_flow_tasks=waitForFile' \
  -v $PWD/cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs \
  -v $PWD/cluster-logs/ui/app:/opt/xplg-service/log \
  -v $PWD/cluster-logs/master/tomcat:/mnt/cluster/master/tomcat:ro \
  -v $PWD/cluster-logs/master/app:/mnt/cluster/master/app:ro \
  -v $PWD/cluster-logs/processor/tomcat:/mnt/cluster/processor/tomcat:ro \
  -v $PWD/cluster-logs/processor/app:/mnt/cluster/processor/app:ro \
  xplghub/images:xplg.RC-Graal-10206
```

Then add paths in the UI exactly like above.

## Option B — Use **Over SSH** to the Docker host (no extra mounts)

Because you already bind-mounted each node’s logs to the host (`./cluster-logs/...`), you can have XPLG read them via SFTP:

1. **Add Data → Over SSH**.
2. Host Address: your Docker host (e.g., `10.0.0.85`), Port `22`, provide a user/key that can read the folders.
3. After **Verify**, set the path(s) to:

   * `/home/xplg/xplg-cluster/cluster-logs/master/tomcat/…`
   * `/home/xplg/xplg-cluster/cluster-logs/processor/tomcat/…`
   * (and `/app` if desired)
4. Save → **Collect Now**.

> “Remote XpoLog / Local XpoLog” tiles are for pulling data from another XpoLog server, not files on your nodes—so they’re not what you want here.

---

### Quick sanity checks

* Host side: confirm files are growing and readable

  ```bash
  ls -lh ./cluster-logs/{ui,master,processor}/tomcat
  tail -f ./cluster-logs/master/tomcat/catalina.*.log
  ```
* Inside UI container (Option A):

  ```bash
  docker exec -it xplg-ui sh -lc 'ls -lh /mnt/cluster/master/tomcat | head'
  ```
* After adding sources: open **Folders & Logs → your items → Log Viewer** and check the latest timestamps / sizes increase.

This way you can keep using “Local” with a clean browse experience (Option A), or avoid any container changes and read via SSH from the host (Option B).

----

docker exec -it xplg-ui sh -lc 'ls -l /home/xplg/xplg-cluster/cluster-logs/master/tomcat/ && \
  tail -n 3 /home/xplg/xplg-cluster/cluster-logs/master/tomcat/catalina.2025-09-16.log'

-----


Yep—let’s debug this cleanly. Two key points first:

* The “Folders & Logs” tree you build in the UI is **virtual metadata**, not OS folders. It’s saved by XPLG under its **home** (the path you pass via `xpolog.home.root.path`, i.e. `/home/data` inside each container).
* If those objects disappear after `down/up`, either (a) the container didn’t start with the **same** host path bound to `/home/data`, or (b) the app couldn’t write there.

Below is a quick, copy-pasteable trace to prove where things are saved and why they “vanish.”

---

## 1) Verify each node really mounts the same host path to `/home/data`

```bash
# On the host
docker inspect xplg-ui --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}'
docker inspect xplg-master --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}'
docker inspect xplg-processor --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

You should see your **absolute** host path (e.g. `/home/xplg/xplg-cluster/data`) mapped to **`/home/data`** for all three.
If any service shows a different source (or a relative `./data` from a different working dir), that explains the “reset.”

---

## 2) Confirm the app is actually using `/home/data`

```bash
# Inside UI container
docker compose exec xplg-ui sh -lc '
  echo "JAVA FLAGS:";
  for p in /proc/[0-9]*; do
    c=$(tr -d "\0" < $p/cmdline 2>/dev/null); case "$c" in *java*) echo "$c";; esac
  done | tr " " "\n" | egrep -i "xpolog\.home|xpolog\.uid|xpolog"
  echo; echo "HOME SNAPSHOT:"; ls -al /home/data; du -h -d1 /home/data 2>/dev/null || true
'
```

You want to see a `-Dxpolog.home.root.path=/home/data` (or equivalent) **in the Java cmdline** and a non-empty `/home/data` directory.

---

## 3) Find *where* the UI stores the virtual tree (and prove persistence)

Create something distinctive in the UI (e.g., a folder named `PERSIST_MARKER_123`), hit **Save**, then:

```bash
docker compose exec xplg-ui sh -lc '
  echo "Searching metadata for marker…";
  grep -R --line-number --fixed-strings "PERSIST_MARKER_123" /home/data 2>/dev/null | head || true
'
```

* If you get matches, great—that’s the on-disk metadata.
* Now do `docker compose down` (not `down -v`) and `docker compose up -d`, then run the same `grep` again.

  * **Still matches?** Your data is persisted correctly; if the UI doesn’t show it, we’ll look for a UI cache/health issue.
  * **No matches now?** Then `/home/data` wasn’t the same location on this new up (most often caused by running compose from a different directory when using `./data`).

> You already switched to **absolute paths** in your compose, which is the right fix. Using relative `./data` from different working directories is the #1 reason people “lose” definitions after a restart.

---

## 4) Permission sanity

Make sure the process can write to the host path:

```bash
# On the host
ls -ld /home/xplg/xplg-cluster/data
# Inside the UI container
docker compose exec xplg-ui sh -lc 'id; touch /home/data/.write_test && ls -l /home/data/.write_test'
```

If `touch` fails, fix ownership on the host:

```bash
sudo chown -R 1000:1000 /home/xplg/xplg-cluster/data   # or 0:0 if the app runs as root
```

---

## 5) Why “folders aren’t showing on the host”

That’s expected—the UI “folders” are **not** filesystem directories. They are metadata files under `/home/data` (you’ll locate them with the marker search in step 3). Your **actual log files** are whatever you point to (e.g., the Tomcat logs under your bind mounts).

---

## 6) Quick health check after an up

```bash
# UI reachable?
curl -skI https://127.0.0.1:30447/ | head

# App writing metadata?
docker compose exec xplg-ui sh -lc '
  egrep -i "error|exception|denied|write|permission" /opt/xplg-service/log/* 2>/dev/null | tail -n 200 || true
'
```

---

### Likely outcomes & fixes

* **Different `/home/data` between runs** → switch to absolute host paths (you did), and always `docker compose up` from the same directory, or set `env_file`/`.env` pinned paths.
* **Wrong/missing Java flag** (no `xpolog.home.root.path` in cmdline) → the image didn’t pick up `sharedstorage`; double-check env var spelling and that your inception flow isn’t overriding the home.
* **Permissions** → fix ownership/permissions on the host path.
* **You expect OS folders to appear** → they won’t; the virtual tree is metadata under `/home/data`.

Run the commands above and tell me which step, if any, deviates. From there we can zero in on the exact reason your UI objects seem to “reset.”

----


Great—thanks for the paste. Two things jump out in that log:

1. `Could not find or load main class xpolog.eye.tools.loaders.AntLoader`
2. ESAPI warnings that then say “SUCCESSFULLY LOADED … via the CLASSPATH”.

### What those mean

* **AntLoader** — that’s a legacy helper the image tries to invoke during startup/maintenance tasks. In the Graal builds it often isn’t on the classpath. It’s noisy, but **it doesn’t block Tomcat or the app**. If the UI is up and you can browse pages, this isn’t the cause of data loss or missing logs.
* **ESAPI** — first tries to load config from files, can’t find them, then **successfully falls back** to the embedded defaults. Again, **not fatal**.

Nothing in that snippet shows “permission denied”, “read failed”, or similar—so the interesting question is still *where is the UI writing its configuration (virtual folders, data sources), and is that really persisted across restarts?*

Below is a tight probe sequence you can run to prove persistence and find the files that back your UI objects.

---

## A. Make sure all nodes really mount the same persisted home

```bash
# On the host
docker inspect xplg-ui --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}'
docker inspect xplg-master --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}'
docker inspect xplg-processor --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

You should see your **absolute** host path (e.g. `/home/xplg/xplg-cluster/data`) → **`/home/data`** in **all three** containers. If any service has a different source, that explains “resets”.

---

## B. Confirm the app is *using* `/home/data`

```bash
docker compose exec xplg-ui sh -lc '
  echo "JAVA FLAGS:";
  for p in /proc/[0-9]*; do
    c=$(tr -d "\0" < $p/cmdline 2>/dev/null); case "$c" in *java*) echo "$c";; esac
  done | tr " " "\n" | egrep -i "xpolog\.home|xpolog\.uid|xpolog"

  echo; echo "HOME SNAPSHOT (/home/data):";
  ls -al /home/data | sed -n "1,80p"
'
```

Look for `-Dxpolog.home.root.path=/home/data` in the Java flags and that `/home/data` isn’t empty.

---

## C. Prove where the UI saves the **virtual folders/data-sources**

1. In the UI, create a folder named **PERSIST\_MARKER\_123** (anything distinctive) and click **Save**.
2. Search for that marker on disk:

```bash
docker compose exec xplg-ui sh -lc '
  grep -R --line-number --fixed-strings "PERSIST_MARKER_123" /home/data 2>/dev/null | head || true
'
```

* If you get hits, that’s your metadata store.
* Now do:

  ```bash
  docker compose down    # (no -v)
  docker compose up -d
  docker compose exec xplg-ui sh -lc '
    grep -R --line-number --fixed-strings "PERSIST_MARKER_123" /home/data 2>/dev/null | head || true
  '
  ```

  If the hits are still there but the UI doesn’t show it, it’s a UI/initializer issue (not persistence). If the hits **disappear**, the container wasn’t started with the same `/home/data` host path.

> Note: the “Folders & Logs” tree is **metadata**, not OS folders. Don’t expect new directories to appear on the host; expect updates under files in `/home/data`.

---

## D. Quick permission check (just in case)

```bash
# Host
ls -ld /home/xplg/xplg-cluster/data

# Inside UI
docker compose exec xplg-ui sh -lc 'id; touch /home/data/.write_test && ls -l /home/data/.write_test'
```

If `touch` fails, fix host perms (e.g., `sudo chown -R 1000:1000 /home/xplg/xplg-cluster/data`).

---

## E. About adding the cluster’s own logs

You already mounted:

* UI Tomcat → `/opt/xplg-service/ServletContainer/logs`
* Master Tomcat → `/home/xplg/xplg-cluster/cluster-logs/master/tomcat` (bind into UI at the same path)
* Processor Tomcat → `/home/xplg/xplg-cluster/cluster-logs/processor/tomcat`

So, in **Add Data → Local → Browse**, you should be able to select, for example:

```
/home/xplg/xplg-cluster/cluster-logs/master/tomcat/catalina.2025-09-16.log
```

If the preview still says **“NO SOURCE LOG DATA”**:

* It’s a preview failure, not necessarily ingestion failure. Click **Add Log → Save**, then open the log in **Log Viewer**.
* Or step up one folder and click **Scan Directory**, then set **File Filters → Include the last: 1 file** and **With size > 1 bytes**, then **Save**.

---

## F. The AntLoader noise

Those console errors are common on this build and not related to persistence or file collection. If they’re distracting, we can route or lower that logger later, but I wouldn’t chase them unless you see functional issues.

---

Run sections **A–C** and tell me what you see (especially whether the marker survives and whether `/home/data` is the same host path on all three). That will pinpoint whether we’re dealing with a mount/persistence mismatch or just a UI-side preview quirk.


----

Great question — this is one of those “what the UI shows vs. what exists on disk” things.

### What my sentence meant

In XpoLog/PortX there are **two very different kinds of “things”**:

1. **Source files/directories (real OS paths).**
   Example: `/opt/xplg-service/ServletContainer/logs/catalina.2025-09-16.log`.
   These are real files on the container filesystem (and, via your bind mounts, on the host under `./cluster-logs/...`). When you add a *Local* source, you’re pointing the app to these real files.

2. **UI objects / “virtual folders” / data-source definitions (metadata).**
   The tree you build in **Folders & Logs** (folders, saved searches, dashboards, the logical “log” objects themselves) are **not OS directories**. They’re **records** saved in the app’s home (your `xpolog.home.root.path`, i.e. `/home/data`) inside one or more **metadata files / small databases**.
   So when you create “Folder A / MyAppLog” in the UI, the app **doesn’t mkdir** anything on the host. Instead, it updates one or more files under `/home/data` (JSON/XML/db/etc., depending on build).

Because of that, you **won’t see a new directory** appear in `./data` on the host when you make a new UI folder. What you *will* see is that **some file in `/home/data` changes** (timestamp/size/content).

### So how do you verify it’s actually written to disk (and survives up/down)?

1. **Make a distinctive marker in the UI**
   Create a folder or source named: `PERSIST_MARKER_123`.

2. **Find where it was written (inside the UI container):**

   ```bash
   docker compose exec xplg-ui sh -lc '
     grep -R --line-number --fixed-strings "PERSIST_MARKER_123" /home/data 2>/dev/null | head || true
   '
   ```

   * If you get matches, that’s the **metadata file** that stores your tree.
   * If you don’t, the build may store it in a binary DB. Still OK — see step 3.

3. **See what actually changed:**

   ```bash
   docker compose exec xplg-ui sh -lc '
     find /home/data -type f -printf "%T@ %s %p\n" | sort -nr | head -20
   '
   ```

   Look for files with very recent timestamps; those are where the metadata lives.

4. **Prove persistence across restarts:**

   ```bash
   docker compose down     # (no -v)
   docker compose up -d

   # After it’s up:
   docker compose exec xplg-ui sh -lc '
     grep -R --line-number --fixed-strings "PERSIST_MARKER_123" /home/data 2>/dev/null | head || true
   '
   ```

   If the marker is still found, the UI objects **persisted** (because `/home/data` is bind-mounted to your host `./data`). If it’s gone, something’s off with the mount or the home path.

### TL;DR

* **Your logs** (Tomcat/app) are real files and you *do* see them on the host in `./cluster-logs/...` because you bind-mounted those paths.
* **Your UI structure** (folders/log definitions) is **metadata**, stored **inside files under `/home/data`**, not as OS directories. That’s why you don’t see new directories on the host — you see **updated files** under `./data` instead.

If you want, I can give you a one-liner that snapshots `/home/data` before and after a UI change to highlight exactly which files were modified.


-----

##### Checking the resources 

Got it — let’s triage the slowness fast and fact-find what’s hurting. Below are **ready-to-run checks** plus **quick mitigations**. Run them as-is and tell me what stands out.

# 1) Is the host or any container resource-starved?

**Host pressure (CPU, run queue, swap):**

```bash
# CPU, run queue, swap activity (look at r, us, sy, id, wa, si/so)
vmstat 1 10

# Per-CPU utilization; press q to quit
mpstat -P ALL 1 5 || true   # if sysstat not installed, skip

# Top offenders by CPU & MEM
top -b -n1 | head -n 30
```

**Docker containers live stats:**

```bash
docker stats --no-stream
```

If you see one of the xplg containers pegged (CPU >> 100% or MEM \~ close to the `-Xmx2g`), that’s our first lead.

# 2) Is I/O the bottleneck? (bind mounts can be slow if scanned recursively)

**Host disk pressure:**

```bash
iostat -xz 1 5 || cat /proc/diskstats | tail -n +1
```

Watch `util` and `await/svctm`. Very high `util` (>80–90%) or long `await` = I/O bound.

**Per-container I/O & file descriptor counts:**

```bash
for c in xplg-ui xplg-master xplg-processor; do
  echo "== $c =="
  docker exec $c sh -lc '
    pid=$(busybox ps | awk "/java/ {print \$1; exit}");
    echo "PID: $pid"
    echo "-- /proc mem & threads --"
    awk "/VmRSS|VmSize|Threads/ {print}" /proc/$pid/status
    echo "-- /proc I/O counters --"
    cat /proc/$pid/io 2>/dev/null || true
    echo "-- Open file descriptors --"
    ls /proc/$pid/fd 2>/dev/null | wc -l
  '
done
```

Huge `read_bytes` growing fast + tons of FDs often means it’s scanning lots of files.

> ⚠️ If you added a **Local** source that points at a **directory root** (e.g. `/home/xplg/xplg-cluster/cluster-logs/`), the UI node can end up **recursively scanning everything**. Prefer **individual files** or very tight globs like `/home/.../tomcat/catalina.*.log`.

# 3) Is the JVM spending time in GC or blocked threads?

**Quick JVM signal checks without JDK tools:**

```bash
for c in xplg-ui xplg-master xplg-processor; do
  echo "== $c =="
  docker exec $c sh -lc '
    pid=$(busybox ps | awk "/java/ {print \$1; exit}");
    echo "-- Threads --"
    ls /proc/$pid/task | wc -l
    echo "-- Blocked threads snapshot --"
    grep -H "blocked" -n /proc/$pid/stack 2>/dev/null | head || true
  '
done
```

**Enable GC logs (persistent) for next restart**
Add this to each service’s `environment:` in compose (keeps your `-Xmx2g`):

```yaml
  JAVA_TOOL_OPTIONS: >-
    -Xlog:gc*,safepoint=info:file=/opt/xplg-service/log/gc.log:time,uptime,level,tags
```

Then:

```bash
docker compose up -d
# After a few minutes:
docker compose exec xplg-ui sh -lc 'tail -n 80 /opt/xplg-service/log/gc.log || true'
```

Frequent long GCs = raise heap or tune G1 (e.g., `-XX:MaxGCPauseMillis=200`).

# 4) Is Tomcat serving slowly or is it the browser/app layer?

**Measure server response times for a static page vs. a dynamic API:**

```bash
# Static (should be quick)
curl -s -o /dev/null -w 'static total:%{time_total} ttfb:%{time_starttransfer}\n' \
  http://127.0.0.1:30307/logeye/root.html

# Try a heavier page the UI loads (if known). Otherwise re-test root with HTTPS:
curl -sk -o /dev/null -w 'tls total:%{time_total} ttfb:%{time_starttransfer}\n' \
  https://127.0.0.1:30447/logeye/root.html
```

If static is fast but the app *pages* crawl, it’s likely **backend indexing/queries** rather than Tomcat/network.

# 5) Look for noisy errors / tight loops in app logs

You already saw:

```
Could not find or load main class xpolog.eye.tools.loaders.AntLoader
```

If this repeats a lot, it’s wasted CPU. Let’s confirm rate:

```bash
for c in xplg-ui xplg-master xplg-processor; do
  echo "== $c =="
  docker exec $c sh -lc '
    log=/opt/xplg-service/log/XpoLogConsole.log
    test -s "$log" || { echo "no log"; exit; }
    echo "AntLoader errors in last 200 lines:"
    tail -n 200 "$log" | egrep -c "AntLoader|ClassNotFoundException" || true
    echo "Recent hot messages:"
    tail -n 200 "$log" | egrep -i "exception|error|denied|timeout|index|parse" || true
  '
done
```

If counts are high: that “tool” keeps failing. We can:

* ensure you don’t have a *post-init* task that triggers it repeatedly, or
* set `XPLG_INCEPTION_PARAMS` to only the flows you need (which you have), and
* check whether a **Local** source definition is pointing at a missing path (causing retries).

# 6) Check that the UI isn’t thrashing the shared config area

`xpolog.home.root.path=/home/data` is bind-mounted. If it sits on slow storage or inside a **very chatty** directory, you’ll feel it. Watch what changes most:

```bash
docker compose exec xplg-ui sh -lc '
  echo "Last writes under /home/data:"
  find /home/data -type f -printf "%T@ %s %p\n" | sort -nr | head -20
'
```

If you see a few files being updated **every second**, we can relocate `/home/data` to a **Docker named volume** (faster than overlay bind in many cases) while keeping log bind mounts as-is.

# 7) Quick mitigations you can apply right now

* **Narrow your Local sources** to *specific files*, not directories.
  Example (inside the UI when adding sources):
  `/home/xplg/xplg-cluster/cluster-logs/master/tomcat/catalina.2025-09-16.log`
  and not `/home/xplg/xplg-cluster/cluster-logs/master/`.

* **Cap the backlog read** when testing large files: choose “From **now**” (or last N MB) on first run so it doesn’t churn through historic content.

* **Bump heap on UI** if `docker stats` shows MEM near limit or GC log shows pressure:
  `ram: "-Xmx3g"` (only if host has room).

* **Consider moving `/home/data` to a named volume** for speed:

  ```yaml
  volumes:
    xplgdata-ui:
    xplgdata-master:
    xplgdata-proc:

  services:
    xplg-ui:
      volumes:
        - xplgdata-ui:/home/data
        - ./cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs
        - ./cluster-logs/ui/app:/opt/xplg-service/log
    # same idea for master/processor
  ```

  This keeps metadata on a fast Docker volume (less overlay2 penalty) while logs stay on host via bind mounts.

* **Ensure you didn’t accidentally add the entire cluster-logs root as a Local source.** If you did, delete that source so it stops recursively indexing.

---

If you paste back the outputs from sections **1–3** (docker stats, vmstat/mpstat, per-container `/proc/$pid/io` and Threads), I’ll pinpoint whether it’s CPU, GC, or I/O — and we’ll apply the right tweak immediately.

---


Thanks for the dumps — super helpful. Here’s what they say and what to fix next.

## What your numbers say

* **Host is idle**: CPU \~99% idle, iowait \~0, load \~0.2. Disk util low. So the “slow” feeling isn’t system pressure.

* **HTTP server is quick**: `root.html` TTFB \~5ms (HTTP) / \~23ms (HTTPS). So it’s not Tomcat/network.

* **App-level errors exist**: repeated

  ```
  Error: Could not find or load main class xpolog.eye.tools.loaders.AntLoader
  ```

  across **all three nodes**. Annoying, but not pegging the box.

* **Key finding**: the shared config area `/home/data` is basically empty:

  ```
  Last writes under /home/data:
  ... /home/data/.write_test
  ```

  That means the **cluster external config flow never ran**. When that flow doesn’t run:

  * cluster wiring (roles, discovery, “Remote Servers”, etc.) is incomplete,
  * UI “Add Log” may spin/fail (can’t persist the new source),
  * anything you created in the UI will vanish on container recreate (it wrote inside the container FS, not to the bind).

## Why: wrong env var for the inception flows

In your current compose, you set:

```
xplg_inception_init_flow: "master"
xplg_inceptions_post_init_flow_tasks: "clusterExternalConf"
```

…but the image expects these as **lines inside one variable** called **`XPLG_INCEPTION_PARAMS`**. When they’re split, the startup scripts ignore them → no `/home/data` population → no persisted cluster state.

## Fix (surgical patch)

Change each service’s `environment` to use `XPLG_INCEPTION_PARAMS` exactly like this:

```yaml
services:
  xplg-master:
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=master"
      XPLG_INCEPTION_PARAMS: |
        xplg_inception_init_flow=master
        xplg_inceptions_post_init_flow_tasks=clusterExternalConf

  xplg-ui:
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=ui"
      XPLG_INCEPTION_PARAMS: |
        xplg_inception_init_flow=ui
        xplg_inceptions_post_init_flow_tasks=waitForFile

  xplg-processor:
    environment:
      ram: "-Xmx2g"
      sharedstorage: "xpolog.home.root.path=/home/data"
      profile: "-Dxpolog.uid.structure=processor"
      XPLG_INCEPTION_PARAMS: |
        xplg_inception_init_flow=processor
        xplg_inceptions_post_init_flow_tasks=waitForFile
```

(You can drop `version:` from the file — compose already warned it’s obsolete.)

Then apply:

```bash
docker compose up -d --force-recreate
# give it ~30–60s to initialize the flows
```

## Verify the fix

1. **Master created shared config** (on host):

```bash
ls -l ./data
# expect multiple subdirs/files (not just .write_test)
```

2. **All nodes see it**:

```bash
docker compose exec xplg-ui        sh -lc 'ls -l /home/data | head'
docker compose exec xplg-processor sh -lc 'ls -l /home/data | head'
```

3. **UI shows full cluster**
   Open **System Status** → you should see **Master, UI, Processor**.

4. **Try adding a log again**
   Use **Local → Browse** inside **xplg-ui**, pick (for example)
   `/home/xplg/xplg-cluster/cluster-logs/master/tomcat/catalina.2025-09-16.log`, click **Add Log**, and set first read to **From now**.
   This time it should **persist** (because it can now write to `/home/data`).

## About the “AntLoader” errors

They’re real but not your current bottleneck. They likely come from an internal tool/class not included in this Graal image. Good next step is just to confirm they’re not spamming constantly:

```bash
for c in xplg-ui xplg-master xplg-processor; do
  echo "== $c =="
  docker exec $c sh -lc "date; tail -n 200 /opt/xplg-service/log/XpoLogConsole.log | egrep -c 'AntLoader|ClassNotFoundException' || true"
done
```

If counts keep jumping every few seconds, tell me and we’ll disable the triggering task. Otherwise, safe to ignore for now.

## “busybox not found” – alt commands

Your image doesn’t have busybox. Use `/proc` directly to find the Java PID when needed:

```bash
docker exec xplg-ui sh -lc '
for p in /proc/[0-9]*; do
  cmd=$(tr -d "\0" < "$p/cmdline" 2>/dev/null || true)
  case "$cmd" in *java*) echo "${p##*/}"; break;; esac
done
' 
```

Then you can read `/proc/$pid/status` and `/proc/$pid/io` without `ps`.

## Why your UI-created things disappeared after down/up

Because the app wasn’t writing to the host bind (`/home/data`). It wrote into the container layer; recreating the container wiped it. After the fix above, UI config/state land in **`./data`** on the host and **survive** restarts.

---

If you want, paste `ls -al ./data` after the restart and a screenshot of **System Status** — that will confirm the cluster is truly wired and persistence is working.


----

##### Wed sep 17 2025

- [History of docker run commands on VM-85]

Perfect—here’s your compose with the **named volume for `/home/data`** plus the **host bind-mounts** you asked for:

```yaml
version: "3.8"

name: xplg-graal-cluster

networks:
  xplgnet:

volumes:
  xplg_data: {}   # Named volume for shared persistent storage (/home/data in all nodes)

# Reusable env map (so we can merge it into each service without overwriting)
x-xplg-common-env: &xplg-common-env
  ram: "-Xmx2g"
  # MUST be the full property string, not just a path
  sharedstorage: "xpolog.home.root.path=/home/data"

# Reusable common service bits
x-xplg-common: &xplg-common
  image: xplghub/images:xplg.RC-Graal-10206
  restart: unless-stopped
  networks: [xplgnet]
  environment: *xplg-common-env
  volumes:
    # Same named volume mounted at the same path in all nodes
    - xplg_data:/home/data

services:
  master:
    <<: *xplg-common
    container_name: xplg-master
    environment:
      <<: *xplg-common-env
      profile: "-Dxpolog.uid.structure=master"
      # Written verbatim into /opt/xplg-service/inception-conf.prop
      XPLG_INCEPTION_PARAMS: |
        xplg_inception_init_flow=master
        xplg_inceptions_post_init_flow_tasks=clusterExternalConf
    expose:
      - "30303"
      - "30443"
    volumes:
      # Persist MASTER logs to host
      - type: bind
        source: /home/xplg/xplg-cluster/cluster-logs/master/tomcat
        target: /opt/xplg-service/ServletContainer/logs
      - type: bind
        source: /home/xplg/xplg-cluster/cluster-logs/master/app
        target: /opt/xplg-service/log

  ui:
    <<: *xplg-common
    container_name: xplg-ui
    depends_on: [master]
    environment:
      <<: *xplg-common-env
      profile: "-Dxpolog.uid.structure=ui"
      XPLG_INCEPTION_PARAMS: |
        xplg_inception_init_flow=ui
        xplg_inceptions_post_init_flow_tasks=waitForFile
    # Publish UI ports to the host
    ports:
      - "30307:30303"  # HTTP
      - "30447:30443"  # HTTPS
    volumes:
      # Persist UI logs to host
      - type: bind
        source: /home/xplg/xplg-cluster/cluster-logs/ui/tomcat
        target: /opt/xplg-service/ServletContainer/logs
      - type: bind
        source: /home/xplg/xplg-cluster/cluster-logs/ui/app
        target: /opt/xplg-service/log
      # Also mount the whole cluster-logs tree for convenient browsing from the UI
      - type: bind
        source: /home/xplg/xplg-cluster/cluster-logs
        target: /home/xplg/xplg-cluster/cluster-logs

  processor:
    <<: *xplg-common
    container_name: xplg-processor
    depends_on: [master]
    environment:
      <<: *xplg-common-env
      profile: "-Dxpolog.uid.structure=processor"
      XPLG_INCEPTION_PARAMS: |
        xplg_inception_init_flow=processor
        xplg_inceptions_post_init_flow_tasks=waitForFile
    expose:
      - "30303"
      - "30443"
    volumes:
      # Persist PROCESSOR logs to host
      - type: bind
        source: /home/xplg/xplg-cluster/cluster-logs/processor/tomcat
        target: /opt/xplg-service/ServletContainer/logs
      - type: bind
        source: /home/xplg/xplg-cluster/cluster-logs/processor/app
        target: /opt/xplg-service/log
```

### Notes

* `/home/data` is stored in the **named volume `xplg_data`** (survives `down` / host reboots).
* Each node’s Tomcat/app logs are persisted to the host under:

  * `/home/xplg/xplg-cluster/cluster-logs/ui/{tomcat,app}`
  * `/home/xplg/xplg-cluster/cluster-logs/master/{tomcat,app}`
  * `/home/xplg/xplg-cluster/cluster-logs/processor/{tomcat,app}`
* The UI also mounts the **entire** `cluster-logs` tree at the same path inside the container for easy “Local → Browse” selection of **any** node’s logs.

Tip (once): ensure the host folders exist before `up`:

```bash
mkdir -p /home/xplg/xplg-cluster/cluster-logs/{ui,master,processor}/{tomcat,app}
```

##### history vm 85

```bash


 3741  IMAGE="xplghub/images:xplg.RC-Graal-10206"
 3742  docker pull "$IMAGE" 
 3743  ls -alhtr ~/
--
 3768  docker image inspect xplghub/images:xpolog-center-7.Release-9787 | jq .
 3769  docker inspect 628 | jq . 
 3770  echo keepAlive
 3771  docker run -d --name flux-trg \\n    -p 30306:30303 \\n    -p 30446:30443 \\n    -v /home/xplg/X8.ODD/x8.odd.storm/odd.test:/app/config \\n    -v /home/xplg/:/home/xplg \\n    -e sharedstorage=/home/xplg/targetExtConf \\n    'xplghub/images:xplg.RC-Graal-10206'\n
 3772  echo keepAlive
 3773  cat /home/xplg/ODDTests/python/pytest/xplg/flux/odd.storm/odd.test/StormForge/fluxTestTopology.yml
 3774  cd /home/xplg/ODDTests/python/pytest/xplg/flux/odd.storm/odd.test/StormForge/
--
 3797  cd /home/xplg/graal/targetExtConf
 3798  pwd
 3799  cd ~
 3800  docker run -d --name xplg-graal \\n  -p 30306:30303 \\n  -p 30446:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n  --restart unless-stopped \\n  xplghub/images:xplg.RC-Graal-10206
 3801  docker ps
 3802  docker run  --platform linux/amd64 \\n  -d  --name xplg-graal \\n  -p 30307:30303 \\n  -p 30447:30443 \\n  -p 38447:8443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v $(pwd)$:/home/xplg \\n  --restart unless-stopped \\n  "$IMG" 
 3803  docker run  -d --name xplg-graal \\n  -p 30307:30303 \\n  -p 30447:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n  --restart unless-stopped \\n  xplghub/images:xplg.RC-Graal-10206 
 3804  docker rm -f /xplg-graal  2>/dev/null || true 
 3805  docker run  -d --name xplg-graal \\n  -p 30307:30303 \\n  -p 30447:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx4g' \\n  -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n  --restart unless-stopped \\n  xplghub/images:xplg.RC-Graal-10206 
 3806  docker ps
 3807  docker restart xplg-graal
 3808  docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"\n 
--
 3866  docker exec -it xplg-graal sh -lc '\n  lg=/opt/xplg-service/ServletContainer/logs;\n  ls -ltr "$lg" | tail -n 5;\n  egrep -i "ProtocolHandler|Server startup|BindException|Address already in use|SEVERE|ERROR" "$lg"/catalina.*.log | tail -n 120\n'\n 
 3867  docker exec -it xplg-graal sh -lc '\n  for p in /proc/[0-9]*; do\n    cmd=$(tr -d "\0" < "$p/cmdline" 2>/dev/null)\n    case "$cmd" in *java*) echo "$p $cmd";; esac\n  done | head -n 5\n'\n 
 3868  docker stop xplg-graal; docker rm xplg-graal
 3869  docker run -d --name xplg-graal \\n  -p 30307:30303 -p 30447:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx2g' \\n  xplghub/images:xplg.RC-Graal-10206 
 3870  curl -sv http://127.0.0.1:30307/ | head -n 20
 3871  curl -skI https://127.0.0.1:30447/ | head -n 20 
 3872  docker exec -it xplg-graal sh -lc \\n  'tail -n 200 /opt/xplg-service/ServletContainer/logs/catalina.2025-09-14.log'\n 
--
 3888  docker ps --format 'table {{.Names}}\t{{.Ports}}' | sed -n '1,5p' 
 3889  docker inspect xplg-graal
 3890  docker stop xplg-graal && docker rm xplg-graal
 3891  docker run -d --name xplg-graal \\n  -p 30307:30303 -p 30447:30443 \\n  -e sharedstorage=/home/xplg/targetExtConf \\n  -e ram='-Xmx2g' \\n  -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n  --restart unless-stopped \\n  xplghub/images:xplg.RC-Graal-10206 
 3892  ls -althr  ./graal/targetExtConf
 3893  docker restart xplg-graal
 3894  docker inspect xplg-graal
--
 3900  ls -althr  ~/
 3901  ls -althr  ~/graal
 3902  docker network create xplgnet || true\n 
 3903  docker run -d --name xplg-master \\n  --network xplgnet \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=master' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=master\nxplg_inceptions_post_init_flow_tasks=clusterExternalConf' \\n  xplghub/images:xplg.RC-Graal-10206\n 
 3904  docker run -d --name xplg-ui \\n  --network xplgnet \\n  -p 30307:30303 -p 30447:30443 \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=ui' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=ui\nxplg_inceptions_post_init_flow_tasks=waitForFile' \\n  xplghub/images:xplg.RC-Graal-10206\n 
 3905  docker ps --format 'table {{.Names}}\t{{.Ports}}' | sed -n '1,5p' 
 3906  docker stop xplg-graal && docker rm xplg-graal 
 3907  docker run -d --name xplg-ui \\n  --network xplgnet \\n  -p 30307:30303 -p 30447:30443 \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=ui' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=ui\nxplg_inceptions_post_init_flow_tasks=waitForFile' \\n  xplghub/images:xplg.RC-Graal-10206\n 
 3908  docker ps --format 'table {{.Names}}\t{{.Ports}}' | sed -n '1,5p' 
 3909  docker ps
 3910  docker rm -f /xplg-ui  2>/dev/null || true 
 3911  docker run -d --name xplg-ui \\n  --network xplgnet \\n  -p 30307:30303 -p 30447:30443 \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=ui' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=ui\nxplg_inceptions_post_init_flow_tasks=waitForFile' \\n  xplghub/images:xplg.RC-Graal-10206\n 
 3912  docker run -d --name xplg-processor \\n  --network xplgnet \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=processor' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=processor\nxplg_inceptions_post_init_flow_tasks=waitForFile' \\n  xplghub/images:xplg.RC-Graal-10206\n 
 3913  docker ps --format "table {{.Names}}\t{{.Ports}}"\n
 3914  curl -sI  http://127.0.0.1:30307/ | head 
 3915  curl -skI https://127.0.0.1:30447/ | head 
--
 4067  ls -althr  ./
 4068  ls -althr  ./cluster-logs/
 4069  docker network create xplgnet 
 4070  docker run -d --name xplg-master \\n  --network xplgnet \\n  --restart unless-stopped \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=master" \\n  -e xplg_inception_init_flow="master" \\n  -e xplg_inceptions_post_init_flow_tasks="clusterExternalConf" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/master/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/master/app:/opt/xplg-service/log" \\n  xplghub/images:xplg.RC-Graal-10206 
 4071  docker run -d --name xplg-ui \\n  --network xplgnet \\n  --restart unless-stopped \\n  -p 30307:30303 -p 30447:30443 \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=ui" \\n  -e xplg_inception_init_flow="ui" \\n  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/ui/app:/opt/xplg-service/log" \\n  xplghub/images:xplg.RC-Graal-10206 
 4072  docker run -d --name xplg-processor \\n  --network xplgnet \\n  --restart unless-stopped \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=processor" \\n  -e xplg_inception_init_flow="processor" \\n  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/processor/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/processor/app:/opt/xplg-service/log" \\n  xplghub/images:xplg.RC-Graal-10206\n 
 4073  docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
 4074  curl -I  http://127.0.0.1:30307/ | sed -n '1,8p' 
 4075  curl -kI https://127.0.0.1:30447/ | sed -n '1,8p' 
--
 4079  ls -al ./data
 4080  docker exec -it xplg-ui sh -lc 'ls -l /home/data; ls -l /opt/xplg-service/ServletContainer/logs | head'\n
 4081  docker stop xplg-ui xplg-master xplg-processor
 4082  docker run  -d --name xplg-graal \\n   -p 30307:30303 \\n   -p 30447:30443 \\n   -e sharedstorage=/home/xplg/targetExtConf \\n   -e ram='-Xmx4g' \\n   -v /home/xplg/graal/targetExtConf:/home/xplg/targetExtConf \\n   --restart unless-stopped \\n    xplghub/images:xplg.RC-Graal-10206 
 4083  docker ps
 4084  docker stop xplg-graal
 4085  echo $HOST_DATA
--
 4089  export HOST_DATA=/home/xplg/graal/data 
 4090  echo $HOST_DATA
 4091  docker network create xplgnet || true 
 4092  docker run -d --name xplg-master \\n  --network xplgnet \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=master' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=master\nxplg_inceptions_post_init_flow_tasks=clusterExternalConf' \\n  xplghub/images:xplg.RC-Graal-10206
 4093  docker rm -f /xplg-processor  2>/dev/null || true 
 4094  docker rm -f /xplg-ui  2>/dev/null || true 
 4095  docker rm -f /xplg-master  2>/dev/null || true 
 4096  docker run -d --name xplg-master \\n  --network xplgnet \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=master' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=master\nxplg_inceptions_post_init_flow_tasks=clusterExternalConf' \\n  xplghub/images:xplg.RC-Graal-10206
 4097  docker run -d --name xplg-ui \\n  --network xplgnet \\n  -p 30307:30303 -p 30447:30443 \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=ui' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=ui\nxplg_inceptions_post_init_flow_tasks=waitForFile' \\n  xplghub/images:xplg.RC-Graal-10206 
 4098  docker run -d --name xplg-processor \\n  --network xplgnet \\n  -v "$HOST_DATA":/home/data \\n  -e ram='-Xmx2g' \\n  -e sharedstorage='xpolog.home.root.path=/home/data' \\n  -e profile='-Dxpolog.uid.structure=processor' \\n  -e XPLG_INCEPTION_PARAMS=$'xplg_inception_init_flow=processor\nxplg_inceptions_post_init_flow_tasks=waitForFile' \\n  xplghub/images:xplg.RC-Graal-10206 
 4099  docker ps
 4100  docker ps --format "table {{.Names}}\t{{.Ports}}" 
 4101  curl -sI  http://127.0.0.1:30307/ | head
--
 4109  cd ./xplg-cluster
 4110  ls -althr  ./
 4111  docker ps --format "table {{.Names}}\t{{.Ports}}" 
 4112  docker run -d --name xplg-ui \\n  --network xplgnet \\n  -p 30307:30303 -p 30447:30443 \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=ui" \\n  -e xplg_inception_init_flow="ui" \\n  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/ui/app:/opt/xplg-service/log" \\n  --health-cmd 'wget -q --spider http://127.0.0.1:30303/ || curl -fsS http://127.0.0.1:30303/ >/dev/null' \\n  --health-interval 15s --health-timeout 3s --health-retries 5 --health-start-period 60s \\n  xplghub/images:xplg.RC-Graal-10206 
 4113  ddd
 4114  ls
 4115  docker rm -f xplg-ui 
 4116  ps -ef
 4117  docker run -d --name xplg-ui \\n  --network xplgnet \\n  -p 30307:30303 -p 30447:30443 \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=ui" \\n  -e xplg_inception_init_flow="ui" \\n  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/ui/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/ui/app:/opt/xplg-service/log" \\n  --health-cmd 'wget -q --spider http://127.0.0.1:30303/ || curl -fsS http://127.0.0.1:30303/ >/dev/null' \\n  --health-interval 15s --health-timeout 3s --health-retries 5 --health-start-period 60s \\n  xplghub/images:xplg.RC-Graal-10206
 4118  docker ps
 4119  ping orchestrator
 4120  docker run -d --name xplg-master \\n  --network xplgnet \\n  --restart unless-stopped \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=master" \\n  -e xplg_inception_init_flow="master" \\n  -e xplg_inceptions_post_init_flow_tasks="clusterExternalConf" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/master/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/master/app:/opt/xplg-service/log" \\n  xplghub/images:xplg.RC-Graal-10206 
 4121  docker rm -f xplg-master
 4122  docker run -d --name xplg-master \\n  --network xplgnet \\n  --restart unless-stopped \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=master" \\n  -e xplg_inception_init_flow="master" \\n  -e xplg_inceptions_post_init_flow_tasks="clusterExternalConf" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/master/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/master/app:/opt/xplg-service/log" \\n  xplghub/images:xplg.RC-Graal-10206 
 4123  docker run -d --name xplg-processor \\n  --network xplgnet \\n  --restart unless-stopped \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=processor" \\n  -e xplg_inception_init_flow="processor" \\n  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/processor/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/processor/app:/opt/xplg-service/log" \\n  xplghub/images:xplg.RC-Graal-10206 
 4124  docker rm -f xplg-processor
 4125  docker run -d --name xplg-processor \\n  --network xplgnet \\n  --restart unless-stopped \\n  -e ram="-Xmx2g" \\n  -e sharedstorage="xpolog.home.root.path=/home/data" \\n  -e profile="-Dxpolog.uid.structure=processor" \\n  -e xplg_inception_init_flow="processor" \\n  -e xplg_inceptions_post_init_flow_tasks="waitForFile" \\n  -v "$(pwd)/data:/home/data" \\n  -v "$(pwd)/cluster-logs/processor/tomcat:/opt/xplg-service/ServletContainer/logs" \\n  -v "$(pwd)/cluster-logs/processor/app:/opt/xplg-service/log" \\n  xplghub/images:xplg.RC-Graal-10206 
 4126  docker ps --format "table {{.Names}}\t{{.Ports}}" 
 4127  docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'\n 
 4128  curl -I  http://127.0.0.1:30307/ | sed -n '1,8p'
--
 4138  docker exec xplg-ui        sh -lc "grep -m1 'Server startup in' /opt/xplg-service/ServletContainer/logs/catalina.*.log" 
 4139  docker exec xplg-master    sh -lc "grep -m1 'Server startup in' /opt/xplg-service/ServletContainer/logs/catalina.*.log" 
 4140  docker exec xplg-processor sh -lc "grep -m1 'Server startup in' /opt/xplg-service/ServletContainer/logs/catalina.*.log"\n 
 4141  docker run --rm --network container:xplg-ui curlimages/curl:8.5.0 \\n  -sS -o /dev/null -w "UI http: %{http_code}\n" http://127.0.0.1:30303/ 
 4142  docker run --rm --network container:xplg-ui curlimages/curl:8.5.0 \\n  -k -sS -o /dev/null -w "UI https: %{http_code}\n" https://127.0.0.1:30443/ 
 4143  docker run --rm --network container:xplg-master curlimages/curl:8.5.0 \\n  -sS -o /dev/null -w "Master http: %{http_code}\n" http://127.0.0.1:30303/ 
 4144  docker run --rm --network container:xplg-master curlimages/curl:8.5.0 \\n  -k -sS -o /dev/null -w "Master https: %{http_code}\n" https://127.0.0.1:30443/ 
 4145  docker run --rm --network container:xplg-processor curlimages/curl:8.5.0 \\n  -sS -o /dev/null -w "Processor http: %{http_code}\n" http://127.0.0.1:30303/ 
 4146  docker run --rm --network container:xplg-processor curlimages/curl:8.5.0 \\n  -k -sS -o /dev/null -w "Processor https: %{http_code}\n" https://127.0.0.1:30443/ 
 4147  docker stop xplg-ui xplg-master xplg-processor
 4148  ls -althr  ./
 4149  touch docker-compose.yml

 
```
----

- [Go to Top](#table-of-contents)