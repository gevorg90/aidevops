# Hosting service: SSL certificate issue

## Symptom and example incident

Support reports that a Website Builder customer cannot access their website and the browser displays **Your connection is not private**.

| Field | Reported value |
|---|---|
| Website ID | `1046089` |
| Domain | `mercuryiconex.org` |

The recovery procedure reported by the operator was to send the Postman **Create_Certificate** request, wait for a successful response, then restart Nginx on `hosting-1` and `hosting-2`, one at a time.

Use the affected customer's domain for subsequent incidents. The website ID is useful for tracking but is not a field in the certificate request.

## Access and prerequisites

| SSH alias | Operating system | Privilege escalation |
|---|---|---|
| `acme` | Debian | `sudo su` (password-free, as configured) |
| `hosting-1` | FreeBSD | `su root` (password-free, as configured) |
| `hosting-2` | FreeBSD | `su root` (password-free, as configured) |

- Use the existing `.ssh/config` aliases. ACME's supplied IP is `35.85.31.232`.
- Postman collection: `E:/backups/Postman/ACME.postman_collection.json`.
- Architecture and certificate deployment paths: [Hosting service inventory](../inventory/hosting-service.md).
- Obtain explicit confirmation before executing production changes, as required by the repository's `AGENTS.md`. Certificate creation and the planned Nginx restarts are the changes in this procedure.
- Never record private keys, passwords, tokens, or credentials in the incident or repository.

## 1. Gather evidence

1. Confirm the website ID, exact failing hostname, browser error code, and incident time. A privacy warning alone does not establish the cause: check certificate expiration, hostname coverage, and certificate chain validation.
2. Open `https://mercuryiconex.org` and record the certificate details and failure. Check any redirect hostname separately if it also fails.
3. In Postman, set the effective `SERVER_IP` variable to `35.85.31.232`. Ensure an environment variable is not overriding the collection value.
4. Send the collection's **Health Check** request:

   ```http
   GET http://35.85.31.232:5555/health
   Host: www.renderforestsites.com
   ```

   Record the status and response. This verifies that the API responds; it does not verify certificate issuance or deployment to either hosting node.
5. Inspect both hosting nodes before making changes. Run separately for each alias:

   ```sh
   ssh hosting-1
   su root
   hostname
   service nginx status
   nginx -t
   ```

   Repeat with `ssh hosting-2`. If the configuration test fails, stop and investigate the reported error before restarting either node. Inspect the configured Nginx error log for relevant failures.

If ACME-side investigation is needed:

```sh
ssh acme
sudo su
hostname
```

Inspect the deployed application's logs and service definition; the ACME systemd unit name and log location are not supplied here. Do not assume a unit name or restart ACME as part of this procedure.

## 2. Create the certificate in Postman

After production-change confirmation, select **Create_Certificate** and replace the collection's sample domain with the affected domain.

| Setting | Value |
|---|---|
| Method | `POST` |
| URL | `http://{{SERVER_IP}}:5555/api/v1/certificates` |
| Header | `Content-Type: application/json` |
| Body type | raw JSON |

```json
{
  "domain": "mercuryiconex.org"
}
```

Send the request once and wait for completion. Record the HTTP status and response body. The exported collection contains no saved success-response example: require a successful HTTP response and an application response confirming success, with no reported issuance or deployment failures. If the response is ambiguous, inspect ACME logs before proceeding.

On a failure or timeout, investigate issuance and deployment state before retrying; do not repeatedly request certificates or restart Nginx blindly.

Certificate creation deploys certificate files but does not itself generate an HTTPS configuration, according to the hosting inventory. This recovery assumes the site's existing HTTPS configuration is present. If it is missing or incorrect, investigate that separately using the verified website metadata.

## 3. Restart and verify hosting-1

Connect from your workstation:

```sh
ssh hosting-1
su root
hostname
nginx -t
```

Only if the configuration test succeeds, run:

```sh
service nginx restart
service nginx status
```

Verify HTTPS against this specific node before proceeding. From a machine with curl and network access to the node, substitute its verified serving IP below; an SSH management address is not necessarily the serving address:

```sh
curl --resolve mercuryiconex.org:443:HOSTING_1_SERVING_IP --connect-timeout 10 --max-time 30 -v -o /dev/null https://mercuryiconex.org/
```

This command is for a Unix shell with curl; on Windows use `curl.exe` and replace `/dev/null` with `NUL`. Replace the IP placeholder before running. `--resolve` preserves the hostname for TLS SNI and certificate validation while targeting one node. Do not use `-k` or `--insecure`.

Require successful certificate validation and the expected website HTTP response. A running process alone is insufficient. Inspect certificate validity and hostname coverage in a TLS client or browser as needed. If the node fails verification, stop, inspect its Nginx logs and certificate deployment, and keep the second node untouched.

## 4. Restart and verify hosting-2

After `hosting-1` passes verification:

```sh
ssh hosting-2
su root
hostname
nginx -t
```

Only if the configuration test succeeds, run:

```sh
service nginx restart
service nginx status
```

Repeat the node-specific HTTPS check with `HOSTING_2_SERVING_IP`:

```sh
curl --resolve mercuryiconex.org:443:HOSTING_2_SERVING_IP --connect-timeout 10 --max-time 30 -v -o /dev/null https://mercuryiconex.org/
```

Require the same certificate and application checks as for `hosting-1`. Never restart both nodes simultaneously.

## 5. Verify recovery and close the incident

- Open `https://mercuryiconex.org` through normal public DNS and confirm that the intended website loads without a certificate warning.
- Check the final destination if the site redirects, including `www` if used by the customer.
- Confirm both nodes passed direct HTTPS checks and that the presented certificate covers the requested hostname, is within its validity period, and has a trusted chain.
- Record the certificate-request result, restart times and results for each node, verification results, and any remaining errors.
- Report the exact actions taken: certificate creation for the affected domain and sequential Nginx restarts on `hosting-1`, then `hosting-2`. Only state recovery after verification succeeds.

If recovery fails, preserve the errors and escalate with the affected domain, website ID, API response, and per-node findings. Do not delete certificate files, disable SSL, or invoke bulk renewal/deletion endpoints as a fallback.

## Sources and verification scope

This runbook combines the operator's reported workflow and access details, the supplied Postman collection's request definitions, and the existing hosting inventory. Other collection requests are outside this incident procedure. No live requests, SSH connections, certificate changes, or service restarts were performed while writing this document. Serving IPs, deployed API behavior, and live certificate state must be checked during execution.
