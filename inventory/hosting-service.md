# Production Hosting Service

The hosting service serves customer websites. The internal `site-maker-api` application sends HTTP requests to the ACME service to manage website Nginx configurations and obtain Let's Encrypt SSL certificates through Certbot.

## Servers

| Server | Operating system | Role |
|---|---|---|
| ACME server | Debian | Python Flask application that generates configurations, manages certificates, and deploys changes to both Nginx servers |
| nginx-1 | FreeBSD | Stores deployed website configurations and serves hosted websites through Nginx |
| nginx-2 | FreeBSD | Stores deployed website configurations and serves hosted websites through Nginx |

## Shared storage

All three servers have shared storage attached from the Amazon EFS filesystem named **Hosting**.

```text
Hosting/
|-- conf.d/          # Generated website Nginx configurations
|-- ssl/             # Let's Encrypt certificates and private keys for some domains
`-- acme-challenge/  # Temporary files for HTTP .well-known/acme-challenge validation
```

The EFS filesystem is mounted at `/data` on the ACME server.

| EFS directory | ACME server path | Nginx server usage |
|---|---|---|
| `conf.d` | `/data/conf.d` | Configurations are copied with SCP to `/usr/local/etc/nginx/conf.d` on each node |
| `ssl` | `/data/ssl` | Mounted at `/usr/local/etc/nginx/ssl-source` on each node |
| `acme-challenge` | `/data/acme-challenge` | Mounted at `/opt/letsencrypt` on each node |

The Nginx configuration directory is populated by deployment copies. A direct EFS mount for `conf.d` on the Nginx servers has not been specified.

Certificate deployment copies `/usr/local/etc/nginx/ssl-source/<domain>` into `/usr/local/etc/nginx/ssl` on both Nginx nodes. Generated HTTPS configurations reference the local certificate files under `/usr/local/etc/nginx/ssl/<domain>/`.

## Configuration deployment flow

For an HTTP website such as `exam.com` without multiple languages:

1. `site-maker-api` sends a request to the ACME application's `POST /api/v1/nginx/config` endpoint.
2. ACME selects the HTTP configuration template from the request fields.
3. ACME writes the configuration to `/data/conf.d/exam.com.conf` on the shared EFS filesystem.
4. ACME copies the file with SCP to `/usr/local/etc/nginx/conf.d/exam.com.conf` on nginx-1 and nginx-2.
5. ACME connects over SSH and runs `nginx -s reload` on each node.

Example request body (`freeSubdomain` is illustrative):

```json
{
  "domain": "exam.com",
  "freeSubdomain": "exam",
  "isCertified": false,
  "isOnline": true,
  "languages": []
}
```

For online sites, `isCertified: true` selects an HTTPS configuration and `false` selects HTTP. A nonempty `languages` list selects a multilingual template: the first language is the default, and the remaining entries are additional languages. `isOnline: false` deploys an unavailable-site configuration.

The HTTP configuration path also removes existing certificate data and updates the database; it is not a read-only preview operation.

## Certificate flow

1. The internal caller requests a certificate using `POST /api/v1/certificates` with a `domain` field.
2. ACME runs `sh_scripts/acme.sh`, which invokes Certbot using `/data/acme-challenge` as its webroot. Challenge files are available to both Nginx nodes through their shared mount at `/opt/letsencrypt`.
3. Certbot stores the issued certificate under `/etc/letsencrypt/live/<domain>` on ACME. The script copies `fullchain.pem` and `privkey.pem` into `/data/ssl/<domain>` as `<domain>.crt` and `<domain>.pem`.
4. The application updates its database and copies the certificate directory from `ssl-source` into the local `ssl` directory on each Nginx node.
5. HTTPS configuration deployment uses `POST /api/v1/nginx/config` with `isCertified: true`; that deployment copies the configuration and reloads Nginx. Certificate creation itself does not generate the HTTPS configuration.

## API reference

The following routes are defined in the local `site-maker-cert/app.py` checkout reviewed on 2026-09-30. All POST payloads below are JSON.

| Method | Endpoint | Input | Purpose |
|---|---|---|---|
| GET | `/health` | None | Returns `{"Message": "ACME server started"}` |
| GET | `/api/v1/is-cname-configured/<domain>` | Domain in path | Sends an HTTP HEAD request and checks for a `Hosted-by` response header; this is not a direct DNS CNAME lookup |
| POST | `/api/v1/nginx/config` | `domain`, `freeSubdomain`, `isCertified`, `isOnline`; optional `languages` | Generates and deploys HTTP, HTTPS, multilingual, or unavailable-site configuration |
| DELETE | `/api/v1/nginx/config/<domain>` | Domain in path | Removes website configuration, certificate data, and database entry |
| POST | `/api/v1/certificates` | `domain` | Obtains and deploys a certificate |
| POST | `/api/v1/certificates/renew` | `domain` | Runs renewal for one domain, updates the database, and deploys certificate files |
| POST | `/api/v1/nginx/disable-ssl` | `domain`, `freeSubdomain` | Removes SSL data and deploys a standard HTTP configuration; does not handle multilingual configuration |
| POST | `/api/v1/nginx/block` | `domain` and/or `freeSubdomain` | Blocks a website |
| POST | `/api/v1/nginx/unblock` | `domain` and/or `freeSubdomain` | Unblocks a website |
| POST | `/api/v1/nginx/delete_domains_and_subdomains` | `domains` and/or `subDomains` arrays | Performs bulk domain and subdomain cleanup |

The repository's `doc/API_ENDPOINTS.md` has differences from the implementation, including renewal routing, the disable-SSL path, and bulk cleanup field names. Use `app.py` to confirm the current request contract.

## Verification notes

- `/health` confirms that Flask responds; it does not verify EFS, certificate issuance, SCP, or either Nginx node.
- Check configuration deployment and website HTTP/HTTPS behavior on both nodes. Some remote operations return failure summaries without raising exceptions, so an API success response alone does not establish successful deployment to both nodes.
- Verify the served certificate after issuance or renewal, as well as the website response.

## Sources and scope

Server roles, operating systems, EFS name, and mount layout were supplied by the service owner. Implementation details were checked against the sibling `site-maker-cert` repository, particularly `app.py`, `src/createNginxconfigs.py`, `src/remoteManager.py`, `src/block_unblock.py`, and `sh_scripts/acme.sh`.

This inventory documents the supplied production architecture and local source code; the live servers and deployed revision were not inspected.
