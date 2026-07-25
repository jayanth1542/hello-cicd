# hello-cicd

A minimal CI/CD pipeline: every push to `main` is automatically tested, built into a Docker image,
pushed to Docker Hub, and deployed as a running container — with zero manual steps.

**Live check:** `curl http://192.168.1.10:8081` (returns which build is currently deployed)

## Architecture

```
GitHub push
    │
    ▼
Jenkins (Pipeline job, polls SCM)
    │
    ├─ 1. Checkout
    ├─ 2. Test          (npm test — gates everything after it)
    ├─ 3. Build image    (docker build, tagged with Jenkins BUILD_NUMBER + latest)
    ├─ 4. Push to Docker Hub
    └─ 5. Deploy         (ansible-playbook deploy.yml)
                │
                ▼
        Ansible (local connection, same host)
                │
                ├─ pull the exact image:tag just pushed
                ├─ stop/remove the previous container (idempotent — safe to re-run anytime)
                ├─ start the new container on :8081
                └─ poll /health until it returns 200 before declaring success
```

Jenkins owns *build and verify*. Ansible owns *deployed state*. Jenkins never runs `docker run`
directly — it hands off to a playbook that describes the desired end state, which is what makes
re-running the deploy safe and repeatable instead of accumulating drift.

## Stack

- **App:** Node.js / Express — deliberately minimal so every layer of the pipeline stays easy to reason about
- **CI:** Jenkins (Pipeline-as-code via `Jenkinsfile`, not Freestyle UI config)
- **Containerization:** Docker, images tagged by build number (not just `latest`) so a specific build can always be redeployed
- **CD:** Ansible, using `community.docker` modules for idempotent container lifecycle management

## Why Ansible instead of a deploy script in the Jenkinsfile

A shell script's `docker run` fails on the second execution because the container name already
exists. Ansible's `docker_container` module checks current state before acting — running the
playbook twice produces the same end state both times, with no manual cleanup. The same playbook
also generalizes to additional hosts by adding lines to `inventory.ini`, with no logic changes.

## Repo layout

| File | Purpose |
|---|---|
| `app.js` | Express app — `/` shows the running build number, `/health` is polled by Ansible post-deploy |
| `Dockerfile` | Builds the app into a container image |
| `Jenkinsfile` | Pipeline-as-code: the 5 stages above |
| `inventory.ini` | Ansible target — local connection, same host as Jenkins |
| `deploy.yml` | Ansible playbook: pull → replace container → health check |

<img width="800" alt="List of file at start of project" src="https://github.com/user-attachments/assets/33578605-51c1-4d36-ae66-c2a0cdb6400c" />

*Repo at the start — just the app, `Dockerfile`, and `package.json`, before any pipeline tooling was added.*

## Local testing (before touching Jenkins)

Every layer was verified in isolation before wiring it into the pipeline — app first, then the
Docker build — so that when something broke later, it was easy to tell which layer was responsible.

<img width="800" alt="Local Image of Testing" src="https://github.com/user-attachments/assets/7dc3e65e-14dc-4270-9dd3-e3fadbc09c1f" />
<img width="800" alt="Local Terminal Testing" src="https://github.com/user-attachments/assets/2b577739-3334-4545-bbf7-af8253fe6c4c" />
<img width="800" alt="Local Testing of Docker" src="https://github.com/user-attachments/assets/a912b741-6336-498b-b063-d5cf098a4373" />

## Setting up the Jenkins pipeline

<img width="800" alt="Git commit Pushing To repo" src="https://github.com/user-attachments/assets/bc88f5df-23e2-4ad7-ae65-3bd5c653dacd" />
<img width="800" alt="Image of Creating the pipeline Name" src="https://github.com/user-attachments/assets/625a893a-ac18-439a-9aa5-27242f06cbb6" />
<img width="800" alt="Configuring the Pipeline" src="https://github.com/user-attachments/assets/c9561d51-4ebe-484d-b8cf-3b09766dd724" />

Pipeline job (not Freestyle) pointed at this repo's `Jenkinsfile` via "Pipeline script from SCM" —
this is what makes the pipeline itself version-controlled rather than click-configured.

## Failure points hit while building this (and the actual fixes)

Two real failures happened on the first two build attempts. Both are kept here rather than hidden,
because debugging them is a bigger part of this project than the parts that worked first try.

| Symptom | Root cause | Fix |
|---|---|---|
| `npm: not found` in the Test stage | Node wasn't installed on the Jenkins VM itself — the earlier project only ever ran Node inside containers | Installed Node/npm directly on the VM via NodeSource |
| `sudo: a password is required` in the Deploy stage | Playbook had `become: true` by default, but the `jenkins` user already has Docker group access and doesn't need root | Removed `become: true` from `deploy.yml` |
| Ansible would hang non-interactively inside a Jenkins job | Unaccepted SSH host key — Jenkins jobs can't answer an interactive "yes" prompt | Accepted the host key once manually (`ssh -o StrictHostKeyChecking=no localhost`) before wiring it into Jenkins |
| Docker build worked on Mac but risked failing on the VM | `dockerfile` was lowercase; macOS is case-insensitive, Ubuntu is not | Renamed to `Dockerfile` before pushing |

<img width="800" alt="Showing the Loading of First Build" src="https://github.com/user-attachments/assets/51c52a8b-d524-4cfc-b6c6-6a728e29c17b" />
<img width="800" alt="Failure of First Build Because Node was not installed on the vm" src="https://github.com/user-attachments/assets/1191afb2-0800-4f5b-8d20-17b86134d80e" />
<img width="800" alt="Failure of Second Build Because Of become: true — it was not needed because the jenkins user could already run docker without it" src="https://github.com/user-attachments/assets/2367098a-e563-493a-a405-6e1b090ebc77" />

## Proof it works end-to-end

<img width="800" alt="Pipeline Image Showing Build" src="https://github.com/user-attachments/assets/14d8cb1f-79af-46fe-9531-8dd84a2fbb7d" />
<img width="800" alt="Success Output Of Pipeline" src="https://github.com/user-attachments/assets/44a76133-4738-4b22-8b85-23aee423bd49" />
<img width="800" alt="Terminal output showing the app responding after a successful pipeline run" src="https://github.com/user-attachments/assets/9dccbd29-a9d3-4758-9c8f-2d6da14311f7" />

Docker Hub before and after the pipeline's push stage ran:

<img width="800" alt="The repo before the Docker image was pushed" src="https://github.com/user-attachments/assets/2267dbc8-e2ff-4d28-a742-31f95f905b72" />
<img width="800" alt="The repo after the Docker image was pushed" src="https://github.com/user-attachments/assets/a13227b9-7989-4819-a079-66547c92870f" />

<img width="800" alt="Final check with everything working, no pipeline running" src="https://github.com/user-attachments/assets/4cb3f14f-cb08-482c-948c-5d480bccb22d" />

Final repo state:

<img width="800" alt="Files at end of project" src="https://github.com/user-attachments/assets/c373c893-6cfe-4848-a47e-0a99f1c417c4" />

## Running it yourself

1. Fork/clone this repo
2. Point a Jenkins Pipeline job at it (`Pipeline script from SCM`, script path `Jenkinsfile`)
3. Add a `dockerhub-creds` credential in Jenkins (username + Docker Hub access token)
4. Ensure the Jenkins agent's user is in the `docker` group and has `ansible` + the `community.docker` collection installed
5. Push a change — the pipeline runs automatically
