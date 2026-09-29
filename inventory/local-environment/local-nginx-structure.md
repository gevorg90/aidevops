# Local Nginx Structure and Landing Pages Routing

## Purpose

This document describes the local development Nginx structure and the standard procedure for adding a new `landing-pages` path to a specific local Kubernetes environment.

This is a common DevOps task when a new landing page must be exposed through the corresponding `website-front-end` hostname.

---

## Infrastructure Overview

### Nginx server

The local development environment is exposed through Nginx installed on the Jenkins node:

```text
Host: jnk
IP:   192.168.0.44
```

Nginx site configuration files are stored under:

```text
/etc/nginx/sites-available/*.conf
```

The local development domain is:

```text
*.local.renderforest.com
```

All local development hostnames point to:

```text
192.168.0.44
```

The request flow is generally:

```text
Developer Browser
        |
        v
*.local.renderforest.com
        |
        v
jnk / Nginx
192.168.0.44
        |
        v
Local Kubernetes application
```

---

# Kubernetes Local Environments

Different development environments are deployed into separate Kubernetes namespaces:

```text
local
local-1
local-2
local-3
local-4
...
local-N
```

The namespace number normally matches the development branch number.

| Kubernetes namespace | Git branch |
|---|---|
| `local` | `development` |
| `local-1` | `development-1` |
| `local-2` | `development-2` |
| `local-3` | `development-3` |
| `local-4` | `development-4` |
| `local-N` | `development-N` |

Example for the `renderforest-api` repository:

```text
Kubernetes namespace: local-1
Git branch:           development-1
Hostname:             api-1.local.renderforest.com
```

The base environment does not use a numeric suffix.

Example for `website-front-end`:

```text
Kubernetes namespace: local
Git branch:           development
Hostname:             www.local.renderforest.com
```
For numbered environments, the hostname follows the same environment number.

Example:

```text
local-4
development-4
www-4.local.renderforest.com
```

Always identify the correct Nginx configuration by hostname before making changes.

---

# React Applications and `landing-pages`

Most React applications can have their own local hostname.

The `landing-pages` repository is different.

`landing-pages` runs as a separate application inside Kubernetes, but it is served behind the `website-front-end` hostname.

For the base local environment:

```text
www.local.renderforest.com
```

For a numbered environment such as `local-4`:

```text
www-4.local.renderforest.com
```

Selected URL paths are matched by Nginx and proxied to the corresponding `landing-pages` Kubernetes application.

Example for `local-4`:

```nginx
proxy_pass http://landing-pages-4;
```

Conceptually:

```text
www-4.local.renderforest.com
        |
        v
website-front-end-4 Nginx config
        |
        +---- normal paths ------> website-front-end-4
        |
        +---- landing paths -----> landing-pages-4
```

---

# Common Task: Add a New Landing Page Path

## Example request

Add the following landing page path to `local-4`:

```text
/ai-video-generator
```

The expected result is:

```text
www-4.local.renderforest.com/ai-video-generator
```

and Nginx must proxy that request to:

```text
landing-pages-4
```

---

## Procedure

### 1. Connect to `jnk`

Connect to the Nginx server:

```text
jnk
192.168.0.44
```

---

### 2. Find the correct `website-front-end` Nginx configuration

Do not guess the configuration filename.

Find it by the target hostname.

For `local-4`:

```bash
grep -Rni "www-4.local.renderforest.com" /etc/nginx/sites-available/
```

If necessary, search more broadly:

```bash
grep -Rni "local.renderforest.com" /etc/nginx/sites-available/
```

Confirm that the selected file contains the expected `server_name`.

Example:

```nginx
server_name www-4.local.renderforest.com;
```

Then open the configuration:

```bash
sudo vim /etc/nginx/sites-available/<config-file>.conf
```

---

### 3. Find the `landing-pages` proxy block

Inside the correct `website-front-end` Nginx configuration, search for the upstream corresponding to the target environment.

For `local-4`:

```text
landing-pages-4
```

In `vim`:

```text
/landing-pages-4
```

Or from the shell:

```bash
grep -n "landing-pages-4" /etc/nginx/sites-available/<config-file>.conf
```

The relevant location block looks similar to:

```nginx
location ~ ^(/(ru|jp|tr|ar|fr|es|de|pt|en|it))?/(renderforest-ai-models|ai-movie-generator|pixverse|sora-2|minimax-h3|kling-ai|seedance-2-5|veo-3|ai-movie-generatori|ai-image-generator|youtube-video-ideas|youtube-video-ideas-suggestions|ai-video-generator|ai-video-editor|music-visualizer-videos|ai-cartoon-generator|ai-branding|ai-commercial-generator|reel-maker|social-media-video-editor|new-promo-video|marketing-teams|image-to-video-ai|team/marketing|team/education|team/learning-development|team/it-cybersecurity|ai-music-video-generator|ai-avatar-video-generator|ai-logo-generator|seedance-2|use-cases/marketing|ai-music-generator|ai-news-generator|solutions/marketing|youtube-video-editor|new-ai-video-generator|ai-reel-generator|release-notes|whats-new|industry/professional-services|photo-video-maker|team/human-resources|industry/healthcare|industry/tech|ai-animation-generator)(/.*)?$ {
    proxy_pass http://landing-pages-4;
}
```

---

### 4. Add the new path

The landing-page paths are stored inside the regex group separated by `|`.

For example, if the requested path is:

```text
/ai-video-generator
```

add:

```text
ai-video-generator
```

to the group.

Do **not** add the leading `/` inside the path list.

Correct:

```text
...|ai-image-generator|ai-video-generator|ai-video-editor|...
```

Incorrect:

```text
...|ai-image-generator|/ai-video-generator|ai-video-editor|...
```

Before adding it, check whether the path already exists:

```bash
grep -n "ai-video-generator" /etc/nginx/sites-available/<config-file>.conf
```

If the path is already present in the correct `landing-pages` location block, no change is required.

---

## Language Prefix Behavior

The regex also supports optional language prefixes:

```nginx
(/(ru|jp|tr|ar|fr|es|de|pt|en|it))?
```

Therefore, adding:

```text
ai-video-generator
```

matches both:

```text
/ai-video-generator
```

and language-prefixed URLs such as:

```text
/en/ai-video-generator
/fr/ai-video-generator
/de/ai-video-generator
```

The final part:

```nginx
(/.*)?$
```

also allows subpaths.

For example:

```text
/ai-video-generator/example
/en/ai-video-generator/example
```

---

# Validate the Nginx Configuration

After editing the file, always test the complete Nginx configuration before reloading.

Run:

```bash
sudo nginx -t
```

Expected result should indicate that the syntax is valid and the configuration test is successful.

For example:

```text
syntax is ok
test is successful
```

If `nginx -t` fails:

**Do not reload Nginx.**

Fix the configuration first and run:

```bash
sudo nginx -t
```

again.

---

# Reload Nginx

Only after `nginx -t` succeeds:

```bash
sudo systemctl reload nginx
```

A reload should be used instead of a restart for a normal configuration change.

---

# Verify the New Route

Test the requested path.

Example:

```bash
curl -I https://www-4.local.renderforest.com/ai-video-generator
```

If HTTPS is not used for that local hostname, test with HTTP instead:

```bash
curl -I http://www-4.local.renderforest.com/ai-video-generator
```

Also test a language-prefixed URL when relevant:

```bash
curl -I https://www-4.local.renderforest.com/en/ai-video-generator
```

The request should be handled by the `landing-pages-4` application and should not return an Nginx configuration error.

---

# Environment Mapping Rule

When receiving a request such as:

```text
Add /some-new-page to local-4
```

derive the environment like this:

```text
Requested environment: local-4
Branch:                development-4
website-front-end:     website-front-end-4
Hostname:              www-4.local.renderforest.com
landing-pages target:  landing-pages-4
```

For the base `local` environment:

```text
Requested environment: local
Branch:                development
Hostname:              www.local.renderforest.com
landing-pages target:  landing-pages
```

Always verify the real hostname and upstream from the existing Nginx configuration rather than relying only on the naming convention.

---

# Quick Runbook

For a request:

```text
Add /PATH to local-N
```

follow this sequence:

```bash
# 1. Find website-front-end config by hostname
grep -Rni "www-N.local.renderforest.com" /etc/nginx/sites-available/

# 2. Open the matching configuration
sudo vim /etc/nginx/sites-available/<config-file>.conf

# 3. Find the landing-pages upstream
# In vim:
/landing-pages-N

# 4. Add PATH to the existing regex list.
# Add only the path name, without the leading "/".

# 5. Validate
sudo nginx -t

# 6. Reload only if validation succeeds
sudo systemctl reload nginx

# 7. Verify
curl -I https://www-N.local.renderforest.com/PATH
```

---

# Safety Rules for the AI Agent

When performing this task automatically or semi-automatically:

1. Identify the requested local environment.
2. Derive the expected hostname, but verify it from Nginx.
3. Find the configuration by `server_name`; do not blindly guess the filename.
4. Confirm the target block proxies to the matching `landing-pages` environment.
5. Check whether the requested path already exists.
6. Modify only the intended landing-pages regex.
7. Preserve all existing paths and regex syntax.
8. Never reload Nginx before running `nginx -t`.
9. If `nginx -t` fails, stop and report the error.
10. Use `systemctl reload nginx`, not restart, for this configuration-only task.
11. Verify the requested URL after reloading.
12. Report which configuration file was modified and which path was added.

---

# Example Completion Report

After completing the task, report something similar to:

```text
Added path: /ai-video-generator
Environment: local-4
Hostname: www-4.local.renderforest.com
Upstream: landing-pages-4
Nginx config: /etc/nginx/sites-available/<config-file>.conf

nginx -t: successful
nginx reload: successful
Route verification: successful
```
