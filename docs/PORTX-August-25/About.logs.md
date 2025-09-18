#### Nav

- [List Container's bind-mounts (host → container, and RW flag)](#checks-to-run-on-vm-where-the-xplg-container-is-running)

- [XPLG-Docs Log Pattern Lang](https://xpolog.atlassian.net/wiki/spaces/XPOL6/pages/1607435738/XpoLog+Patterns+Language)

- [XPLG-Docs Addding a Local Log](https://xpolog.atlassian.net/wiki/spaces/XPOL/pages/1594132663/Adding+a+Local+Log)

- [About logs in general](#about-logs-in-general)

- [What is rsyslog?](#1-what-is-rsyslog)

- [About Git-Runner Logs](#git-runner-logs)

- [Syslog UDP Vs Syslog TCP]()

- [Go to Bottom](#bottom)

---

---

##### about logs in general

Got it 👍 — you basically want **dummy log files** that look like real-world logs, so that PORTX (XpoLog app) can ingest them and you can check parsing.

Here’s a good approach:

---

## 1. Decide which _log flavors_ you want to test

Usually you want a representative set:

- **Web server logs** – Apache/Nginx common/combined format
- **Application logs** – generic timestamp + level + message (Java, Python, Node.js style)
- **System logs** – syslog style (`<pri>timestamp host process[pid]: message`)
- **Custom JSON logs** – many apps now log JSON events
- **Database logs** – e.g. MySQL or PostgreSQL query logs

---

## 2. Generate some sample files

You don’t need real infra — you can either:

- **Use built-in Linux tools**: `logger`, `journalctl`, `dmesg`, `syslog-ng` test messages
- **Run a small container** with Apache/Nginx/Postgres and grab its logs
- **Use a generator script** (Python, Go, Node.js) to emit fake but structured lines

---

### Example snippets:

#### Web server logs (Apache/Nginx)

```bash
echo '127.0.0.1 - - [18/Aug/2025:10:33:21 +0000] "GET /index.html HTTP/1.1" 200 1024' >> access.log
echo '192.168.1.20 - john [18/Aug/2025:10:33:45 +0000] "POST /login HTTP/1.1" 403 321' >> access.log
```

#### Application logs

```bash
echo '2025-08-18 10:35:01,123 INFO  [auth-service] User login successful: user=alice' >> app.log
echo '2025-08-18 10:36:22,567 ERROR [payment-service] Payment failed: id=12345 reason="card declined"' >> app.log
```

#### Syslog

```bash
logger -p local0.info "Fake syslog: service started"
logger -p local0.err "Fake syslog: service crashed with code=137"
```

(this writes to `/var/log/syslog` or `/var/log/messages`, you can copy lines out)

#### JSON logs

```bash
echo '{"timestamp":"2025-08-18T10:37:00Z","level":"warn","service":"gateway","message":"Rate limit exceeded","client_ip":"203.0.113.5"}' >> json.log
```

#### Database log (MySQL-style)

```bash
echo '2025-08-18T10:38:12.345678Z   15 Query SELECT * FROM users WHERE id=42' >> mysql.log
```

---

## 3. Automate a bit (optional)

You could write a Python script to generate **N random lines** with random IPs, timestamps, status codes, etc. That way you can create 1000-line files for testing parsing performance.

---

## 4. Upload to PORTX

- Put them under `/tmp/test-logs/` or whatever dir your workflow/playbook already provisions.
- Then use your existing **`ansible-upload-files-to-remote.yml`** reusable to send them to the target VM.

---

✅ This gives you **diverse, but deterministic** sample logs.
You’ll be able to check: does PORTX auto-detect log format? Does it parse fields correctly? Does JSON get expanded?

---

Do you want me to write you a **Python “log generator” script** that can output multiple formats (nginx, syslog, json, etc.) into files with N entries each? That way you can keep it in your repo and run it anytime.

---

- [Go to Top](#nav)

---

#### Checks to run on vm where the xplg container is running

We have **running container(s) on the “storm” VM with bind-mounts**, and you want me to **point you to an existing host path to upload logs** (so you can test the xplg-portx viewer/parser) **without proposing new mounts or code changes**. Got it.

Here’s the clean, no-changes way to pick the _right existing_ upload path:

## Goal

Find a **writable existing bind-mount** for the xplg-portx container, then drop logs into its **host side** so they appear inside the container at the **container side** path you’ll select in the UI.

## 4 quick checks (safe, read-only inspection)

1. **Locate the xplg container name/ID**

- `docker ps --format '{{.Names}}\t{{.ID}}' | grep -i xplg`
- `docker ps --format '{{.ID}}  {{.Names}}  {{.Ports}}' | grep 30306`
```bash
 docker exec -it 92a  sh -lc 'ls -ld /app/config ; id' 
 3595  docker ps
 3596  docker logs --tail=200 flux-sync-ci-flux-src-1
 3597  docker logs --tail=200 flux-sync-ci-flux-tgt-1
 3598  docker inspect flux-sync-ci-flux-tgt-1 \ --format '{{json .Config.Labels}}' | jq .
 3599  docker logs flux-sync-ci-flux-tgt-1 -f
 3600  docker logs --tail=200 flux-sync-ci-flux-tgt-1

```

2. **List its bind-mounts (host → container, and RW flag)**

- `docker inspect <container_name_or_id> --format '{{range .Mounts}}{{.Type}}  {{.Source}}  ->  {{.Destination}}  (RW={{.RW}}){{println}}{{end}}'`

3. **Pick your upload spot**

- Choose a line where:

  - `Type` is `bind`
  - `RW=true` (writable)
  - `Destination` looks like a data/ingest area (common: `/data`, `/var/log`, `/logs`, `/ingest`, `/app/data`, etc.)

- Your **host upload path** is the `Source`. Your **in-container path** is the `Destination`.

4. **Sanity-verify from both sides (non-destructive)**

- Inside container: `docker exec -it <container> sh -lc 'ls -ld <Destination> ; id'`
- On host: `ls -ld <Source>`

  - You want write permission for the container’s runtime user (seen via `id` above). If needed later, we can adjust ownership/perms on the host path only (no compose change).

## What to do next (still no code changes)

- Create a **subfolder** under the chosen **host Source** (e.g., `…/inbox/`) and place your sample logs there.
- Confirm they appear inside the container at **`<Destination>/inbox`**:

  - `docker exec -it <container> sh -lc 'ls -l <Destination>/inbox'`

- In the XPLG UI, point the data source/pattern to **`<Destination>/inbox/*.log`** (that’s the container-side path).

### Example of the “decision table” you’ll fill from step 2

| Container  | Host Source (chosen)       | Container Destination | RW   | Subdir you’ll use |
| ---------- | -------------------------- | --------------------- | ---- | ----------------- |
| xplg-portx | `/home/xplg/whatever-data` | `/data`               | true | `/inbox`          |

> Tip: You already have a convenient sample log on the VM:
> `/home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log`
> You can copy it into the chosen **host Source** (e.g., `…/inbox/`) to test parsing right away.

## If you paste the output of step **(2)** (the mount list), I’ll point to **the exact host path** you should use and the matching **container path** to select in the UI—no new mounts, no compose/Ansible changes.

##### Unpacking outputs of probes

```bash

 xplg@flux1  ~  docker ps --format '{{.Names}}\t{{.ID}}'
flux-sync-ci-diag-1     c46a4b0ab058
flux-sync-ci-flux-src-1 92a738ea5c96
flux-sync-ci-flux-tgt-1 c42daf6ac097
 xplg@flux1  ~  docker ps
CONTAINER ID   IMAGE                                         COMMAND                  CREATED          STATUS                    PORTS                                                                                              NAMES
c46a4b0ab058   curlimages/curl:8.5.0                         "/entrypoint.sh tail…"   40 minutes ago   Up 40 minutes                                                                                                                flux-sync-ci-diag-1
92a738ea5c96   xplghub/images:xpolog-center-7.Release-9787   "/bin/sh -c ./start.…"   40 minutes ago   Up 40 minutes (healthy)   0.0.0.0:30305->30303/tcp, [::]:30305->30303/tcp, 0.0.0.0:30445->30443/tcp, [::]:30445->30443/tcp   flux-sync-ci-flux-src-1
c42daf6ac097   xplghub/images:xpolog-center-7.Release-9787   "/bin/sh -c ./start.…"   40 minutes ago   Up 40 minutes (healthy)   0.0.0.0:30306->30303/tcp, [::]:30306->30303/tcp, 0.0.0.0:30446->30443/tcp, [::]:30446->30443/tcp   flux-sync-ci-flux-tgt-1
 xplg@flux1  ~  docker ps --format '{{.Names}}\t{{.ID}}' | grep -i xplg
 ✘ xplg@flux1  ~  docker inspect 92a c42  --format '{{range .Mounts}}{{.Type}}  {{.Source}}  ->  {{.Destination}}  (RW={{.RW}}){{println}}{{end}}'
bind  /home/xplg  ->  /home/xplg  (RW=true)
bind  /home/xplg/ODDTests/python/pytest/xplg/flux/src  ->  /app/config  (RW=true)

bind  /home/xplg/ODDTests/python/pytest/xplg/flux/src  ->  /app/config  (RW=true)
bind  /home/xplg  ->  /home/xplg  (RW=true)

 xplg@flux1  ~  docker exec -it 92a  sh -lc 'ls -ld /app/config ; id'
drwxr-xr-x 9 1000 1000 4096 Aug 29 14:55 /app/config
uid=0(root) gid=0(root) groups=0(root)
 xplg@flux1  ~  ls -ld /home/xplg/ODDTests/python/pytest/xplg/flux/src
drwxr-xr-x 9 xplg xplg 4096 Aug 29 14:55 /home/xplg/ODDTests/python/pytest/xplg/flux/src
 xplg@flux1  ~  docker exec -it 92a  sh -lc 'ls -ld /home/xplg/ODDTests/python/pytest/xplg/flux/src  ; id'
drwxr-xr-x 9 1000 1000 4096 Aug 29 14:55 /home/xplg/ODDTests/python/pytest/xplg/flux/src
uid=0(root) gid=0(root) groups=0(root)
 xplg@flux1  ~ 
```

You’re asking two things:

1. whether it’s better to **use logs the container already produces** (vs. seeding “fake” samples), and
2. given your VM output, **which existing bind-mount path** you should use to upload/point XPLG at—**without adding new mounts or code changes**.

Here’s what your evidence shows (trusted: your `docker inspect/exec` output):

- You have **two RW bind mounts** on the XPLG containers (`flux-src` / `flux-tgt`):

  - **Host `/home/xplg` ⇄ Container `/home/xplg` (RW=true)**
  - **Host `/home/xplg/ODDTests/python/pytest/xplg/flux/src` ⇄ Container `/app/config` (RW=true)**

- The container runs as **root** (`uid=0`), and the mounted host dirs are owned by **UID 1000 (xplg)**—still fine for read/write inside the container because you’re root there.

### 1) Should we use the container’s own logs?

**Yes, that’s ideal**—real traffic beats synthetic. The only constraint is **visibility from inside the container via an existing mount** (since you don’t want new mounts). If the product’s internal logs are **already under `/home/xplg`**, you can point XPLG at them directly.
If they’re **not** under `/home/xplg`, a no-change workaround is to **copy** (or rotate) the container’s internal logs **into `/home/xplg/...`** from inside the container. That exposes them through the existing bind mount—no Docker/Compose edits needed.

### 2) The clean, no-changes ingest path to use now

Based on your mounts, the **safest upload/ingest location** is under the `/home/xplg` bind:

- **Host upload path (recommended):** `/home/xplg/xplg-ingest`
- **Container path to select in XPLG UI:** `/home/xplg/xplg-ingest/*.log`

Why not `/app/config`? It’s clearly a **config** mount; it’s better practice to keep log inputs out of config paths.

### Two simple ways to proceed (no new mounts)

- **A. Use the container’s own logs (preferred):**
  From **inside the container**, copy its current log files into `/home/xplg/xplg-ingest/`. Then in XPLG, add a data source pointing to `/home/xplg/xplg-ingest/*.log`.
  _Result:_ You’re testing with authentic logs, and they’re visible via the existing `/home/xplg` bind.

- **B. Use existing real logs on the VM:**
  You already have real samples like
  `/home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log`
  (this path is under `/home/xplg`, so it’s **already visible** inside the container). Copy or symlink it into `/home/xplg/xplg-ingest/` and point XPLG at that folder.

### Quick verification (non-destructive, just checks)

- **Inside container:** confirm you can see the folder
  `ls -ld /home/xplg/xplg-ingest`
- **From host:** confirm perms/ownership
  `ls -ld /home/xplg/xplg-ingest`
- **In XPLG UI:** set the source to `/home/xplg/xplg-ingest/*.log` and begin defining patterns.

## If you want, I can prepare a **minimal checklist of shell commands** to (1) create `/home/xplg/xplg-ingest`, (2) copy either the container’s own logs or the `synthetic_flux_source.log` into it, and (3) verify visibility from both sides—**but I won’t write any code/automation until you confirm this plan**.

You’re asking me to:

1. give you a concise **checklist of shell commands** to stage and verify logs on the “storm” VM using the **existing** `/home/xplg` bind mount (no new mounts), and
2. answer your question **A**: if `/home/xplg` is already bind-mounted, why not just point the XPLG UI directly at the _real_ log path instead of creating a new folder and copying?

---

## A) Directly point to real logs vs. creating a new folder

- **Point directly (totally fine when possible):**
  If the logs you want are already **under `/home/xplg/...`**, you can point the XPLG UI straight to that container-side path (same path inside the container because `/home/xplg` is bind-mounted RW). This is simplest and avoids duplication.

- **Why a dedicated folder sometimes helps (`/home/xplg/xplg-ingest`):**

  - **Isolation:** keeps test inputs separate from configs, venvs, or huge directories under `/home/xplg`, so XPLG doesn’t accidentally scan a ton of files.
  - **Repeatability:** one stable path you can reuse across tests, docs, and CI.
  - **Cleanup:** easy to wipe old test sets without touching other files.
  - **Mixed sources:** you can copy/symlink a few chosen files from different places into one small folder.
  - **Permissions sanity:** you can enforce expected ownership/mode on just that folder.

If you already know the **exact** real log file(s) you want and they live under `/home/xplg/...`, go direct. Otherwise, use the small ingest folder to keep things neat.

---

## Checklist of shell commands (no changes to Docker/Compose)

> Use either **Option 1 (Direct)** _or_ **Option 2 (Ingest folder)**.
> Replace `<CONT>` with your container name, e.g. `flux-sync-ci-flux-src-1` (or `flux-sync-ci-flux-tgt-1`).

### Option 1 — Point the UI directly at real logs under `/home/xplg`

1. **Pick the logs (host):**

```bash
# Example: you mentioned this real log already exists
ls -l /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log
```

2. **Confirm the same path is visible in the container:**

```bash
CONT=flux-sync-ci-flux-src-1
docker exec -it "$CONT" sh -lc 'ls -l /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log ; id'
```

3. **(Optional) Preview a few lines to ensure readability:**

```bash
docker exec -it "$CONT" sh -lc 'head -n 50 /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log'
```

4. **In XPLG UI:** configure the **container-side** path (same as host under this mount), e.g.:

```
/home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log
# or, for a folder
/home/xplg/flux-logs-for-sync-tests/*.log
```

---

### Option 2 — Use a tidy ingest folder under `/home/xplg/xplg-ingest`

1. **Create the folder on host and set ownership (host):**

```bash
sudo mkdir -p /home/xplg/xplg-ingest
sudo chown -R xplg:xplg /home/xplg/xplg-ingest
ls -ld /home/xplg/xplg-ingest
```

2. **Seed it with one or more logs (host):**

```bash
# Example: copy the real sample you already have into the ingest folder
cp -v /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log /home/xplg/xplg-ingest/

# (Optional) bring in any other log files you want to test
# cp -v /home/xplg/some/other.log /home/xplg/xplg-ingest/
```

3. **Verify visibility from inside the container:**

```bash
CONT=flux-sync-ci-flux-src-1
docker exec -it "$CONT" sh -lc 'ls -l /home/xplg/xplg-ingest ; head -n 20 /home/xplg/xplg-ingest/synthetic_flux_source.log'
```

4. **In XPLG UI:** point the data source to:

```
/home/xplg/xplg-ingest/*.log
```

---

### (Optional) If you prefer to copy “the container’s own logs” into the bind mount

If the product writes its logs somewhere **outside** `/home/xplg`, you can still expose a subset **without changing mounts** by copying from the app’s log path **inside the container** into the bind-mounted folder:

```bash
CONT=flux-sync-ci-flux-src-1

# 1) (Discover) List likely app log locations (kept shallow to avoid noise)
docker exec -it "$CONT" sh -lc 'for d in /var/log /opt /app /data; do [ -d "$d" ] && find "$d" -maxdepth 2 -type f -name "*.log" 2>/dev/null; done | head -n 50'

# 2) (Copy) Choose specific files and copy them into the bind-mounted ingest folder
docker exec -it "$CONT" sh -lc 'cp -v /path/to/app.log /home/xplg/xplg-ingest/'

# 3) (Verify) Confirm they landed
docker exec -it "$CONT" sh -lc 'ls -l /home/xplg/xplg-ingest'
```

> Note: The container runs as `root` (per your output), so the copy will succeed. If host ownership matters later, you can normalize it:

```bash
sudo chown -R xplg:xplg /home/xplg/xplg-ingest
```

---

If you paste back **the exact real log path(s)** you want to use (or the output of the “Discover” step), I’ll map them 1:1 to the **container-side** path you should enter in the XPLG UI and confirm any edge cases (permissions, globbing, multi-line parsing, etc.).

---

You’re asking me to confirm:

1. whether you can **point the XPLG UI directly** at the log path you already verified inside the container, and
2. give you a **concise checklist of shell commands** to stage/verify logs using the existing `/home/xplg` bind mount—**no new mounts or code**.

**Answer:** Yes—you can point the XPLG UI directly at the file you showed:

```
/home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log
```

Your output proves it’s readable inside the container (runs as root, path exists, `head` works), so you’re good to go.

---

## Minimal checklist (safe, no changes)

> Replace `CONT=flux-sync-ci-flux-src-1` with the container you’re using (or `flux-sync-ci-flux-tgt-1`).
> These steps only confirm visibility and help you point the UI correctly.

```bash
# 0) Choose the container you want XPLG UI in
CONT=flux-sync-ci-flux-src-1

# 1) Verify the exact file from INSIDE the container
docker exec -it "$CONT" sh -lc 'ls -lh /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log && head -n 20 /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log'

# 2) (Optional) Verify from the HOST that perms/size look sane
ls -lh /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log

# 3) In the XPLG UI, add a source pointing to the container-side path:
#    /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log
#    (or use a glob if you drop more files: /home/xplg/flux-logs-for-sync-tests/*.log)

# 4) (Optional) If you want a neat, reusable folder:
sudo mkdir -p /home/xplg/xplg-ingest && sudo chown -R xplg:xplg /home/xplg/xplg-ingest
cp -v /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log /home/xplg/xplg-ingest/
docker exec -it "$CONT" sh -lc 'ls -lh /home/xplg/xplg-ingest'
# In UI you could then point to: /home/xplg/xplg-ingest/*.log
```

---

## Why your `find` returned nothing

Your `find` looked for `*.log` under `/var/log /opt /app /data` (depth ≤ 2). Many containers:

- log to **stdout/stderr** (so no files exist); or
- keep logs **deeper** than 2 levels; or
- use **non-`.log` extensions/names**.

##### find potential logs on the containers file system

If you ever want to quickly check for likely log files **without** scanning the whole FS, try a slightly broader, still-safe search:

```bash
docker exec -it "$CONT" sh -lc '
for d in /var/log /opt /app /data /home/xplg; do
  [ -d "$d" ] && find "$d" -maxdepth 4 -type f \( -name "*.log" -o -name "*log*" \) 2>/dev/null
done | head -n 80
'
```

---

## Optional: capture live container stdout logs into a file (no compose changes)

If the app only writes to stdout, you can materialize recent logs into the bind mount and then point XPLG at that file:

```bash
# On the host:
CONT=flux-sync-ci-flux-src-1
OUT=/home/xplg/xplg-ingest/stdout-$(date +%Y%m%d-%H%M%S).log
mkdir -p /home/xplg/xplg-ingest
docker logs --since 1h "$CONT" > "$OUT"
ls -lh "$OUT"

# In UI: /home/xplg/xplg-ingest/*.log
```

---

If you want, next I can give you a tiny “first pattern” for that synthetic file (timestamp + message) so you can immediately see columns in the table—only if you confirm you want to go there.

- [Go to Top](#nav)

---

You’re asking me to: **walk you through an actual, concrete implementation in the XPLG (XpoLog) UI**—using the file you already validated inside the container—so you can see parsed columns in the table.

Below are precise, UI-only steps (no code changes), backed by XPLG docs for the Add Data wizard, manual/automatic patterns, and date-format syntax. ([xplg.com][1], [xpolog.atlassian.net][2])

---

# Step-by-step in the UI (using your real file)

**Pre-req (you already did):**
File exists and is readable **inside the container** at:
`/home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log`

**1) Open the XPLG UI**

- Your container maps HTTPS **30443 → host 30445/30446**.
  Open:

  - Source instance: `https://<vm-ip>:30445`
  - (If you use the target one instead: `https://<vm-ip>:30446`)
    Default GUI ports are 30303/30443; your host ports reflect the container’s published ports. ([xpolog.atlassian.net][3])

**2) Add the file as a data source**

- In the left/top navigation choose **Add Data** (aka _Add Logs_ / _Quick Start → Adding Data_). ([xplg.com][1])
- Choose **Local Files** (you’re selecting a path visible to the UI container).
- In the **Path** field, enter:

  ```
  /home/xplg/flux-logs-for-sync-tests/synthetic_flux_source.log
  ```

  (If you’ll add more files later, you can use a glob like `.../*.log`.)

- Click **Next**.

**3) Parsing step — start with Auto, then switch to Manual**

- Let XPLG try **Automatic** parsing first (it often detects simple timestamped lines). ([xpolog.atlassian.net][2])
- Then click **Advanced / Manual Pattern** to lock in an explicit pattern. (Manual patterns are recommended for stable, tabular results.) ([xpolog.atlassian.net][4])

**4) Enter a concrete pattern for your sample lines**
Your lines look like:
`2025-08-06T19:01:32+00:00 LINE 1`
Use this **manual pattern** (copy/paste exactly):

```
{date:Time,yyyy-MM-dd'T'HH:mm:ssXXX} {string:Message}
```

- `XXX` handles ISO-8601 offsets like `+00:00`.
- Field names (`Time`, `Message`) are the column headers you’ll see.
- XPLG’s Patterns Language supports this date format and named columns. ([xpolog.atlassian.net][5])

> Optional (just display formatting):
> If you prefer the table to show the date as `yyyy-MM-dd HH:mm:ss`, append a **UI view format** after a semicolon:
>
> ```
> {date:Time,yyyy-MM-dd'T'HH:mm:ssXXX;yyyy-MM-dd HH:mm:ss} {string:Message}
> ```
>
> (Same parsed value, different display.) ([xpolog.atlassian.net][5])

**5) Click “Verify Pattern”**

- The wizard shows a preview table with two columns: **Time** and **Message**.
- If preview is empty or misaligned, adjust and verify again (this is explicitly recommended by the Patterns doc). ([xpolog.atlassian.net][5])

**6) Save**

- Assign to a Folder/AppTag if prompted, then **Save**. From now on, queries/alerts/aggregations can use the **Time** and **Message** fields. ([xpolog.atlassian.net][6])

---

## Notes & tips (grounded in docs)

- **Automatic vs Manual:** Auto parsing gives you a quick start; **Manual** ensures stable schema (best for alerts/dashboards). ([xpolog.atlassian.net][2])
- **Date tokens:** XPLG uses Java-style date tokens (e.g., `yyyy`, `MM`, `dd`, `HH`, `mm`, `ss`, `SSS`, `X/XX/XXX` for ISO-8601 zones). ([xpolog.atlassian.net][7])
- **Multiple files:** Use globs (e.g., `/home/xplg/flux-logs-for-sync-tests/*.log`) when you add more logs. ([xplg.com][1])
- **Ports:** Default GUI ports are **30303 (HTTP)** / **30443 (HTTPS)**; yours are remapped to host **30445/30446**—use those in the browser. ([xpolog.atlassian.net][3])

---

## Want me to tailor the pattern further?

If you drop another real log (Apache/Nginx, JSON app logs, etc.), I can give you the exact **ready-to-paste manual pattern** and which fields you’ll get in the table—directly aligned to XPLG’s syntax and verified with the wizard. ([xpolog.atlassian.net][5])

[1]: https://www.xplg.com/quick-start/?utm_source=chatgpt.com "XpoLog Log Management Setup - Quick Start | XpoLog - XPLG"
[2]: https://xpolog.atlassian.net/wiki/spaces/XPOLOG50/pages/1616174371/?utm_source=chatgpt.com "Applying Patterns on the Log - XPLG 5 - XPLG Technical Documentation"
[3]: https://xpolog.atlassian.net/wiki/spaces/XPOL/pages/1707180033/Configure%2BPorts%2Band%2BCertificates%2Bto%2BXpoLog%2BGUI?utm_source=chatgpt.com "Configure Ports and Certificates to XpoLog GUI"
[4]: https://xpolog.atlassian.net/wiki/spaces/XPOLOG50/pages/1616174373/Configuring%2Ba%2BManual%2BPattern?utm_source=chatgpt.com "Configuring a Manual Pattern - XPLG 5 - XPLG Technical Documentation"
[5]: https://xpolog.atlassian.net/wiki/spaces/XPOL/pages/1594131792/XPLG%2BPatterns%2BLanguage?utm_source=chatgpt.com "XPLG Patterns Language"
[6]: https://xpolog.atlassian.net/wiki/spaces/XPOL/pages/1594132149/?utm_source=chatgpt.com "Applying Patterns on the Log - XPLG - XPLG Technical Documentation"
[7]: https://xpolog.atlassian.net/wiki/spaces/XPOLOG50/pages/1616174376/Creating%2Ba%2BPattern%2BUsing%2Bthe%2BWizard?utm_source=chatgpt.com "Creating a Pattern Using the Wizard - XPLG 5 - XPLG Technical Documentation"


- [Go to Top](#nav)
----


##### 
You want: (1) a plain-English explanation of **rsyslog**, (2) a slow, clear plan for ingesting logs from a **GitHub Actions self-hosted runner VM** into your **XPLG** instance, and (3) to start with the **what/why/how** plus where the runner’s logs live, then use XPLG’s **SSH connector** to pull those files. That matches your screenshot flow.

---

## 1) What is rsyslog?

**rsyslog** is the standard Linux logging daemon that can collect local logs and **forward** them over the network (TCP/UDP), with filtering, parsing, and buffering. It’s widely used as the default system logger on many distros. ([Rsyslog][1])

*(We’ll keep rsyslog as an optional “streaming” path later—today we’ll start with file pickup over SSH, as you asked.)*

---

## 2) Where do GitHub self-hosted runner logs live? (WHAT)

On a Linux runner:

* **Per-job diagnostic logs** are written by the runner into the runner install directory’s **`_diag/`** folder. Filenames often start with `Worker_…`. This is the easiest file target for pickup. ([GitHub Docs][2])
  Example path (adjust user/install dir):
  `/home/<runner_user>/actions-runner/_diag/*.log`

* If you installed the runner **as a service** via `./svc.sh`, the **service output** also goes to **systemd journal** (view with `journalctl -u 'actions.runner*'`). That’s not a file; if you want those logs in XPLG, either export them to a file or forward them with rsyslog later. ([GitHub Docs][3])

**Why these two matter (WHY):**

* `_diag/*.log` gives you job/run-level details your pipelines produce.
* journald gives you service health events (runner starting, stopping, errors).

---

## 3) How to ingest with XPLG **over SSH** (HOW)

You’re on the right screen, but the view in your screenshot is a **local** browser inside the container. For **remote** pickup, choose the **SSH/SFTP** source first, then browse the remote filesystem.

**Steps (one-time setup):**

1. In XPLG UI: **Add Data → Over SSH / Remote Linux** (labels vary slightly by version; look for “Over SSH”). XPLG supports remote collection via SSH/SFTP, agent-less. ([xpolog.atlassian.net][4])
2. Enter the runner VM details: host/IP, port **22**, auth (user + password or key).
   *Tip:* a read-only SFTP user is safer.
3. After **Test Connection** succeeds, use the remote path picker to select:
   `**/home/<runner_user>/actions-runner/_diag/**`
   and add a file mask such as `Worker_*.log` (and optionally `Runner_*.log`).
4. Give the source a clear name, set timezone, and **save**.

**Prereqs on the runner VM:**

* Ensure SSH/SFTP is reachable (port 22 open from the XPLG host).
* Make sure the SSH user can **read** the runner directory (`_diag/`).
  (You may need to adjust permissions/group ownership.)

**Note about “less”/commands:** XPLG’s SSH reader may rely on standard Unix tools on the remote host (e.g., `less`). If they’re missing, XPLG suggests “LogAway” as a fallback—there’s an official doc about it. ([xpolog.atlassian.net][5])

---

## 4) Sanity checks (so you know it’s working)

On the runner VM:

```bash
# See latest per-job logs being created
ls -ltr /home/<runner_user>/actions-runner/_diag | tail

# Generate a tiny test entry in a new diag file (optional)
echo "$(date -Is) test-from-runner" >> /home/<runner_user>/actions-runner/_diag/manual_test.log
```

Back in XPLG:

* Open **Search**, filter by your new data source or filename (e.g., `manual_test.log`), and confirm events arrive.

---

## 5) Next step (optional): add **real-time** streaming via rsyslog

Once file pickup over SSH is running, consider enabling **TCP syslog** (XPLG listener on port 1468) and forward:

* Runner service events from **journald** via rsyslog → XPLG (live).
* Any custom pipeline summaries you emit with `logger` at the end of jobs.

This gives you both **file history** (SSH) and **live events** (syslog). We can wire this later; XPLG’s “Over SSH” path already gets you started today.

---

## 6) Recap & choices

* **WHAT**: GitHub runner logs in `_diag/*.log`; service logs in journald. ([GitHub Docs][2])
* **WHY**: `_diag` = job detail; journald = service health.
* **HOW (now)**: XPLG **Over SSH** → point at `_diag/*.log`. ([xpolog.atlassian.net][4])
* **HOW (later)**: add **rsyslog → TCP 1468** for live streaming and journald coverage. ([Rsyslog][1])

If you tell me the exact runner user and install path (e.g., `/home/runner/actions-runner`), I’ll give you an SSH data-source config that matches your environment (path masks, scheduling, and a couple of safe permission commands).

[1]: https://www.rsyslog.com/doc/getting_started.html?utm_source=chatgpt.com "Getting Started with rsyslog - rsyslog documentation"
[2]: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/monitor-and-troubleshoot?utm_source=chatgpt.com "Monitoring and troubleshooting self-hosted runners - GitHub Docs"
[3]: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/configure-the-application?utm_source=chatgpt.com "Configuring the self-hosted runner application as a service"
[4]: https://xpolog.atlassian.net/wiki/spaces/XPOL6/pages/1607435690/?utm_source=chatgpt.com "Adding a Log to XpoLog - XPLG 6 - XPLG Technical Documentation"
[5]: https://xpolog.atlassian.net/wiki/spaces/XPOL/pages/1594132668/Adding%2Ba%2BLog%2BOver%2BSSH?utm_source=chatgpt.com "Adding a Log Over SSH - XPLG - XPLG Technical Resources"

- [Go to Top](#nav)
---




---

## Where do GitHub self-hosted runner logs live? (WHAT)

On a Linux runner:

* **Per-job diagnostic logs** are written by the runner into the runner install directory’s **`_diag/`** folder. Filenames often start with `Worker_…`. This is the easiest file target for pickup. ([GitHub Docs][2])
  Example path (adjust user/install dir):
  `/home/<runner_user>/actions-runner/_diag/*.log`

* If you installed the runner **as a service** via `./svc.sh`, the **service output** also goes to **systemd journal** (view with `journalctl -u 'actions.runner*'`). That’s not a file; if you want those logs in XPLG, either export them to a file or forward them with rsyslog later. ([GitHub Docs][3])

**Why these two matter (WHY):**

* `_diag/*.log` gives you job/run-level details your pipelines produce.
* journald gives you service health events (runner starting, stopping, errors).

---


---

## 4) Sanity checks (so you know it’s working)

On the runner VM:

```bash
# See latest per-job logs being created
ls -ltr /home/<runner_user>/actions-runner/_diag | tail

# Generate a tiny test entry in a new diag file (optional)
echo "$(date -Is) test-from-runner" >> /home/<runner_user>/actions-runner/_diag/manual_test.log
```

Back in XPLG:

* Open **Search**, filter by your new data source or filename (e.g., `manual_test.log`), and confirm events arrive.

---

## 5) Next step (optional): add **real-time** streaming via rsyslog

Once file pickup over SSH is running, consider enabling **TCP syslog** (XPLG listener on port 1468) and forward:

* Runner service events from **journald** via rsyslog → XPLG (live).
* Any custom pipeline summaries you emit with `logger` at the end of jobs.

This gives you both **file history** (SSH) and **live events** (syslog). We can wire this later; XPLG’s “Over SSH” path already gets you started today.

---

## 6) Recap & choices

* **WHAT**: GitHub runner logs in `_diag/*.log`; service logs in journald. ([GitHub Docs][2])
* **WHY**: `_diag` = job detail; journald = service health.
* **HOW (now)**: XPLG **Over SSH** → point at `_diag/*.log`. ([xpolog.atlassian.net][4])
* **HOW (later)**: add **rsyslog → TCP 1468** for live streaming and journald coverage. ([Rsyslog][1])

If you tell me the exact runner user and install path (e.g., `/home/runner/actions-runner`), I’ll give you an SSH data-source config that matches your environment (path masks, scheduling, and a couple of safe permission commands).

[1]: https://www.rsyslog.com/doc/getting_started.html?utm_source=chatgpt.com "Getting Started with rsyslog - rsyslog documentation"
[2]: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/monitor-and-troubleshoot?utm_source=chatgpt.com "Monitoring and troubleshooting self-hosted runners - GitHub Docs"
[3]: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/configure-the-application?utm_source=chatgpt.com "Configuring the self-hosted runner application as a service"
[4]: https://xpolog.atlassian.net/wiki/spaces/XPOL6/pages/1607435690/?utm_source=chatgpt.com "Adding a Log to XpoLog - XPLG 6 - XPLG Technical Documentation"
[5]: https://xpolog.atlassian.net/wiki/spaces/XPOL/pages/1594132668/Adding%2Ba%2BLog%2BOver%2BSSH?utm_source=chatgpt.com "Adding a Log Over SSH - XPLG - XPLG Technical Resources"

----

#### Git runner logs diagnosis


```json
 xplg@tests  /opt/runners  sed -n "1,150p" ./actions-runner-2/_diag/Worker_20250825-171537-utc.log  
[2025-08-25 17:15:37Z INFO HostContext] No proxy settings were found based on environmental variables (http_proxy/https_proxy/HTTP_PROXY/HTTPS_PROXY)
[2025-08-25 17:15:37Z INFO HostContext] Well known directory 'Bin': '/opt/runners/actions-runner-2/bin.2.328.0'
[2025-08-25 17:15:37Z INFO HostContext] Well known directory 'Root': '/opt/runners/actions-runner-2'
[2025-08-25 17:15:37Z INFO HostContext] Well known config file 'Credentials': '/opt/runners/actions-runner-2/.credentials'
[2025-08-25 17:15:37Z INFO HostContext] Well known directory 'Bin': '/opt/runners/actions-runner-2/bin.2.328.0'
[2025-08-25 17:15:37Z INFO HostContext] Well known directory 'Root': '/opt/runners/actions-runner-2'
[2025-08-25 17:15:37Z INFO HostContext] Well known config file 'Runner': '/opt/runners/actions-runner-2/.runner'
[2025-08-25 17:15:37Z INFO Worker] Version: 2.328.0
[2025-08-25 17:15:37Z INFO Worker] Commit: 6f9a9110adce65f44ca6f7c0ae142b9cc6af937b
[2025-08-25 17:15:37Z INFO Worker] Culture: en-US
[2025-08-25 17:15:37Z INFO Worker] UI Culture: en-US
[2025-08-25 17:15:37Z INFO Worker] Waiting to receive the job message from the channel.
[2025-08-25 17:15:37Z INFO ProcessChannel] Receiving message of length 22123, with hash '2d00744e27c6f2865fadf6f1c3388380ea3a3c4480852ad1a69fb3c36b5fad0d'
[2025-08-25 17:15:37Z INFO Worker] Message received.
[2025-08-25 17:15:37Z INFO Worker] Job message:
 {
  "fileTable": [
    ".github/workflows/ci-orchestrator-ensure-infra-compose-run-pytest-push-to-allure.yml",
    "XpoLog/X8.ODD/.github/workflows/ensure-infra-compose-orchestrator-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/install-python-on-remote-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/install-docker-on-remote-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/create-directories-on-remote-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/upload-files-orch-to-remote-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/clone-repo-json-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/ensure-compose-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/run-pytest-ansi.yml@beacc53819f7260b3667232234955f36baeef97f",
    "XpoLog/X8.ODD/.github/workflows/push-results-to-allure-ansi.yml@beacc53819f7260b3667232234955f36baeef97f"
  ],
  "mask": [
    {
      "type": "regex",
      "value": "***"
    },
    {
      "type": "regex",
      "value": "***"
    },
    {
      "type": "regex",
      "value": "***"
    }
 ],
  "steps": [
    {
      "type": "action",
      "reference": {
        "type": "repository",
        "name": "actions/checkout",
        "ref": "v4",
        "repositoryType": "GitHub"
      },
      "contextName": "__actions_checkout",
      "condition": "success()",
      "id": "bfa610ba-92ad-4a69-b100-b7c565c6c140",
      "name": "__actions_checkout"
    },
    {
      "type": "action",
      "reference": {
        "type": "script"
      },
      "contextName": "loadcfg",
      "inputs": {
        "type": 2,
        "map": [
          {
            "key": "script",
            "value": {
              "type": 3,
              "file": 2,
              "line": 23,
              "col": 14,
              "expr": "format('ansible-playbook -i localhost, -c local \\\n  .ansible/playbooks/load_infra_compose_spec.yml \\\n  --extra-vars \"config_path={0}/{1}\"\n', github.workspace, inputs.infra_path)"
            }
          }
        ]
      },
      "condition": "success()",
      "id": "ea1ff886-b6d9-44aa-a955-45f353680bae",
      "name": "loadcfg"
    }
```

####  explain what you’re seeing in that runner log sample


## What you’re looking at (WHAT)

From your screenshot, these are **GitHub Actions self-hosted runner** `_diag` lines. Typical format is:

```
2025-08-25 17:15:37Z INFO HostContext Well known directory 'Root': '/opt/runners/actions-runner-2'
2025-08-25 17:15:37Z INFO Worker      Receiving message of length 22123, with hash 7a0074...
2025-08-25 17:15:37Z INFO Worker      Job message:
{ ... JSON payload for the job ... }
```

So you’ll see two kinds of content:

1. **Single-line status lines**
   `ISO8601 time (Z) + LEVEL + Component + free-text message`
2. **Multi-line JSON blocks** following a status line (e.g., the “Job message:” body)

## Why parse it (WHY)

* Extract **timestamp**, **level**, **component** (HostContext, Worker, Runner, Listener…), and **message** to filter and alert.
* When a JSON block appears, parse it so fields like `run_id`, `job`, `workflow`, etc., become searchable.

## How to parse safely in XPLG (HOW)

### A) Set multiline rules (so JSON stays with its header)

New event when a line **starts with a timestamp**; otherwise append to the previous event.

* **Multiline → New line starts when regex matches:**
  `^\d{4}-\d{2}-\d{2}[ T]\d{2}:\d{2}:\d{2}Z`
* **Otherwise:** treat as continuation.

This keeps the “Job message:” JSON together with its header line.

### B) Primary extractor for the header line

Use a simple regex that works across your samples:

```
^(?P<ts>\d{4}-\d{2}-\d{2}[ T]\d{2}:\d{2}:\d{2}Z)\s+(?P<level>[A-Z]+)\s+(?P<component>\S+)\s+(?P<message>.*)$
```

Map fields:

* `ts` → **@timestamp / Date**
* `level` → **severity / level**
* `component` → **component**
* `message` → **message**

### C) Secondary JSON extractor (on the message body)

Add a second rule **after** the header extractor that runs only if the **event body contains JSON**. Two ways:

* **System JSON parser** (preferred): apply JSON parsing if the event contains a `{` after the header.
  Result: JSON keys (e.g., `job`, `workflow`, `repository`, `runId`) become fields.

* **Regex fallback** (if you don’t want full JSON): add targeted extractors, e.g.:

  * Known directory lines:
    `^Well known directory '(?P<dir_name>[^']+)':\s+'(?P<dir_path>[^']+)'`
  * Receive length/hash:
    `^Receiving message of length (?P<len>\d+), with hash (?P<hash>[0-9a-fA-F]+)`

Place JSON rule **after** the header rule and above any narrower regexes.

### D) Normalize levels (optional)

Map `INFO|WARN|ERROR|DEBUG` to your platform’s severity field for easy filters/dashboards.

### E) Test against real samples

In the **Verify Pattern** area:

* Paste \~20 lines with a header, a few normal lines, and one JSON block.
* Ensure one event is created per header (JSON should remain attached).
* Verify that `ts / level / component / message` extract correctly and JSON keys appear when present.

## Minimal step-by-step (matching your UI)

1. **Add Data → Over SSH / Remote Linux** → select your runner VM and path:
   `/home/<runner_user>/actions-runner/_diag/*.log`
2. **Collection/Multiline**: “New entry when line matches” → the timestamp regex above.
3. **Parser**:

   * Add **Custom Pattern (Regex)** → the header regex; map fields.
   * Add **System Pattern → JSON** (or custom JSON detect) to parse bodies.
   * (Optional) Add the two small regex extractors for directory and “Receiving message…” lines.
4. **Save** → open **Search** and check new fields.

## Sanity checks you can run right now

* On the runner VM:

  ```bash
  echo "$(date -Is) INFO Tester Job message:" >> /home/<runner_user>/actions-runner/_diag/manual_test.log
  printf '{ "pipeline":"demo","job":"lint","status":"ok" }\n' >> /home/<runner_user>/actions-runner/_diag/manual_test.log
  ```
* In XPLG Search: confirm one event whose message contains both the header and JSON, and that `pipeline/job/status` appear as fields.

---

If you want, paste 10–15 raw lines from a real `_diag` log (remove secrets), and I’ll give you the exact regex + JSON rule order tailored to your sample.


You want me to: (1) make sense of the runner `_diag` log you pasted, and (2) give you a clear, step-by-step parsing recipe for XPLG (using the SSH file source you’re already on).

Here’s exactly what your sample shows and how to parse it.

---

# What the log is (from your sample)

Each **event header** is a single line in this shape:

```
[2025-08-25 17:15:37Z INFO HostContext] Well known directory 'Root': '/opt/runners/actions-runner-2'
[2025-08-25 17:15:37Z INFO Worker] Job message:
```

* `2025-08-25 17:15:37Z` → UTC timestamp (the trailing `Z`).
* `INFO` → level.
* `HostContext` / `Worker` / `ProcessChannel` → component.
* The remainder → human message.

Sometimes the header is followed by a **multi-line JSON block**, e.g. after `Job message:`:

```
[... INFO Worker] Job message:
{
  "fileTable": [ ... ],
  "steps": [ ... ],
  ...
}
```

So you need **multiline grouping** (so the JSON stays attached to its header), then a **header extractor**, and then a **JSON extractor** applied to the message body (when present).

---

# Parsing recipe (copy into XPLG)

## 1) Multiline grouping

* **New event when line matches (regex):**

```
^\[\d{4}-\d{2}-\d{2}[ T]\d{2}:\d{2}:\d{2}Z [A-Z]+ [^\]]+\]
```

* **Otherwise:** append to previous line (so the JSON block joins the header).

## 2) Header extractor (regex)

Add a custom regex pattern:

```
^\[(?P<ts>\d{4}-\d{2}-\d{2}[ T]\d{2}:\d{2}:\d{2}Z) (?P<level>[A-Z]+) (?P<component>[^\]]+)\] (?P<message>.*)$
```

Map fields:

* `ts`      → Date/Time (UTC)
* `level`   → Level / Severity (string)
* `component` → Component (string)
* `message` → Message (string; will include “Job message:” when present)

## 3) JSON extractor (conditional)

Add a second rule **after** the header regex:

* **Apply when:** event body contains a JSON block (e.g., `message` contains `{` OR the full event text matches `(?s)Job message:\s*\{`).
* **Extractor:** JSON (system pattern), **source** = the part **after** the header (XPLG usually applies to the whole event; that’s fine—JSON parser will ignore non-JSON lines).
* **Behavior:** enable flattening (if available) so arrays like `fileTable[]` and `steps[]` become fields (multi-value or CSV).

This turns keys like `steps[].reference.type`, `steps[].contextName`, etc., into searchable fields when a job payload is present.

## 4) Helpful micro-extractors (optional)

For quick filters without full JSON:

* **Known directory lines**

  ```
  ^Well known directory '(?P<dir_name>[^']+)':\s+'(?P<dir_path>[^']+)'
  ```

* **Receiving message length / hash**

  ```
  ^Receiving message of length (?P<len>\d+), with hash '(?P<hash>[0-9a-fA-F]+)'
  ```

Put these **after** the header regex and **before** the JSON rule (they only hit non-JSON message lines).

---

# How to test it (right now)

1. In **Verify Pattern**, paste 20–30 lines from your sample, including:

   * a few normal header lines,
   * the `Job message:` header,
   * the opening `{` and several JSON lines.

2. Confirm:

   * Each header begins a **single event** (the JSON is attached, not split).
   * `ts`, `level`, `component`, `message` are populated.
   * JSON keys appear when the event has a block (e.g., `fileTable`, `steps[0].reference.type`, etc.).

3. Save the parser, ingest the file over SSH, and check in **Search**:

   * Filter by `component:Worker` or `message:*Job message*`.
   * Confirm JSON fields are searchable.

---

# Tips / gotchas

* **Timezone:** your timestamps are UTC (`Z`). In the source settings, set the time zone to **UTC** so events land correctly.
* **Rotation:** `_diag` files rotate; the SSH source should rescan new files. If needed, set an appropriate scan interval.
* **Permissions:** make sure the SSH/SFTP user can read `/opt/runners/actions-runner-2/_diag/*.log`.
* **Noise control:** once fields are extracted, you can chart `count by level`, `top components`, and list `steps[].name` from the JSON to see which parts of a workflow run.

---

If you paste 5–10 lines that include the **entire** `Job message:` header plus the **first JSON line** (with any secrets redacted), I’ll tune the JSON condition for your exact text so it only attempts JSON parsing when it should.

----

totally—here’s the quick, practical rundown.

# Syslog UDP vs Syslog TCP (what matters)

**UDP (RFC 5426)**

* **No connection/handshake** → very low overhead, low latency.
* **Best-effort only** → packets can be **dropped, duplicated, or out of order**; sender doesn’t know.
* **Message size** effectively limited by MTU (\~1500 bytes) unless IP fragmentation (frag loss hurts).
* **Great for** lab/local host → collector, or noisy-but-not-critical telemetry.
* **Back-pressure**: none; if receiver is slow, you just lose data.
* **Firewalling**: simple (single port, stateless).

**TCP (RFC 6587 framing)**

* **Connected** stream → delivery with **ACKs, retransmit, ordering**.
* **Back-pressure** works → if the collector slows, the sender blocks/buffers (protects data).
* **Big messages** fine (no MTU issue).
* **TLS (RFC 5425)** possible on top of TCP for encryption/auth.
* **Overhead**: slightly higher (sessions, keepalives), but worth it for reliability.
* **Great for** production or anything you must not lose.

**What to pick here**

* For your **local Fluent Bit → same host container**, UDP is fine for tests.
* For **steady “don’t lose it” ingestion**, prefer **TCP** (or the **HTTP listener** you’re already using, which gives you a 200 response and generally reliable delivery).

---

# “When I edit the listener port in the UI, does it change the container port?”

Short answer: **it changes what XPLG binds *inside* the container**, **it does not modify Docker’s port-publish rules**.

Concretely:

* XPLG (in the container) will listen on the **port number you set** in the UI (in that container’s network namespace).
* Your Docker run/compose currently publishes **host 5514/udp and 5514/tcp → container 5514/udp and 5514/tcp**.
* If you **change the syslog account to a different port** (say 1468) in the UI:

  * The container now listens on **1468**.
  * Your host is **still publishing 5514 → 5514**. Traffic to host:5514 won’t reach the new 1468 listener.
  * Fix it by **either**:

    1. Updating Docker to also publish **1468 (udp/tcp)** to the container, **or**
    2. Pointing your sender (Fluent Bit) directly at the **container IP:1468** (e.g., 172.20.0.2:1468), **or**
    3. Put the listener back on **5514** so it matches the existing mapping.

> Note: TCP and UDP can share the **same numeric port** (e.g., 5514/tcp and 5514/udp) because they are different protocols. But **only one XPLG account per protocol per port** inside the container—two UDP accounts on 5514 will conflict and cause the “port not available” alert you saw.

---

## Quick sanity checks

* See what’s actually listening **inside** the container:

  ```bash
  docker exec xplg-single sh -lc 'ss -lntup | egrep ":5514|:1468|:30303|:30443|:8088" || true'
  ```
* Verify host → container publish rules (you already did via `docker inspect`).
* Test UDP send from host (lands in Syslog UDP account):

  ```bash
  echo "<134>$(date -u) host=$(hostname) msg=test-udp-5514" | nc -u -w1 127.0.0.1 5514
  ```
* Test TCP connectivity (for a Syslog TCP account on 5514):

  ```bash
  nc -vz 127.0.0.1 5514
  ```

If you want, I can sketch the exact changes if you decide to move your Syslog TCP account off 5514, but for tomorrow I’d keep:

* **Syslog UDP** on **5514/udp** (matches compose),
* **Syslog TCP** on **5514/tcp** (also matches compose), and
* keep using the **HTTP listeners** for your two Fluent Bit streams (syslog tail + tomcat/app tail).



---

#### Bottom
------