# deploy-workflows

Reusable GitHub Actions workflows for putting st44 sites live. One copy, called
by every site, instead of ~110 lines of `deploy.yml` duplicated per repository.

## Why this repository is public

This repository is **public**. That was a deliberate decision, not a default,
and this is the reason:

A **public** repository cannot call a reusable workflow in a **private** one,
even within the same account and even with *Settings → Actions → General →
Access* set to `user`. The run fails immediately with no jobs and no log — there
is no error message to read anywhere, which is why it is written down here.
Measured, not read in the docs:

| Reusable workflow's repo | Calling repository          | Result                              |
| ------------------------ | --------------------------- | ----------------------------------- |
| private                  | private, owned by `tidemann` | ✅ the call resolves and runs        |
| private                  | public, owned by `tidemann`  | ❌ fails at startup, before any job  |
| public (today)           | public, owned by `tidemann`  | ✅ the call resolves and runs        |

Evidence (2026-09-30): an identical two-line probe workflow calling the same
trivial reusable workflow at the same ref failed at startup from the public
`tidemann/food-st44` and succeeded from a private repository in the same
account. An inline control job on the same branch of `food-st44` was green, so
the branch, the trigger and the file were all fine. After this repository was
made public, the same call from `food-st44` went green.

The site repositories are public, so the choice was to make every site private
or to make this repository public. Nothing secret lives here — the workflow
reads its host, user and key from the **calling** repository's secrets, and the
GHCR packages it pushes stay **private**. Public here costs nothing; private
sites would have cost the sites their visibility.

`main` is protected: pull request required, `ci` required, force-push and
deletion blocked.

## What a calling repository needs before its first run

The three secrets must exist on the calling repository **before** the first
call. With `secrets: inherit`, a missing required secret fails the whole run
before any job starts:

```
Error when evaluating 'secrets'.
Secret DEPLOY_KEY is required, but not provided while calling.
```

That error names the missing secrets, so it is legible — unlike the private/
public failure above. `agent-deploy-key` sets all three; run it for the site
first.

## build-and-deploy

`.github/workflows/build-and-deploy.yml` builds the calling repository's image,
pushes it to GHCR tagged by commit SHA, then SSHes to the server and sends the
compose file to the site's own `deploy.sh` on stdin. That script — the forced
command on the site's deploy key — installs the compose file, pulls, recreates,
and gates on health.

The GHCR package **stays private**. CI never ships image bytes to the server and
never hands a registry credential to it: the pull happens on the host, as the
deploy user, whose own docker config holds the credential.

**One ssh call, on purpose.** Each site's deploy key is installed with
`restrict,command="/srv/apps/<app-name>/deploy.sh"`, so it can run that one
script and nothing else — no `scp`, no `docker`, no shell, no reaching into
another site's directory. The workflow names the script in its ssh line even
though the forced command overrides it, so the same run works whether or not the
key has been tightened yet. That is what makes the workflow→key switch order
safe: flip the workflow, deploy green, then tighten the key.

### Calling it

```yaml
name: Deploy

on:
  workflow_dispatch:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  deploy:
    uses: tidemann/deploy-workflows/.github/workflows/build-and-deploy.yml@v1
    with:
      app-name: food-st44
      site-host: food.st44.no
    secrets: inherit
```

Pin `@v1`. Never `@main` — see [Cutting a new v1](#cutting-a-new-v1).

`secrets: inherit` passes the three secrets `agent-deploy-key` already sets on
every site repository, so joining a new site needs no extra secret.

### Inputs

| Input           | Required | Default                    | Meaning                                                            |
| --------------- | -------- | -------------------------- | ------------------------------------------------------------------ |
| `app-name`      | yes      | —                          | Short site name. Image name, compose service and container name, default remote directory and deploy script, concurrency group. |
| `site-host`     | yes      | —                          | Public hostname, no scheme. Checked over HTTPS at the end.          |
| `remote-dir`    | no       | `/srv/apps/<app-name>/infra` | Directory on the server holding the compose file.                 |
| `deploy-script` | no       | `/srv/apps/<app-name>/deploy.sh` | The forced command on the site's key; reads a compose file on stdin, installs it, pulls, recreates and gates on health. |
| `compose-file`  | no       | `infra/docker-compose.yml` | Path to the compose file in the calling repository.                 |
| `image-tag`     | no       | the commit SHA             | Tag to publish and deploy. Override only for a deliberate re-tag.   |
| `verify-forced-command` | no | `false`              | After deploying, use the site key to prove it is really forced to `deploy.sh`: a shell command, a read of another site's compose, and an `scp` must all be refused. Fails the run if any succeeds. Switch on only after the site's key has been re-minted with the forced command. |
| `verify-other-site-path` | no | (empty)              | When `verify-forced-command` is true, the path on the server whose read must be refused — another site's compose file, the cross-site case a leaked key must not reach. |

### Secrets

| Secret        | Meaning                                       |
| ------------- | --------------------------------------------- |
| `DEPLOY_KEY`  | Private SSH key for the deploy user.          |
| `SERVER_HOST` | Hostname or IP of the server.                 |
| `SERVER_USER` | Deploy user on the server.                    |

All three are required, and all three are what `agent-deploy-key` mints and sets.

### What the calling repository must provide

1. A `Dockerfile` at the repository root that builds a container serving
   `GET /healthz` with the body `ok` on port 80.
2. A compose file (default `infra/docker-compose.yml`) that:
   - names the image `${IMAGE}` — this workflow renders the pushed image into
     the file before sending it, so the tag stays immutable and per-commit
     without editing the file;
   - sets `container_name:` **and** the service name to `app-name`, because the
     host's `deploy.sh` health check resolves the service by that name on the
     proxy network;
   - sets a top-level `name:` so two stacks deploying from a directory called
     `infra` do not claim the same compose project;
   - attaches to the **existing external** proxy network and publishes **no**
     host ports — the proxy terminates TLS and reaches the container over that
     network.
3. A `deploy.sh` on the host, installed by `agent-deploy-script <app-name>`
   (or by `agent-docker mkapp <app-name> --deploy`), which is the forced command
   on the site's key. It reads the compose file on stdin, validates it, keeps the
   previous one as `docker-compose.yml.prev`, pulls, recreates, prints the
   container state, then runs the health check and exits non-zero if it fails.
   The workflow's internal health gate lives inside that script — a forced
   command cannot run `docker exec` itself.

A worked example lives in `tidemann/food-st44`, which is the first live caller:
its whole `.github/workflows/deploy.yml` is the five-line block above.

One thing a caller has to get right that is easy to miss: the calling workflow
needs `permissions: packages: write` at its top level. A called workflow cannot
grant itself more than its caller holds, so without it the build job cannot push
to GHCR.

A caller should also stop publishing the image from its own CI. This workflow is
the publisher; a CI job that still pushes the same SHA tag on a push to `main`
makes two publishers race for one tag. Keep CI's build as a smoke test with
`push: false` and drop its `packages: write`.

### What it does, in order

1. **Build and push** the image to `ghcr.io/<owner>/<app-name>:<commit-sha>`.
   Build once — the deploy job promotes this exact tag and never rebuilds.
2. **Render** the compose file, resolving `${IMAGE}` to the pushed image, so the
   file the host installs carries the immutable per-commit tag.
3. **One ssh call** to `/srv/apps/<app-name>/deploy.sh` (the forced command),
   sending the rendered compose file on stdin. That script installs it, keeps
   the previous one as `docker-compose.yml.prev`, pulls, recreates, prints the
   container state, and runs the internal health gate — exiting non-zero if the
   container does not answer, which fails the job. Its output names the previous
   image, which is the rollback anchor.
4. **Check `https://<site-host>/healthz`** for `200 ok` over valid TLS. A green
   container is not a finished deploy.
5. **Prove the key is forced** (only with `verify-forced-command: true`): with
   the site key in hand, attempt `id`, a read of another site's compose file
   (see `verify-other-site-path`), and an `scp` — each must be refused, or the
   run fails. This is what makes a leaked key harmless by construction, and it
   is verified on every deploy, not once.
6. **Write a step summary** naming the image, its digest, the remote path, the
   deploy script, the public health URL and the previous image.

### Rolling back

Every run's step summary names the image that was running before it. To go back,
re-run the workflow with `image-tag` set to that previous SHA — no rebuild,
because the previous image is still in GHCR under its own SHA tag:

```yaml
uses: tidemann/deploy-workflows/.github/workflows/build-and-deploy.yml@v1
with:
  app-name: <app>
  site-host: <site-host>
  image-tag: <previous-sha>
secrets: inherit
```

`image-tag` is the only input meant to be overridden; it exists precisely for a
deliberate re-tag. Fix forward by pushing a new commit, or revert the commit.

## Cutting a new v1

`v1` is a moving tag that callers pin. Moving it changes every site's deploy at
once, so it is a deliberate act, done only after the change is on `main` and CI
is green.

```bash
git switch main && git pull
git tag -f v1
git push -f origin v1
```

Breaking the calling contract — removing an input, adding a required one,
changing what a caller must provide — means cutting `v2` instead and migrating
callers one at a time:

```bash
git tag v2 && git push origin v2
```

Leave `v1` where it is until the last caller has moved.

## CI

`.github/workflows/ci.yml` runs on every push to `main` and every pull request:
`actionlint` (which also shellchecks every `run:` block) and a check that every
third-party action is pinned by SHA. A caller pins a tag here, so an unpinned
action in this repository would silently move under every site at once.
