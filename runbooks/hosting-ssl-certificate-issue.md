# Hosting service: SSL certificate issue

Use this procedure when a hosted website displays **Your connection is not private** or has an SSL certificate error.

Example incident: website ID `1046089`, domain `mercuryiconex.org`.

Use the HTTP request definitions in the Postman collection for certificate operations; the Postman app is not required. Do not run Certbot commands manually on the ACME server: the ACME HTTP request also performs database operations that must remain part of the workflow.

## 1. Send the certificate renewal HTTP request

The **Renew_Certificate** request in `E:/backups/Postman/ACME.postman_collection.json` defines the following call:

- Set `SERVER_IP` to `35.85.31.232`.
- Method: `POST`
- URL: `http://{{SERVER_IP}}:5555/api/v1/certificates/renew`
- Header: `Content-Type: application/json`
- Body: raw JSON, using the affected domain:

```json
{
  "domain": "mercuryiconex.org"
}
```

Send the request and wait for a successful response before restarting Nginx.

For example, send the same request directly from PowerShell, replacing the domain for the current incident:

```powershell
Invoke-RestMethod -Method Post `
  -Uri 'http://35.85.31.232:5555/api/v1/certificates/renew' `
  -ContentType 'application/json' `
  -Body '{"domain":"mercuryiconex.org"}'
```

## 2. Restart Nginx on hosting-1, then hosting-2

Use the existing `.ssh/config` aliases. Both hosting servers run FreeBSD, and `su root` is password-free as configured.

From your workstation, connect to `hosting-1`:

```sh
ssh hosting-1
su root
service nginx restart
```

Once the restart completes successfully, open another workstation terminal and repeat for `hosting-2`:

```sh
ssh hosting-2
su root
service nginx restart
```

Restart the servers one at a time. Afterward, open the customer's website and confirm it loads without an SSL warning.
