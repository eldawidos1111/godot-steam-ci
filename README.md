# Godot to Steam: Build and Release Pipeline

The GitHub Actions pipeline that builds and ships **Dungeon Potato**, a commercial Godot 4 game, to Steam. Every push to the release branch cross-compiles Windows and Linux exports on an Ubuntu runner and uploads both depots.

> **Paths:** these live at `.github/workflows/` in the real repository. They are kept under `workflows/` here so GitHub does not schedule the cron job or offer a Steam deploy against this sample repo.

## Shape

```
push to release branch ─┐
                        ├─► build (ubuntu-latest)
workflow_dispatch ──────┘     ├─ derive build label from commit subject
   [upload? yes/no]           ├─ install Wine (32 + 64 bit)
                              ├─ rewrite export_presets.cfg paths
                              ├─ export Windows + Linux via firebelley/godot-export
                              └─ upload artifacts
                                        │
                                        ▼
                              deploy (gated)
                                        ├─ validate required secrets, fail fast
                                        ├─ resolve artifact root
                                        ├─ template depot VDFs
                                        └─ game-ci/steam-deploy
```

## Secrets

No Steam identifier is committed: not the credentials, and not the App ID or depot IDs.

```yaml
env:
  STEAM_APP_ID: ${{ secrets.STEAM_APP_ID }}
  STEAM_DEPOT_ID_WINDOWS: ${{ secrets.STEAM_DEPOT_ID_WINDOWS }}
  STEAM_DEPOT_ID_LINUX: ${{ secrets.STEAM_DEPOT_ID_LINUX }}
```

The committed depot descriptors are templates, substituted at deploy time into a `generated/` directory that is never written back to the tree:

```vdf
"DepotBuildConfig"
{
    "DepotID" "__STEAM_DEPOT_ID_WINDOWS__"
    "ContentRoot" "__CONTENT_ROOT_WINDOWS__"
    ...
}
```

The deploy job validates its inputs before doing any work, so a missing secret fails immediately rather than inside `steamcmd` several minutes later:

```bash
required=(STEAM_APP_ID STEAM_DEPOT_ID_WINDOWS STEAM_DEPOT_ID_LINUX)
for name in "${required[@]}"; do
  if [[ -z "${!name:-}" ]]; then
    echo "$name secret is missing." >&2
    exit 1
  fi
done
```

## Release control from the commit message

The build label derives from the commit subject and doubles as the Steam build description. A `{LIVE}` marker in the subject promotes the build to the `public` branch; without it the build uploads and waits for manual promotion in Steamworks.

```bash
if [[ "$SUBJECT" == *"{LIVE}"* ]]; then
  AUTO_SUBMIT="true"
fi
```

```yaml
releaseBranch: ${{ env.AUTO_SUBMIT == 'true' && 'public' || '' }}
```

The label is sanitized into a form safe for both a filesystem path and a Steam build description, falling back rather than emitting an empty string:

```
commit subject → strip {LIVE} → trim → non-alphanumerics to '-' → trim '-.'
              → fall back to branch name → then full SHA → then short SHA
              → truncate to 48 chars
```

## Artifact resolution

Exporter output location depends on the project layout, the runner's workspace root, and how `upload-artifact` flattened the tree. The deploy job probes three candidate bases, then tries an exact label match, then a recursive search, then falls back to "if exactly one directory is here, use it".

Ambiguity is a hard failure that prints what it found, rather than picking a build to ship:

```bash
elif [[ "${#siblings[@]}" -gt 1 ]]; then
  echo "Ambiguous artifact directories under $artifact_base:"
  printf '  %s\n' "${siblings[@]}"
  exit 1
```

Every script block runs under `set -euo pipefail`.

## Cross-compilation

Windows builds are produced on `ubuntu-latest`. Wine is installed with i386 multiarch and its path handed to the export action, keeping the whole matrix on one runner:

```bash
sudo dpkg --add-architecture i386
sudo apt-get install -y wine64 wine32
echo "WINE_PATH=$(which wine)" >> $GITHUB_OUTPUT
```

Export paths are rewritten into `export_presets.cfg` before the export runs, so output is namespaced per build label instead of overwriting a fixed location.

## Gated deploys

`workflow_dispatch` exposes an `upload_to_steam` boolean for build-only smoke tests. Automatic pushes always deploy:

```yaml
if: github.event_name != 'workflow_dispatch' || (github.event_name == 'workflow_dispatch' && github.event.inputs.upload_to_steam == 'true')
```

## Artifact retention

`clear-artifacts.yml` is a nightly cron deleting build artifacts older than two days. Godot exports are large and private-repo artifact storage is billed.

## Files

| File | Purpose |
|------|---------|
| `workflows/steam-upload.yml` | Build and deploy pipeline |
| `workflows/clear-artifacts.yml` | Nightly artifact cleanup cron |
| `steam/steam_app_build.vdf` | App build descriptor template |
| `steam/depot_windows.vdf` | Windows depot template |
| `steam/depot_linux.vdf` | Linux depot template |

## Related

[godot-data-driven-shop](https://github.com/eldawidos1111/godot-data-driven-shop) is a gameplay code sample from the same project.

## License

[MIT](LICENSE). Swap in your own secrets.
