# ChanterSync

## Clone super-repo (recursively pulls submodules)
```bash
git clone --recurse-submodules git@github.com:techapostle/ChanterSync.git
cd ChanterSync
```

## Submodules track `main`
All submodules in this repo are configured to track their `main` branch via `.gitmodules`.

## Pull the latest `main` for each submodule
Fast path (uses tracking branch from `.gitmodules`):
```bash
git submodule update --init --remote --recursive --jobs 6
```

Explicit path (ensures each submodule is on `main` before pulling):
```bash
git submodule foreach --recursive '
  git fetch origin && \
  git checkout main && \
  git pull --ff-only origin main
'
```

## Commit updated submodule pointers in the super-repo
After updating submodules to newer commits, the super-repo will have updated pointers. Commit them so collaborators and CI use the same revisions:
```bash
git add .gitmodules */ .
git commit -m "Update submodules to latest main"
```

## Sync (push) the super-repo to remote
Push the updated pointers and `.gitmodules` to this repo's remote:
```bash
git push
```

## (Optional) Push each submodule's `main` to its remote
Only needed if you made changes inside a submodule locally and want to publish them. Requires write access to each submodule remote.
```bash
git submodule foreach --recursive 'git push origin main'
```
