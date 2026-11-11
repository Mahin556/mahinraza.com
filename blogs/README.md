### DEVELOPMENT INSTRUCTIONS

```bash
docker compose up            # foreground, logs visible
docker compose up -d         # background
docker compose logs -f dev
docker compose exec dev bash   # shell inside the container
docker compose down          # stop (add -v to wipe node_modules volume)

```

Open http://localhost:4321.

Run one-off commands (build, add packages) through the container so versions match:

```bash
docker compose run --rm dev npm run build
docker compose exec dev npm i <pkg>
```
```bash
node -v    # should be v24.x (if the tag isn't found, use :22)
npm -v
```

### Scaffold into a temp dir, then merge into the repo

The repo isn’t empty, so don’t scaffold in place. From /workspaces/blog:

```bash
npm create astro@latest /tmp/blog -- --template blog --no-install --no-git --yes
shopt -s dotglob
cp -rn /tmp/blog/* .      # -n: won't overwrite your README/compose.yaml
rm -rf /tmp/blog
npm install
```
`--no-install` matters: it avoids copying a second node_modules over your volume.

```bash
npm run dev -- --host 0.0.0.0    # http://localhost:4321 on Windows
npm run build
git checkout -b post/slug && git commit && git push
gh auth login                     # once, if git push auth isn't forwarded
```

### Open it

Option A, repo in WSL:

```bash
# Ubuntu terminal
cd ~/blog && code .
```

Then **F1 → Dev Containers: Reopen in Container**.

Option B, fastest I/O (repo lives in a Docker volume, no Windows or WSL files involved):

**F1 → Dev Containers: Clone Repository in Container Volume** → paste the GitHub URL.

First build takes a few minutes. After that, the VS Code terminal is a shell inside the container.

### Daily use (all inside the VS Code terminal)

```bash
npm run dev -- --host 0.0.0.0    # http://localhost:4321 on Windows
npm run build
git checkout -b post/slug && git commit && git push
gh auth login                     # once, if git push auth isn't forwarded
```

Rebuild after changing the compose or devcontainer config: **F1 → Dev Containers: Rebuild Container**.

### Git auth

- VS Code copies your `.gitconfig` into the container automatically.
- With HTTPS remotes, the credential helper is forwarded, so push usually works.
- With SSH remotes, the Windows `ssh-agent` service must be running with your key loaded (`Get-Service ssh-agent | Set-Service -StartupType Automatic -PassThru | Start-Service`, then `ssh-add`). If opened from WSL, it uses the WSL agent.
- If neither works, run `gh auth login` inside the container (the HTTPS option is simplest).

### Extending

- More services (db, redis): add them as sibling services in `compose.yaml`. The `dev` container reaches them by service name (`db:5432`).
- More system tools: switch `image:` to a `build:` with a small Dockerfile (`FROM` the same image, `RUN apt-get install ...`).
- Python/FastAPI instead: image `mcr.microsoft.com/devcontainers/python:3.12`, drop the node_modules volume, `postCreateCommand: "pip install -r requirements.txt"`, forward 8000.
- GPU/vLLM: needs `deploy.resources.reservations.devices` and an NVIDIA CUDA base image. Ask and I'll give that variant.

### Gotchas

- Commit `.devcontainer/` and `compose.yaml` so the setup is reproducible on any machine.
- Add `node_modules` and `dist` to `.gitignore` (the Astro template already does).
- If the container can't be reached on 4321, the dev server wasn't started with `--host 0.0.0.0`.
- Keep the repo off `/mnt/c` for Option A, or file watching and installs get slow.