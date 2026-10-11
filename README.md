<h1 align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="marke/logo-wortmarke-dunkel.svg">
    <source media="(prefers-color-scheme: light)" srcset="marke/logo-wortmarke-hell.svg">
    <img src="marke/logo-wortmarke-hell.svg" alt="TecGate" width="388">
  </picture>
</h1>

**TecGate puts a web interface on [Traefik](https://traefik.io/): you enter a domain and a
target, and it writes the reverse-proxy configuration, obtains the certificate, keeps the
DNS record straight and filters the traffic — without you editing a YAML file.**

It runs Traefik itself, as a child process of its own, and stores everything in one SQLite
file. There is no external database, no configuration generator to run first, and nothing
to compile.

> ### This repository holds no source code
>
> It holds the documentation, the Compose file and the environment template — that is all.
> TecGate is distributed as a container image on Docker Hub:
> **[`amslertec/tecgate`](https://hub.docker.com/r/amslertec/tecgate)**. If you came here
> looking for the code, it is not here.

---

## Read this before you switch anything on

Two sentences that are cheap to read now and expensive to discover in production. Both are
explained in full under [Limits](#limits-and-caveats), which is the section of this
document worth your time.

1. **Content inspection fails closed.** If the `contentguard` container dies, every host
   that has content inspection enabled answers **502** — it is dead, not unfiltered. That
   is deliberate, it applies in detection mode too, and it means an enabled host depends on
   that container being up.
2. **This is one operator's tooling, published in case it is useful.** One installation, no
   support, no release schedule, no promise that the next image keeps today's behaviour.

---

## What it manages

| Area | What happens there |
|---|---|
| **Proxy hosts** | Domain → target. TLS, several targets with load balancing and sticky sessions, health checks, "internal networks only", switch a host off |
| **Certificates** | Let's Encrypt via HTTP-01 or DNS-01, certificates you upload yourself, and self-signed ones TecGate issues for its own interface |
| **DNS** | Records at Hetzner DNS: read them, write them, and see which of your domains actually point at this machine |
| **Containers** | What runs on this Docker host, and on remote hosts through a small agent |
| **Rules** | The WAF: network, country, rate, path, reputation, bot challenge, internal-only, content inspection |
| **Requests** | What got through, what did not, and why — grouped, searchable, with the real visitor address |
| **Bans** | Your own bans and the ones the rules produced, with escalating durations |
| **Blocklists** | Lists you subscribe to, plus one of your own. Stage 0 of the check chain: a hit costs no lookup |

Plus an **overview** page that summarises all of it, **users** with two roles
(administrator, viewer) and **settings**.

**The interface speaks German and English.** The log does not — see
[Limits](#limits-and-caveats).

---

## Quick start

Three files, four values, one command.

```sh
mkdir tecgate && cd tecgate

curl -O    https://raw.githubusercontent.com/amslertec/tecgate/main/docker-compose.yml
curl -o .env https://raw.githubusercontent.com/amslertec/tecgate/main/.env.example

# The container runs as an unprivileged user, so the data directory has
# to belong to it — see "Why the chown" below.
mkdir -p data && sudo chown -R 1000:1000 data

$EDITOR .env
docker compose up -d
```

### Why the `chown`

TecGate does not run as root inside its container. That is the right way round — code
running in the container is not root in the container either — but it has a price, and
this is it: the directory the container writes to has to belong to the user it runs as.

That user is **uid 1000**, fixed in the image so this instruction can be exact rather
than approximate. It writes three things into `./data`: the database, the Traefik
configuration, and `acme.json` with your ACME account key.

**Upgrading from a version that ran as root?** The `chown` alone is not enough, and this
is where it bites: `/guard` and `/contenttargets` are not folders on your disk but Docker
volumes. They were created while the container was root, so they belong to root, and a
`chown` on `./data` never touches them. TecGate then starts, fails on the first write and
restarts in a loop — with messages that point at SQLite, not at ownership.

Do this instead, once:

```sh
docker compose down
docker volume rm tecgate_contenttargets tecgate_guardsocket
sudo chown -R 1000:1000 ./data
docker compose up -d
```

**Nothing is lost.** Those two volumes hold the guard's socket and the target table, and
TecGate writes both again on the next start. (Volume names are prefixed with your project
directory — `docker volume ls` shows the exact ones if yours differ.)

The second consequence lives in `.env`: `DOCKER_GID`. The container reads
`/var/run/docker.sock` to list your containers, that socket belongs to a group, and root
got in without being told. Find the number with
`getent group docker | cut -d: -f3`. Get it wrong and nothing crashes — the container
page simply stays empty.

**The second `curl` renames the file on the way down.** In this repository the template is
called `.env.example`; `docker compose` only reads a file called `.env`. The `-o` does the
rename — without it you would have a template sitting next to a stack that ignores it.

### The four values you must change first

`.env` ships with placeholders from ranges reserved for documentation. **They work
nowhere, on purpose:** a template that happens to start never gets filled in. All four are
declared in `docker-compose.yml` as `${...:?...}`, so a missing one stops
`docker compose up` with a sentence naming it instead of running on with an empty string.

| In `.env` | What it must become |
|---|---|
| `ACME_EMAIL` | The mail address of your Let's Encrypt account. Expiry warnings go there, and Let's Encrypt does not check that the address exists |
| `LAN_ADDRESS` | The address at which **this machine** is reachable on your own network. **Not** `localhost`, **not** `127.0.0.1` — Traefik runs in a container, and from where it stands `localhost` is the container |
| `CERT_NAMES` | The names the interface's own certificate should carry. Usually the same value as `LAN_ADDRESS`; separate several with commas |
| `DOCKER_GID` | The numeric id of your host's `docker` group — `getent group docker \| cut -d: -f3`. Needed because the container is not root and cannot read the Docker socket without it |

### Then open the interface

```
http://<LAN_ADDRESS>:8090
```

The first visit lands in the setup wizard. It creates the first administrator account —
there is no default user and no default password — and optionally takes a Hetzner API
token, SMTP credentials and a domain for the interface itself. All three can be added later
under *Settings*.

> **Finish the wizard promptly.** Until it is complete, the endpoint that creates the first
> account is open: **anyone who can reach port 8090 can make themselves an administrator.**
> The wizard says so on its first screen. It is the reason to do this on a machine that is
> not yet exposed, or behind a firewall.

Once it is done, `https://<LAN_ADDRESS>:8091` serves the same interface over TLS, with the
self-signed certificate whose names you set in `CERT_NAMES`. Your browser will warn about
it — that is what self-signed means. If you would rather have a real certificate for the
interface, give it a domain under *Settings*; it then becomes a proxy host like any other
and gets a Let's Encrypt certificate.

---

## What you need first

- **A server with Docker** and the Compose plugin (`docker compose`, v2). Nothing else is
  installed on the host.
- **Ports 80 and 443 free** (443 as TCP, and as UDP if you want HTTP/3). Traefik binds them. If something else already listens there —
  another reverse proxy, Apache, a stray nginx — TecGate will not start.
- **A writable directory** for `./data`. It is created on first start.
- **A domain whose DNS you control**, if you want certificates from Let's Encrypt. HTTP-01
  needs the domain to resolve to this machine's public address and port 80 to be reachable
  from the internet.
- **A Hetzner API token**, and only if you want DNS-01 challenges or DNS record management.
  Optional, and Hetzner is the only provider — see [Limits](#limits-and-caveats).
- **About 1.6 GiB of RAM** for the three containers at their ceilings (1 GiB + 512 MiB +
  64 MiB). In practice they sit far below that; the ceilings exist so that a burst cannot
  take the machine with it.

---

## The settings in `docker-compose.yml`

These are the variables the program itself reads, so their names are German. Three of them
get their value from `.env`; the rest have working defaults and you will rarely touch them.

**Not every row is in `docker-compose.yml`.** Those marked *(image)* come from the image's
own `ENV`, those marked *(program)* are the program's built-in default. To change one, add
it to the `environment:` block of the `tecgated` service.

| Variable | Default | What it does |
|---|---|---|
| `ACME_EMAIL` | from `.env` | Contact address of the Let's Encrypt account |
| `LOKALE_DOCKER_ADRESSE` | from `.env` (`LAN_ADDRESS`) | The address at which Traefik reaches this machine's containers |
| `ZERT_NAMEN` | from `.env` (`CERT_NAMES`) | Extra names for the interface's self-signed certificate, comma-separated |
| `DATEN_VERZEICHNIS` | `/data` *(image)* | Database, `acme.json`, certificates, generated Traefik configuration |
| `ADRESSE` | `:8090` *(image)* | Plaintext listener of the interface |
| `ADRESSE_TLS` | `:8091` *(program)* | TLS listener with the self-signed certificate |
| `DOCKER_SOCKET` | `/guard/docker.sock` | Path to the guard's socket — **not** the Docker socket |
| `TRAEFIK_PFAD` | `/usr/local/bin/traefik` *(image)* | The Traefik binary inside the image |
| `LOG_STUFE` | `INFO` | `DEBUG` · `INFO` · `WARN` · `ERROR` |
| `LOG_FORM` | `lesbar` *(program)* | `lesbar` for human-readable lines, anything else for JSON |
| `GOMEMLIMIT` | `450MiB` | Soft target for the Go collector, below the 1 GiB hard limit |
| `INHALT_ZIEL_VERZEICHNIS` | `/contenttargets` | Where the target table for the content watch is written |
| `INHALT_ZAEHLER_QUELLE` | `http://contentguard:8096/zaehler` | Where TecGate fetches the content-inspection counters |
| `INHALT_TREFFER_QUELLE` | `http://contentguard:8096/treffer` | Where it fetches the individual hits |
| `WAF_AUS` | unset *(program)* | Exactly `1` disables the whole WAF. A kill switch from the environment, not from the database — for the case where a rule you cannot reach has locked you out |

The `contentguard` service reads `INHALT_ZIELTABELLE`, `INHALT_ADRESSE` (`:8095`, where
Traefik reaches it) and `INHALT_ZAEHLER_ADRESSE` (`:8096`, where TecGate fetches from).
`dockerguard` reads `DOCKER_SOCKET` and `WACHT_SOCKET`.

`TECGATE_VERSION` in `.env` decides which image tag the stack runs. It is commented out and
defaults to `latest`; pin it if you would rather choose yourself when the software changes.

---

## What the three containers are

All three run the **same image** and differ only in their entrypoint.

| Service | What it is | Network | Ports |
|---|---|---|---|
| `tecgated` | The server: interface, API, and Traefik as its child process | default | 80, 443 (TCP and UDP), 8090, 8091 |
| `dockerguard` | The guard in front of the Docker socket | **none** | none |
| `contentguard` | Coraza with the OWASP Core Rule Set | default | none published |

**Why the guard exists.** Mounting the Docker socket read-only protects the file, not the
API: `POST /containers/create` goes straight through a `:ro` socket, and whoever can do
that owns the host. So the socket is mounted in exactly one container, which has no network
at all, a read-only filesystem, all capabilities dropped, and passes exactly two calls
through: `GET /containers/json` and `GET /info`. `tecgated` reaches it through a Unix
socket on a shared volume. The old defence — "TecGate has no write path" — was a promise
made by a program; this one is made by the kernel.

---

## What TecGate can do, area by area

### Proxy hosts

A host is a set of domains, one or more targets (up to ten), and how to reach them. Per
target: scheme, address, port, a weight, whether it is in rotation, whether to verify its
certificate, and whether it gets the visitor's original host name. Per host: the load
balancing strategy (round robin or power-of-two-choices), sticky sessions, and a health
check with its own path, interval and timeout. There is a switch that makes a host reachable
**from internal networks only** — Traefik enforces that one on its own, before the rest of
the chain runs — and switching a host off leaves visitors a page with status 503 saying so,
rather than a refused connection.

While you type a domain, the wizard suggests DNS records that already exist and offers to
create the missing one, and it can check whether the target actually answers before you
save.

### Certificates

Let's Encrypt over **HTTP-01** (the default: a challenge file over port 80) or **DNS-01**
(a TXT record — which works for hosts with no public address of their own). You can also
upload your own certificate and key, and TecGate issues self-signed ones for its own
interface. The list shows remaining life, issuer, origin and which hosts use each one; ACME
certificates are renewed by Traefik on its own, uploaded ones are your job and the page
says so instead of offering a button that would not work.

### DNS

Records at **Hetzner DNS**. Read them, add them, change them, and see at a glance which of
your domains actually resolve to this machine. Two switches, both **off** by default:
following your public address when it changes, and rewriting your own records to match. They
are separate on purpose — wanting to know that your address changed and wanting your DNS
rewritten are two different wishes.

### Containers

What runs on this Docker host, grouped by Compose stack: image, state, the address a proxy
host could point at, and whether one already does. **Remote hosts** report through a small
agent (`tecgate-agent`) that pushes over HTTPS; you
add a host in the interface, it issues a token once, and you run one command on the remote
machine. The token is shown exactly once and only its hash is stored.

### Rules — the WAF

Eight kinds of rule. Every rule is scoped either to particular hosts or to all of them. What
a rule may *do* depends on its kind, and the server enforces that rather than offering a
combination that would do nothing: `network`, `country`, `path` and `reputation` can
**block** or **allow** (an exception that ends the chain) and can run **armed** or
**observe-only**; `bot` only ever **challenges**; `content` only ever **blocks**, but keeps
the observe mode — that is its detection level; `rate` only blocks and `internal only` only
allows, and both are always armed, because Traefik enforces them and Traefik cannot pretend.

| Kind | What it matches |
|---|---|
| `network` | An address or CIDR range |
| `country` | An ISO country code, from the request's origin |
| `rate` | Requests per interval, with a burst allowance — enforced by Traefik |
| `path` | A pattern in the request path. The dialog offers template groups to adopt and edit — credentials (`/.env`, `/.git/`, `/.ssh/`), backups, WordPress (`/wp-login.php`, `/xmlrpc.php`), admin panels |
| `reputation` | What AlienVault OTX and AbuseIPDB say about the address |
| `bot` | A challenge in the browser — see below |
| `internal only` | Reachable from private networks only — enforced by Traefik |
| `content` | Request body and headers against the OWASP CRS — see below |

**Bot protection** has three modes: a JavaScript proof-of-work sum only; a **ladder** (the
default — everyone gets the sum, and only whoever fails it gets the puzzle); or the puzzle
for everybody. A pass is remembered in a cookie for 60 minutes by default, the challenge
must be solved within 120 seconds, five failed attempts count against you, and the sum costs
16 bits of work. All four are adjustable per rule, and each has bounds the server enforces.
A rule can carry exceptions — address ranges that are never challenged.

**Reputation** is a two-stage lookup, with a cache in front of both. OTX first — it answers
without an API key, on a smaller quota; storing one under *Settings* raises the quota. Only
what OTX flags goes on to AbuseIPDB, and **that stage does need a key**: without one, the
protection runs on OTX alone and the interface says so. The two measure different things on
purpose — OTX collects threat reports that never expire, AbuseIPDB counts reports from the
last 90 days and then forgets — so OTX alone would ban addresses permanently that have been
clean for months. **Nothing is looked up at all unless a rule of this kind is active.** The
remaining daily quota is **read from the service, not estimated**, and when a service goes
quiet the interface names the stage that stopped rather than silently blocking less.

**Bans** escalate: 15 minutes, then 60, then 24 hours by default, with the level decaying
after 7 days of quiet. Private networks are never bannable — that floor is not a setting.

### Blocklists

Stage 0 of the chain: what is on a list costs no network lookup and no quota. Lists you
subscribe to by URL, refreshed on a schedule you set, plus one list of your own you maintain
by hand. Each list carries its own effect — **block** (a hit is banned at once, permanently)
or **suspect** (a hit skips straight to the second reputation stage instead of being
banned).

**Four external lists come pre-configured, and three of them are on out of the box.** This
is the one thing in TecGate that reaches out over the network without you asking for it, so
it is worth knowing before you start:

| List | Effect | State | Refresh |
|---|---|---|---|
| Spamhaus DROP | block | **on** | every 24 h |
| blocklist.de | block | **on** | every 6 h |
| FireHOL Level 1 | suspect | **on** | every 24 h |
| Tor exit nodes | suspect | **off** | every 12 h |

FireHOL is *suspect* rather than *block* because it aggregates other people's lists and
contains private ranges; Tor is off because "arrived over Tor" is not by itself evidence of
anything. Switch any of them off on the Blocklists page.

A list you add is fetched once as a trial and shows you what arrived before it goes live,
and every refresh counts what came and what went. Private ranges in a downloaded list are
skipped and counted, never applied — that floor is not a setting.

### Content inspection

**[Coraza](https://coraza.io/) with the OWASP Core Rule Set 4.29.0**, at paranoia level 1
with an anomaly threshold of 5, running in the `contentguard` container. For a host that has
it enabled, Traefik's service points at the watch instead of at the host's target.

- It inspects the **request**: method, path, query string, headers — and the **beginning of
  the body**, `131072` bytes (128 KiB) by default, adjustable per host between `8192` and
  `1048576`.
- That number is an **inspection depth, not an upload limit**. A 50 MB upload passes
  through complete and unbuffered; only its first 128 KiB were read. The limit exists
  because those bytes sit in memory **per concurrent request**.
- Body forms it understands: URL-encoded forms, multipart, XML and JSON. Anything else —
  video, images, `application/octet-stream` — streams through untouched.
- It never inspects the **response** body, and never the inside of a WebSocket tunnel after
  the upgrade. WebSocket setup is ordinary HTTP and is inspected.
- Three levels per host: **off**, **detect** (log the hit, let the request through) and
  **block**. Going from *off* straight to *block* is refused — by the server, not just by
  the button — because there would be no numbers to show you. Going back down is always one
  click, without a confirmation: loosening is never the dangerous direction.
- Hits are logged with the rule ID, category, score, host, method and path. **Never with the
  matched value** — that value is a password or an OAuth token often enough that logging it
  would be the leak.

### Requests, and the overview

Every request that the chain touched, grouped into visits so that one page load is one line
rather than forty, searchable, with the real visitor address (not Traefik's). The overview
page rolls all of it up: traffic, per-host figures, what needs attention, and what stands
out.

---

## Running it

### Updating

```sh
docker compose pull
docker compose up -d
```

Database migrations run at startup. **Take a backup first** (below) — a migration that has
run once will not run again, so a half-applied upgrade is not something you fix by
restarting.

If you pinned `TECGATE_VERSION`, `docker compose pull` does nothing until you edit that
line. That is the point of pinning.

### Backing up

Everything that matters is in **`./data`**: the SQLite database `daten.db`, `acme.json` with
the private key of your ACME account, the issued certificates, the session key, the
generated Traefik configuration and the access log.

```sh
docker compose down
tar czf tecgate-backup-$(date +%F).tar.gz data/
docker compose up -d
```

**Stop the stack first.** SQLite writes ahead of the database file; copying `daten.db` while
the server runs gets you a file that may be missing its most recent writes. If you cannot
afford the downtime, snapshot the filesystem instead of copying the file.

`acme.json` deserves its own sentence: Traefik keeps it in memory and rewrites it whole
whenever anything changes, and the private key of your ACME account lives in it. **Never
edit it while the stack is running.** Two writers destroy it.

### Logs

```sh
docker compose logs -f tecgated
docker compose logs -f contentguard
docker compose logs -f dockerguard
```

Set `LOG_STUFE: DEBUG` in `docker-compose.yml` for more, `LOG_FORM` to anything other than
`lesbar` for JSON lines. Each service keeps at most 5 × 10 MB of log; without that ceiling
Docker's default would keep every line until the disk filled.

Traefik's own output runs through TecGate's log, at Traefik's own level — an error from
Traefik arrives as an error, not as an info line. That matters more than it sounds: a
rejected Let's Encrypt certificate is a Traefik error, and it used to go past unnoticed.

### When something is wrong

| Symptom | Where to look |
|---|---|
| `docker compose up` stops naming a variable | The value is missing from `.env`. That is the `${...:?...}` form doing its job |
| Nothing on 8090 | `docker compose ps` — did `tecgated` start? Is something else on 80 or 443? |
| A host answers **502** | If it has content inspection enabled, check `contentguard` first. That is what failing closed looks like |
| A host answers **503** with a page saying so | The host is switched off in the interface |
| A certificate never arrives | *Certificates* shows the reason. HTTP-01 needs port 80 reachable from the internet and the domain pointing here; DNS-01 needs a Hetzner token with write access |
| The browser warns about the interface certificate | It is self-signed. Add the name you actually use to `CERT_NAMES`, or give the interface a domain under *Settings* |
| You locked yourself out with a rule | Set `WAF_AUS: "1"` in the `tecgated` environment and restart. It comes from the environment precisely so that it works when the database says otherwise |

---

## Limits and caveats

The honest section. Read it before you decide to run this.

### The one that will cost you an outage

**Content inspection fails closed.** If the `contentguard` container is down — stopped,
crashed, out of memory, or unable to load its rule set — every host with content inspection
enabled answers **502**. It is dead, not unfiltered.

- This is deliberate. The alternative is a host that looks protected and is not.
- **It applies in detection mode too.** "Detect only" is not "fail open"; both modes depend
  on the same container.
- It affects **only** hosts that have it switched on. Everything else keeps running.
- So: enabling content inspection on a host makes that host depend on a second container.
  Decide that consciously, per host.

### What content inspection cannot do

- **It is a layer, never the only one.** Bound parameters make SQL injection impossible; a
  rule set makes it less likely. A pattern set sees strings, not intentions.
- **It sees the first 128 KiB of a body.** Whoever knows that can push a payload behind
  filler. That is not a weakness of this implementation but of body inspection as such —
  the alternative, no limit at all, is a way to exhaust the machine with a single upload.
- **XXE payloads pass.** Coraza's XML processor does not resolve the entity, and CRS 4.29.0
  has no rule against it at paranoia level 1. Measured, not assumed.
- **A body sent with a content type the rule set does not inspect is not inspected** — even
  if the application behind reads it anyway. Every content-type-driven body inspection has
  this, ModSecurity included.
- **Response bodies are never inspected.** Inspecting them would mean buffering them, and a
  buffered video is not a stream.

### Which hosts cannot have it

Three states make content inspection impossible, and the server refuses to enable it rather
than accepting the setting and quietly ignoring it:

- **A host with more than one target.** Traefik's load balancing would then sit in front of
  the watch instead of in front of your targets, and sticky sessions and health checks would
  stop working. Switching a target off is not enough — remove it.
- **A target that does not get the visitor's `Host` header.** The watch finds its target
  through exactly that header.
- **A host with its own health check.** Traefik would be checking the watch instead of the
  target, get a 502, and take your only server out of rotation.

### And the rest

- **One operator, one installation.** TecGate was built for a specific machine and published
  in case it is useful. There is no support, no roadmap, no release schedule, and no promise
  that the next image keeps today's behaviour or today's database schema.
- **DNS works with Hetzner and nothing else.** Not Cloudflare, not Route 53, not a generic
  RFC 2136 backend. If you are elsewhere, everything except the DNS area still works —
  including HTTP-01 certificates — but DNS-01 does not, and neither does record management.
- **The interface speaks German and English. The log speaks German only.** `docker compose
  logs tecgated` gives you lines like `oberfläche erreichbar` and `regelwerk laedt nicht`.
  This is the first thing you see when something breaks, and it is worth knowing before it
  breaks. It is a consequence of the program being written in German throughout.
- **Going from "off" straight to "block" is refused**, for content inspection and by design.
  Observe first, look at the numbers, then arm it. The confirmation shows you the count of
  requests that would have been blocked — the protection is the number, not the exclamation
  mark.
- **The interface is published on every interface of the machine**, on 8090 in plaintext and
  8091 over TLS. On a machine facing the internet, restrict them: bind them to your LAN
  address in `docker-compose.yml`, put a firewall in front, or drop the plaintext port once
  8091 works for you.
- **The setup wizard is open until it is finished.** Anyone who can reach port 8090 before
  you complete it can create the first administrator account.
- **`tecgated` runs as uid 1000 without capabilities** and is PID 1: it supervises Traefik,
  which binds 80 and 443 through `net.ipv4.ip_unprivileged_port_start: 80` in the compose file.
  The data directory and named volumes must belong to uid 1000.
- **The Docker socket is still on the machine.** The guard narrows what TecGate can ask for
  to two read-only calls; it does not remove the socket. A third call is only ever added
  together with the feature that needs it.
- **Rate limiting and "internal only" are enforced by Traefik**, not by TecGate's own check
  chain. They behave slightly differently from the other rule kinds as a result — they are
  earlier and cheaper, and they do not appear in the same counters.
- **Request data is kept for 30 days by default** (adjustable from 1 to 365). It contains IP
  addresses. Per-minute metrics are kept 48 hours, hourly metrics a year.
- **Two things reach out on their own, from the first start.** Three blocklists are active
  out of the box and are fetched on a schedule (Spamhaus DROP, blocklist.de, FireHOL Level 1
  — see [Blocklists](#blocklists)), and the Traefik version check asks `api.github.com` once
  a day whether a newer release exists. Both are on by default and both can be switched off;
  the version check is on because it only reads a public list, and a WAF quietly running an
  outdated Traefik is the bigger risk.
- **Everything else is on request only.** Let's Encrypt when a certificate is due, the
  Hetzner API if you gave it a token, the reputation services only while a rule of that kind
  is active, and lists you added yourself. The "suggest my public address" button in the
  setup wizard asks `icanhazip.com`, once, on that click.
- **No high availability.** One SQLite file, one connection, one machine. There is no
  clustering story and there is not going to be one here.

---

## Licence and mark

**The documentation and the example files in this repository are MIT** — see
[`LICENSE`](LICENSE). There is no source code here, so that is what the licence covers:
`README.md`, `DOCKERHUB.md`, `docker-compose.yml` and `.env.example`. Copy them, change
them, use them in your own setup.

**The image is not covered by it.** `amslertec/tecgate` is distributed from Docker Hub, and
its terms travel inside it: `/usr/share/licenses/tecgate/LICENSE` and
`/usr/share/licenses/owasp-crs/LICENSE`. The bundled **OWASP Core Rule Set is Apache-2.0**
and keeps its own obligations — MIT here does not lift them. The image comes with no
warranty of any kind.

**The name, the mark and the wordmark are not covered either.** A wordmark is a trademark,
not software; released under MIT, anyone could put it on a product of their own. Passing it
on unchanged in order to refer to TecGate is fine — using it as your own mark, or for a
derived product, is not. The terms are in [`TRADEMARK.md`](TRADEMARK.md).
