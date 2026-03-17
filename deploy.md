# Deployment Guide

This repository is served as a static site from a Docker container on a cloud server (e.g., GCP), accessible at `dev.cppdigest.org/boost/`. GitHub Pages remains active solely to redirect legacy visitors to the new URL.

## Overview

```
Push to main
  ├── deploy.yml  →  SSH into server  →  git pull /opt/boost  →  nginx container serves updated files
  └── pages.yml   →  (only if index.html or 404.html changed)  →  GitHub Pages serves redirect-only artifact
```

The site content lives in a git clone at `/opt/boost` on the server. An nginx Docker container bind-mounts that directory read-only, so a `git pull` is all that is needed to deploy content changes — no Docker image rebuild, no container restart.

---

## Repository Files

| File | Purpose |
|---|---|
| `Dockerfile` | Builds the nginx:alpine container image with a custom nginx config |
| `nginx.conf` | Container-internal nginx config — serves files from the bind-mounted directory |
| `docker-compose.yml` | Defines the `boost-site` container, port mapping, and bind mount |
| `.github/workflows/deploy.yml` | CD workflow — SSHs into the server and runs `git pull` on every push to `main` |
| `.github/workflows/pages.yml` | GitHub Pages workflow — deploys redirect-only artifact when `index.html` or `404.html` change |
| `index.html` | Root page — contains a hostname-conditional redirect for GitHub Pages visitors |
| `404.html` | GitHub Pages 404 handler — redirects deep-linked legacy URLs to their GCP equivalents |

---

## Part 1 — GitHub Repository Configuration

### 1.1 Add Repository Secrets

Go to **Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret name | Value |
|---|---|
| `SERVER_HOST` | IP address or hostname of your cloud server |
| `SERVER_USER` | SSH username on the server (e.g. `ubuntu`) |
| `SERVER_SSH_KEY` | Full contents of the private SSH key (the matching public key must be in `~/.ssh/authorized_keys` on the server) |

### 1.2 Switch GitHub Pages Source to GitHub Actions

Go to **Settings → Pages → Build and deployment → Source** and change from **"Deploy from a branch"** to **"GitHub Actions"**.

This is required for `pages.yml` to deploy. Without it the workflow will fail with a permissions error.

> After this change, GitHub Pages will only update when the `pages.yml` workflow runs — which only happens when `index.html` or `404.html` are modified. Routine content pushes (e.g. updating the `develop/` directory) will not trigger a Pages redeploy.

---

## Part 2 — Server Setup (one-time)

SSH into your server and run the following commands once to prepare the environment.

### 2.1 Install Docker and Docker Compose

If not already installed:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
docker compose version
```

### 2.2 Clone the Repository

```bash
sudo git clone https://github.com/CppDigest/cppdigest.github.io.git /opt/boost
```

### 2.3 Set Ownership and Permissions

Transfer ownership to your SSH user so that `git pull` works without `sudo`, and set world-readable permissions so the nginx container user (UID 101 inside `nginx:alpine`) can read the files via the bind mount:

```bash
sudo chown -R $USER:$USER /opt/boost
find /opt/boost -type d -exec chmod 755 {} \;
find /opt/boost -type f -exec chmod 644 {} \;
```

**Why this works:** The `nginx:alpine` worker process runs as the `nginx` user (UID 101). The bind mount is owned by your SSH user, so nginx reads the files as "other". Standard `755` (directories) and `644` (files) permissions grant world-read access. The deploy workflow's `git pull` runs as your SSH user (the owner), so no `sudo` is needed there either.

### 2.4 Build and Start the Container

```bash
cd /opt/boost
docker compose up -d --build
```

### 2.5 Verify the Container is Serving

```bash
curl http://localhost:9102/
```

You should receive the HTML content of `index.html`. If you see an nginx error page, check `docker logs boost-site`.

---

## Part 3 — Host nginx Configuration (one-time)

The server runs a host-level nginx instance that routes subdirectory requests to individual Docker containers. Add a `location` block for `/boost/` inside the existing `server` block for `dev.cppdigest.org`.

```nginx
location /boost/ {
    proxy_pass http://127.0.0.1:9102/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

The trailing slash on `proxy_pass` causes nginx to strip the `/boost/` prefix before forwarding requests to the container. All internal HTML links in this repo use relative paths, so no URL rewriting is needed.

Test and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Verify end-to-end from your local machine:

```bash
curl https://dev.cppdigest.org/boost/
```

---

## Part 4 — Continuous Deployment

After the one-time setup above, all future deployments are automatic.

### How deploy.yml works

On every push to `main` (or manual dispatch), the workflow SSHs into the server and runs:

```bash
cd /opt/boost
git fetch origin main
CHANGED=$(git diff HEAD origin/main --name-only)
git pull origin main
if echo "$CHANGED" | grep -qE "^(Dockerfile|nginx\.conf|docker-compose\.yml)$"; then
  docker compose up -d --build
fi
```

- **Content changes** (HTML files, JSON data): `git pull` updates files on disk; the container's bind mount reflects changes immediately. No restart needed.
- **Infrastructure changes** (`Dockerfile`, `nginx.conf`, `docker-compose.yml`): the container is rebuilt and restarted automatically.

### How pages.yml works

Triggers only when `index.html` or `404.html` are modified. It copies those two files into a temporary `_pages/` directory and uploads that as the GitHub Pages artifact. The full site content (hundreds of HTML files and large JSON data) is never pushed to GitHub Pages.

---

## Part 5 — GitHub Pages Redirect Behaviour

### index.html redirect

A small script near the top of `<head>` in `index.html` fires only when the page is served from `cppdigest.github.io`:

```html
<script>
  if (window.location.hostname === 'cppdigest.github.io') {
    window.location.replace('https://dev.cppdigest.org/boost/');
  }
</script>
```

When served from `dev.cppdigest.org`, the hostname check fails silently and the full page content displays normally. The same file is used for both deployments.

### 404.html deep-link forwarding

GitHub Pages serves `404.html` for any path not present in the Pages artifact (which only contains `index.html` and `404.html`). This handles bookmarked deep links such as `cppdigest.github.io/v1.90/asio.html` by reconstructing the equivalent GCP URL:

```javascript
window.location.replace(
  'https://dev.cppdigest.org/boost' + window.location.pathname
);
```

For example:
- `cppdigest.github.io/v1.90/` → `dev.cppdigest.org/boost/v1.90/`
- `cppdigest.github.io/v1.90/libraries/asio.html` → `dev.cppdigest.org/boost/v1.90/libraries/asio.html`

---

## Part 6 — Validation Checklist

After completing all steps above, run through this checklist:

1. Push a content-only change to `main` (e.g. edit a file in `develop/`)
   - `deploy.yml` should trigger and complete in ~10s
   - `pages.yml` should **not** trigger (confirm in the Actions tab)
2. Browse to `https://dev.cppdigest.org/boost/` and confirm the change is live
3. Push a change to `index.html`
   - Both `deploy.yml` and `pages.yml` should trigger
4. Visit `https://cppdigest.github.io/` and confirm it redirects to `https://dev.cppdigest.org/boost/`
5. Visit `https://cppdigest.github.io/v1.90/` and confirm it redirects to `https://dev.cppdigest.org/boost/v1.90/`
6. Test manual dispatch: go to **Actions → Deploy to GCP → Run workflow**

---

## Troubleshooting

**Container is not serving files**

```bash
docker logs boost-site
docker compose ps
```

Check that `/opt/boost` is world-readable:
```bash
ls -la /opt/boost
```

**deploy.yml fails with "Permission denied" on git pull**

Ensure ownership was transferred in step 2.3:
```bash
ls -la /opt/
sudo chown -R $USER:$USER /opt/boost
```

**pages.yml fails with "Resource not accessible by integration"**

The GitHub Pages source has not been switched to GitHub Actions. See step 1.2.

**nginx returns 502 Bad Gateway for /boost/**

The Docker container is not running or is not bound to port 9102:
```bash
docker compose ps
curl http://localhost:9102/
```

**Host nginx config change not taking effect**

```bash
sudo nginx -t        # check for syntax errors
sudo systemctl reload nginx
```
