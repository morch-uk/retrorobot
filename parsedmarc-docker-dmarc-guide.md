# Self-Hosted DMARC Reporting with parsedmarc, Elasticsearch and Kibana

A practical guide for running a self-hosted DMARC reporting stack in a
**Debian 13 LXC on Proxmox** using **Docker Compose**, **parsedmarc**,
**Elasticsearch**, and **Kibana**.

This guide also includes a detailed troubleshooting workflow for IMAP
connectivity, TLS, Docker networking, IMAP IDLE, and parsedmarc restart
behaviour.

> **Note:** Versions are pinned for repeatability. Review upstream
> release notes before upgrading.

## Architecture

``` text
Proxmox
└── Debian 13 LXC
    └── Docker Compose
        ├── parsedmarc 11.0.3
        │     └── IMAPS / 993 → DMARC mailbox
        ├── Elasticsearch 8.19.7
        │     └── 9200/tcp (Docker network only)
        └── Kibana 8.19.7
              └── http://LXC-IP:5601
```

## 1. Recommended LXC resources

A reasonable starting point is:

-   2 vCPU
-   6 GB RAM
-   32--64 GB storage, depending on retention
-   Debian 13 (Trixie), amd64
-   Bridged networking with a fixed or reserved IP

Elasticsearch and Kibana are the memory-heavy components. parsedmarc
itself is comparatively lightweight.

## 2. Elasticsearch kernel setting

Check the virtual memory map limit:

``` bash
sysctl vm.max_map_count
```

A suitable value is:

``` text
vm.max_map_count = 1048576
```

If necessary, add the following to `/etc/sysctl.conf`:

``` text
vm.max_map_count=1048576
```

Then apply it:

``` bash
sysctl --system
```

When Docker is running inside an LXC, make sure the setting is effective
in the environment where Elasticsearch actually runs.

## 3. Install Docker Engine and Compose

``` bash
apt update
apt install -y ca-certificates curl

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

cat >/etc/apt/sources.list.d/docker.sources <<'EOF'
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: trixie
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker

docker version
docker compose version
```

## 4. Create the application directory

``` bash
mkdir -p /opt/dmarc/dashboards
cd /opt/dmarc
```

## 5. Create the protected environment file

Create:

``` bash
nano /opt/dmarc/.env
```

Example:

``` ini
IMAP_HOST=mail.example.com
IMAP_PORT=993
IMAP_USER=dmarc-reports@example.com
IMAP_PASSWORD=REPLACE_WITH_REAL_PASSWORD

KIBANA_PORT=5601
ES_JAVA_OPTS=-Xms2g -Xmx2g
```

Protect it:

``` bash
chmod 600 /opt/dmarc/.env
ls -l /opt/dmarc/.env
```

> **Security:** `docker compose config` can expand environment variables
> and display the IMAP password. Do not paste its full output into
> tickets, chats, or public documentation. Protect `.env` and any
> backups containing it.

### Why this guide uses `.env`

A Docker secret can also be used, but a Docker-in-LXC deployment may
encounter permissions such as:

``` text
Cannot read secret file for PARSEDMARC_IMAP_PASSWORD_FILE:
/run/secrets/imap_password (PermissionError)
```

The parsedmarc image runs as a non-root user. A root-only `.env` file is
a simple alternative for a private deployment.

## 6. Create `docker-compose.yml`

Create `/opt/dmarc/docker-compose.yml`:

``` yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.19.7
    restart: unless-stopped
    environment:
      - network.host=0.0.0.0
      - http.host=0.0.0.0
      - node.name=elasticsearch
      - discovery.type=single-node
      - cluster.name=parsedmarc-cluster
      - bootstrap.memory_lock=true
      - xpack.security.enabled=false
      - xpack.license.self_generated.type=basic
      - ES_JAVA_OPTS=${ES_JAVA_OPTS:--Xms2g -Xmx2g}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:9200/_cluster/health?wait_for_status=yellow >/dev/null"]
      interval: 10s
      timeout: 10s
      retries: 30

  kibana:
    image: docker.elastic.co/kibana/kibana:8.19.7
    restart: unless-stopped
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "${KIBANA_PORT:-5601}:5601"
    depends_on:
      elasticsearch:
        condition: service_healthy

  parsedmarc:
    image: ghcr.io/domainaware/parsedmarc:11.0.3
    restart: unless-stopped
    environment:
      PARSEDMARC_IMAP_HOST: ${IMAP_HOST}
      PARSEDMARC_IMAP_PORT: ${IMAP_PORT:-993}
      PARSEDMARC_IMAP_USER: ${IMAP_USER}
      PARSEDMARC_IMAP_SSL: "true"
      PARSEDMARC_IMAP_PASSWORD: ${IMAP_PASSWORD}

      PARSEDMARC_MAILBOX_WATCH: "true"
      PARSEDMARC_MAILBOX_DELETE: "false"

      PARSEDMARC_GENERAL_SAVE_AGGREGATE: "true"
      PARSEDMARC_GENERAL_SAVE_FAILURE: "false"
      PARSEDMARC_GENERAL_SAVE_SMTP_TLS: "true"

      PARSEDMARC_ELASTICSEARCH_HOSTS: http://elasticsearch:9200
      PARSEDMARC_ELASTICSEARCH_SSL: "false"

    depends_on:
      elasticsearch:
        condition: service_healthy

volumes:
  elasticsearch-data:
```

> **Privacy:** Failure/forensic (`RUF`) report storage is disabled above
> because those reports can contain original headers or message content.
> Enable it only after considering the privacy and retention
> implications.

## 7. Validate and start the stack

``` bash
cd /opt/dmarc
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
```

Expected state:

-   Elasticsearch: **healthy**
-   Kibana: **up**, with port `5601` published
-   parsedmarc: **up**

Elasticsearch will normally show:

``` text
9200/tcp, 9300/tcp
```

rather than:

``` text
0.0.0.0:9200->9200/tcp
```

because this configuration deliberately keeps Elasticsearch internal to
Docker.

## 8. Test Elasticsearch and Kibana

### Elasticsearch

This will normally **fail** from the LXC host:

``` bash
curl http://127.0.0.1:9200
```

That is expected because port 9200 is not published.

Instead, test Elasticsearch from another container on the Compose
network:

``` bash
docker exec -it dmarc-kibana-1 curl -s http://elasticsearch:9200
```

A JSON response confirms Elasticsearch is reachable.

> **Security:** Do not publish port 9200 unless you have a specific
> reason. This starter configuration sets
> `xpack.security.enabled=false`, so Elasticsearch itself has no
> authentication.

### Kibana

Check its logs:

``` bash
docker compose logs --tail=100 kibana
```

Then browse to:

``` text
http://LXC-IP:5601
```

Do not expose this unauthenticated Kibana instance directly to the
Internet. Restrict it to a trusted LAN/VPN or protect it with an
authenticated HTTPS reverse proxy.

## 9. Import the parsedmarc dashboard

Download the dashboard matching the pinned parsedmarc release:

``` bash
cd /opt/dmarc

curl -fsSL \
  https://raw.githubusercontent.com/domainaware/parsedmarc/11.0.3/dashboards/opensearch/opensearch_dashboards.ndjson \
  -o dashboards/parsedmarc.ndjson
```

Import it into Kibana:

``` bash
curl -sS -X POST \
  'http://127.0.0.1:5601/api/saved_objects/_import?overwrite=true' \
  -H 'kbn-xsrf: true' \
  --form file=@/opt/dmarc/dashboards/parsedmarc.ndjson
```

Open Kibana and locate the imported dashboard. If it initially appears
empty, widen the dashboard time range.

## 10. parsedmarc mailbox watching

The long-running service should have:

``` yaml
PARSEDMARC_MAILBOX_WATCH: "true"
```

With watch mode enabled, parsedmarc can use **IMAP IDLE** rather than
simply polling the mailbox every few minutes. The mail server must
support and enable IMAP IDLE.

Check parsedmarc:

``` bash
docker compose logs --tail=100 parsedmarc
```

A healthy debug run eventually reaches:

``` text
Watching for email - Ctrl-C once to quit, twice to force
```

------------------------------------------------------------------------

# Troubleshooting

The following tests deliberately work from the bottom of the stack
upward. This is useful when parsedmarc only reports:

``` text
WARNING:imap.py:180:IMAP connection timeout. Reconnecting...
```

Do not immediately disable SSL or assume the password is wrong.

## A. Docker secret `PermissionError`

Possible symptom:

``` text
CRITICAL:cli.py:3079:Cannot read secret file for
PARSEDMARC_IMAP_PASSWORD_FILE: /run/secrets/imap_password (PermissionError)
```

One practical workaround is to use the protected `.env` method described
above:

``` yaml
PARSEDMARC_IMAP_PASSWORD: ${IMAP_PASSWORD}
```

Then recreate parsedmarc:

``` bash
docker compose up -d --force-recreate parsedmarc
```

## B. Test TCP/993 from the LXC

Replace the address with your mail server:

``` bash
nc -vz 192.0.2.10 993
```

A successful result should report that `993 (imaps)` is open.

## C. Test TCP/993 from inside parsedmarc

This distinguishes LXC connectivity from Docker-container connectivity:

``` bash
docker exec -it dmarc-parsedmarc-1 python3 -c \
"import socket; s=socket.create_connection(('192.0.2.10',993),10); print('TCP 993 CONNECTED'); s.close()"
```

Expected:

``` text
TCP 993 CONNECTED
```

## D. Test TLS and certificate validation

Use the IP for the TCP connection but the real certificate hostname for
SNI:

``` bash
docker exec -it dmarc-parsedmarc-1 python3 -c \
"import socket,ssl; s=socket.create_connection(('192.0.2.10',993),10); s=ssl.create_default_context().wrap_socket(s,server_hostname='mail.example.com'); print('TLS CONNECTED:',s.version()); print(s.getpeercert()); s.close()"
```

A successful result proves:

-   TCP connectivity
-   TLS negotiation
-   CA trust
-   certificate validity
-   hostname validation

If your certificate is issued to `mail.example.com`, configure
parsedmarc with that hostname rather than the server's IP address.

## E. Test the IMAP greeting

``` bash
docker exec -it dmarc-parsedmarc-1 python3 -c \
"import socket,ssl; s=socket.create_connection(('192.0.2.10',993),10); s=ssl.create_default_context().wrap_socket(s,server_hostname='mail.example.com'); print('TLS OK'); print(s.recv(4096).decode(errors='replace')); s.close()"
```

A healthy server should return something similar to:

``` text
* OK IMAP4rev1 ...
```

## F. Test IMAP login and mailbox listing

Use the same account configured for parsedmarc:

``` bash
docker exec -it dmarc-parsedmarc-1 python3
```

Then:

``` python
import imaplib
import ssl

m = imaplib.IMAP4_SSL(
    "mail.example.com",
    993,
    ssl_context=ssl.create_default_context()
)

print("CONNECTED:", m.welcome)
print("LOGIN:", m.login("dmarc-reports@example.com", "YOUR_PASSWORD"))
print("MAILBOXES:", m.list()[0])
m.logout()
```

Using an interactive Python session avoids unnecessarily placing the
password in shell history.

If connection, login, and `LIST` all work, basic networking, TLS,
credentials, and IMAP are functioning.

## G. Check whether IMAP IDLE is supported

A server can accept IMAP login while still having IDLE disabled.

Check capabilities:

``` python
import imaplib
import ssl

m = imaplib.IMAP4_SSL(
    "mail.example.com",
    993,
    ssl_context=ssl.create_default_context()
)

m.login("dmarc-reports@example.com", "YOUR_PASSWORD")
print(m.capability())
m.logout()
```

Look for:

``` text
IDLE
```

If the server responds to an IDLE command with something similar to:

``` text
NO IDLE not enabled
```

enable IMAP IDLE on the mail server.

A successful IDLE request should receive a continuation such as:

``` text
+ idling
```

## H. Test using IMAPClient

For a test closer to parsedmarc's IMAP path:

``` bash
docker exec -it dmarc-parsedmarc-1 python3
```

Then:

``` python
from imapclient import IMAPClient

m = IMAPClient(
    "mail.example.com",
    port=993,
    ssl=True,
    timeout=30
)

m.login("dmarc-reports@example.com", "YOUR_PASSWORD")

print("CAPABILITIES:", m.capabilities())
print("FOLDERS:", m.list_folders())
print("SELECT:", m.select_folder("INBOX"))

print("STARTING IDLE")
m.idle()
print("IDLE STARTED")

print(m.idle_check(timeout=10))

print("DONE:", m.idle_done())
m.logout()
```

A successful result should:

1.  advertise `IDLE`;
2.  select `INBOX`;
3.  print `IDLE STARTED`;
4.  return normally from `idle_check`;
5.  terminate IDLE cleanly.

Mailbox updates such as `EXISTS`, `RECENT`, or `EXPUNGE` are normal IMAP
responses.

## I. `watch=false` causes a clean `code 0` exit

Temporarily setting:

``` yaml
PARSEDMARC_MAILBOX_WATCH: "false"
```

can cause the container to finish normally rather than remain a
long-running service.

With:

``` yaml
restart: unless-stopped
```

Docker may then continually restart it:

``` text
parsedmarc-1 exited with code 0 (restarting)
```

Check the exit state:

``` bash
docker inspect dmarc-parsedmarc-1 \
  --format='ExitCode={{.State.ExitCode}} Status={{.State.Status}} Error={{.State.Error}} OOMKilled={{.State.OOMKilled}}'
```

A result such as:

``` text
ExitCode=0 Status=exited Error= OOMKilled=false
```

indicates a clean exit rather than a crash or OOM condition.

The image entrypoint can be inspected with:

``` bash
docker image inspect ghcr.io/domainaware/parsedmarc:11.0.3 \
  --format='Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}}'
```

For the long-running collector, restore:

``` yaml
PARSEDMARC_MAILBOX_WATCH: "true"
```

## J. Run parsedmarc interactively with debug logging

Stop the normal collector:

``` bash
docker compose stop parsedmarc
```

Run an ephemeral debug instance:

``` bash
docker compose run --rm parsedmarc --debug
```

Useful healthy output includes:

``` text
Initializing Elasticsearch client: hosts=['http://elasticsearch:9200'], ssl=False
Found 0 messages in INBOX
Processing 0 messages
Watching for email - Ctrl-C once to quit, twice to force
```

This proves parsedmarc can:

-   initialize its Elasticsearch client;
-   access the IMAP mailbox;
-   process the current mailbox state;
-   enter its long-running watch loop.

Restart the normal service afterwards:

``` bash
docker compose up -d parsedmarc
```

## K. Why `curl 127.0.0.1:9200` fails

If `docker compose ps` shows:

``` text
9200/tcp, 9300/tcp
```

that means those ports are exposed **inside Docker**, not published to
the LXC host.

Test from Kibana instead:

``` bash
docker exec -it dmarc-kibana-1 curl -s http://elasticsearch:9200
```

Use Kibana on the published port `5601` for normal browser access.

------------------------------------------------------------------------

# Quick IMAP timeout checklist

When parsedmarc reports an IMAP timeout, check in this order:

1.  Can the LXC reach TCP/993?
2.  Can the parsedmarc container reach TCP/993?
3.  Does TLS negotiate successfully?
4.  Does certificate validation succeed?
5.  Does the server return an IMAP greeting?
6.  Can the configured account log in?
7.  Can it list/select the mailbox?
8.  Does the server advertise `IDLE`?
9.  Does an actual IDLE request return `+ idling`?
10. Does IMAPClient successfully enter and leave IDLE?
11. Does `parsedmarc --debug` reach `Watching for email`?

This sequence helps avoid changing unrelated SSL, firewall, or
authentication settings.

# Day-to-day commands

``` bash
cd /opt/dmarc

# Status
docker compose ps

# Recent parsedmarc logs
docker compose logs --tail=100 parsedmarc

# Follow parsedmarc
docker compose logs -f parsedmarc

# Restart parsedmarc
docker compose restart parsedmarc

# Restart the whole stack
docker compose restart

# Stop and start
docker compose down
docker compose up -d

# Docker disk usage
docker system df

# Test Elasticsearch from the Docker network
docker exec -it dmarc-kibana-1 curl -s http://elasticsearch:9200
```

# Backups

Elasticsearch data is stored in the `elasticsearch-data` Docker volume.

For long-term Elasticsearch backups, prefer Elasticsearch snapshots
rather than copying a live data directory.

Also back up:

``` text
/opt/dmarc/docker-compose.yml
/opt/dmarc/.env
/opt/dmarc/dashboards/
```

Treat `.env` and backups containing it as secrets.

# Updating

Versions in this guide are deliberately pinned.

When updating parsedmarc:

``` bash
docker compose pull parsedmarc
docker compose up -d parsedmarc
docker compose logs --tail=100 parsedmarc
```

Change the image tag deliberately in `docker-compose.yml` rather than
relying on `latest`. Review upstream release notes and use an
appropriate dashboard export for the version being deployed.

# Final data path

``` text
DMARC mailbox
    │
    │ IMAPS / 993
    │ IMAP IDLE
    ▼
parsedmarc
    │
    │ HTTP / 9200 over Docker network
    ▼
Elasticsearch
    │
    ▼
Kibana
    └── http://LXC-IP:5601
```

# References

-   parsedmarc documentation:
    <https://domainaware.github.io/parsedmarc/>
-   parsedmarc GitHub repository:
    <https://github.com/domainaware/parsedmarc>
-   parsedmarc releases:
    <https://github.com/domainaware/parsedmarc/releases>
-   Docker Engine on Debian:
    <https://docs.docker.com/engine/install/debian/>
-   Elasticsearch self-managed installation:
    <https://www.elastic.co/docs/deploy-manage/deploy/self-managed/installing-elasticsearch>
-   Original archived Debricked article that inspired the stack:
    <https://web.archive.org/web/20210129184913/https://debricked.com/blog/2020/05/14/analyse-and-visualize-dmarc-results-using-open-source-tools/>

## Notes

This guide is intentionally generic. Replace example hostnames, IP
addresses, mailbox names, passwords, and resource allocations to suit
your environment.
