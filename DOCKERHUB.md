# TecGate

**A web interface for [Traefik](https://traefik.io/): enter a domain and a target, and
TecGate writes the reverse-proxy configuration, obtains the certificate, keeps the DNS
record straight and filters the traffic — proxy hosts, Let's Encrypt, Hetzner DNS,
containers, a WAF with bot challenges, and request-body inspection with
[Coraza](https://coraza.io/) and the OWASP Core Rule Set.**

It runs Traefik itself, as a child process, and stores everything in one SQLite file. No
external database, no config generator, nothing to compile.

**Full documentation: https://github.com/amslertec/tecgate**

---

## Quick start

```sh
mkdir tecgate && cd tecgate

curl -O      https://raw.githubusercontent.com/amslertec/tecgate/main/docker-compose.yml
curl -o .env https://raw.githubusercontent.com/amslertec/tecgate/main/.env.example

$EDITOR .env      # three values: ACME_EMAIL, LAN_ADDRESS, CERT_NAMES
docker compose up -d
```

Then open `http://<LAN_ADDRESS>:8090` and finish the setup wizard. It creates the first
administrator account — there is no default user and no default password.

**Finish it promptly:** until it is complete, anyone who can reach port 8090 can create that
account.

The second `curl` renames the file as it downloads. `docker compose` only reads a file
called `.env`, and the template is called `.env.example` so that nobody runs the template as
their configuration.

## Three services, one image

`docker-compose.yml` starts all three from this image; they differ only in their entrypoint.

| Service | What it is |
|---|---|
| `tecgated` | The server: interface, API, and Traefik as its child process. Ports 80, 443, 8090, 8091 |
| `dockerwacht` | A guard in front of the Docker socket. No network, read-only, all capabilities dropped, exactly two read calls allowed through |
| `inhaltswache` | Coraza with the OWASP CRS. No published port |

Everything that must survive lives in `./data` — the database, `acme.json`, the
certificates. That directory is your backup.

## Tags

`latest` follows the newest release. Version tags (`0.1.0` and so on) stay put. Pin one with
`TECGATE_VERSION` in `.env` if you would rather decide yourself when the software changes.

## Two things worth knowing before you start

- **Content inspection fails closed.** If `inhaltswache` is down, every host with content
  inspection enabled answers **502** — dead, not unfiltered. That applies in detection mode
  too, and only to hosts where you switched it on.
- **DNS management works with Hetzner and nothing else**, and the program's log lines are
  German even though the interface is German *and* English.

Both, and the rest of the limits, are set out in full in the README on GitHub. That section
is worth the five minutes.

## Licence

TecGate is **MIT**. The bundled **OWASP Core Rule Set is Apache-2.0** and keeps its own
obligations; both licence texts ship inside this image, at
`/usr/share/licenses/tecgate/LICENSE` and `/usr/share/licenses/owasp-crs/LICENSE`.

**The name, the mark and the wordmark are excluded** — a wordmark is a trademark, not
software. Terms: https://github.com/amslertec/tecgate/blob/main/TRADEMARK.md

This is one operator's tooling, published in case it is useful. No support, no release
schedule, no warranty.
