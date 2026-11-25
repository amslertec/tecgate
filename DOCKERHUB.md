# TecGate

**A web interface for [Traefik](https://traefik.io/): enter a domain and a target, and
TecGate writes the reverse-proxy configuration, obtains the certificate, keeps the DNS
record straight and filters the traffic.**

It runs Traefik itself, as a child process, and stores everything in one SQLite file. No
external database, no config generator, nothing to compile. The interface speaks German
and English.

**Full documentation: https://github.com/amslertec/tecgate**

---

## What it does

| Area | What you get |
|---|---|
| Proxy hosts | Domain to target in one form, load balancing, HTTP/3, maintenance windows, a security score per host |
| Certificates | Let's Encrypt over HTTP-01 or DNS-01, uploaded certificates, self-signed ones for internal hosts, expiry warnings |
| DNS | Hetzner DNS records kept in step with your hosts |
| WAF | Rules by address, network, country, rate and path, bot challenges, verified good bots, block lists, IP reputation, a learning mode per host |
| Content inspection | Request bodies checked by [Coraza](https://coraza.io/) with the OWASP Core Rule Set |
| Watching | Fault mails, a weekly report, a certificate transparency watch for your domains, statistics, the access log, a change history with undo |
| Access | Users and roles, passkeys, a second factor, API keys with scopes, travel access for trips abroad |
| Standby | A second TecGate stays in sync and takes over a shared address when the first one fails |
| Appearance | Five designs, accent colours and fonts, chosen per user or fixed by the administrator |

## Quick start

```sh
mkdir tecgate && cd tecgate

curl -O      https://raw.githubusercontent.com/amslertec/tecgate/main/docker-compose.yml
curl -o .env https://raw.githubusercontent.com/amslertec/tecgate/main/.env.example

$EDITOR .env      # two values: ACME_EMAIL, CERT_NAMES
mkdir -p data && sudo chown -R 1000:1000 data
docker compose up -d
```

Then open `http://<this machine's address>:8090` and finish the setup wizard. It creates the
first administrator account: there is no default user and no default password.

**Finish it promptly:** until it is complete, anyone who can reach port 8090 can create that
account.

The container runs as uid 1000, which is why `data` has to belong to it. The second `curl`
renames the template to `.env`, the only file `docker compose` reads.

## Three services, one image

`docker-compose.yml` starts all three from this image; they differ only in their entrypoint.

| Service | What it is | Ports |
|---|---|---|
| `tecgated` | The server: interface, API, and Traefik as its child process | 80, 443 (TCP and UDP for HTTP/3), 8090, 8091 |
| `contentguard` | Coraza with the OWASP Core Rule Set | none published |
| `vrrp` | keepalived for a standby pair; idle on a single instance | host network |

No container mounts the Docker socket, and TecGate never talks to the Docker daemon.
Everything that must survive lives in `./data`: the database, `acme.json`, the
certificates. That directory is your backup.

## Tags

`latest` follows the newest release. Version tags (`1.0.0` and so on) stay put. Pin one with
`TECGATE_VERSION` in `.env` if you would rather decide yourself when the software changes.
Every image is scanned with Trivy before it is pushed and carries an SBOM and provenance
attestations.

## Worth knowing before you start

- **Content inspection fails closed.** If `contentguard` is down, every host with content
  inspection enabled answers **502**: dead, not unfiltered. That applies in detection mode
  too, and only to hosts where you switched it on.
- **DNS management works with Hetzner and nothing else.**
- **A standby pair needs both machines in the same network** and a free address for the
  shared IP. Update the standby first, then the primary.

These, and the rest of the limits, are set out in full in the README on GitHub.

## Licence

TecGate is under the **TecGate Software License 1.0**: free to use, privately and
commercially, on as many machines and hosts as you like. Reselling or redistributing it,
modified images, decompiling and offering it to others as a hosted service are not
allowed. The full text ships inside this image at `/usr/share/licenses/tecgate/LICENSE`.

The bundled **Traefik is MIT** and the **OWASP Core Rule Set is Apache-2.0**; both keep
their own licences (`/usr/share/licenses/traefik/LICENSE`,
`/usr/share/licenses/owasp-crs/LICENSE`).

**The name, the mark and the wordmark are excluded**: a wordmark is a trademark, not
software. Terms: https://github.com/amslertec/tecgate/blob/main/TRADEMARK.md

This is one operator's tooling, published in case it is useful. No support, no release
schedule, no warranty.
