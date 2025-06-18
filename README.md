# wipac-dev-publish-image-action

This GitHub Action builds and pushes Docker images to Docker Hub or GitHub Container Registry (GHCR), and optionally requests Singularity image builds or removals on CVMFS.

## Inputs

### Required

| Name     | Description                                                                                                                                                                                                                                            |
|----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `image`  | Fully qualified Docker image name (without tag).<br>Examples: `ghcr.io/foo/bar`, `foo/bar`.                                                                                                                                                            |
| `action` | What to do:<br>- `BUILD` – Build and push Docker image<br>- `CVMFS_BUILD` – `BUILD`, then request CVMFS to build/persist it<br>- `CVMFS_REMOVE` – Remove Singularity image(s) from CVMFS<br>- `CVMFS_REMOVE_THEN_BUILD` – Remove then rebuild on CVMFS |

### Optional

> NOTE:
>
> Many input attributes only apply to certain `action`-based use cases. However, by design, any irrelevant input attributes for a given use case will be ignored. This allows workflows to be flexible, by having only one `uses: WIPACRepo/wipac-dev-publish-image-action` section, with logic around `action`. See the workflow snippet in the [Example Usage section](#example-usage).

#### If using DockerHub...

| Name                 | Required | Description         |
|----------------------|----------|---------------------|
| `dockerhub_username` | ✅        | Docker Hub username |
| `dockerhub_token`    | ✅        | Docker Hub token    |

#### If using the GitHub Container Registry (ghcr.io)...

| Name         | Required | Description                                    |
|--------------|----------|------------------------------------------------|
| `ghcr_token` | ✅        | GitHub token for authenticating with `ghcr.io` |

#### If interacting with CVMFS...

| Name                | Required                           | Description                                                                                                                                  |
|---------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `gh_cvmfs_token`    | ✅                                  | GitHub PAT used to interact with  [`WIPACrepo/build-singularity-cvmfs-action`](https://github.com/WIPACrepo/build-singularity-cvmfs-action/) |
| `cvmfs_dest_dir`    | ✅                                  | CVMFS destination directory for Singularity images                                                                                           |
| `cvmfs_remove_tags` | yes, only if removing CVMFS images | Newline-delimited list of image **tags** to remove from CVMFS (e.g., `latest`, `main-[SHA]`)                                                 |

#### Miscellaneous Build Configuration

| Name              | Required | Description                                                             |
|-------------------|----------|-------------------------------------------------------------------------|
| `free_disk_space` | no       | `true` to make space on GitHub runner before building image             |
| `build_platform`  | no       | Target build platform. Default: `linux/amd64`<br>Example: `linux/arm64` |

## Example Usage

Here are a few examples, out of many possible configurations.

### Static/Single Action

Docker Hub only:

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: WIPACrepo/wipac-dev-publish-image-action@...
        with:
          image: myorg/myimage
          action: BUILD
          dockerhub_username: ${{ secrets.DOCKERHUB_USERNAME }}
          dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
```

ghcr.io + CVMFS:

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: WIPACrepo/wipac-dev-publish-image-action@...
        with:
          image: ghcr.io/myorg/myimage
          action: CVMFS_BUILD
          ghcr_token: ${{ secrets.GITHUB_TOKEN }}
          gh_cvmfs_token: ${{ secrets.CVMFS_PAT }}
          cvmfs_dest_dir: myorg
```

### Dynamic/Flexible Action

ghcr.io + CVMFS:

```yaml
jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - name: Determine action
        id: set_action
        run: |
          if [[ "${{ github.event.inputs.replace_all }}" == "true" ]]; then
            echo "action=CVMFS_REMOVE_THEN_BUILD" >> "$GITHUB_OUTPUT"
          else
            echo "action=CVMFS_BUILD" >> "$GITHUB_OUTPUT"
          fi

      - uses: WIPACrepo/wipac-dev-publish-image-action@...
        with:
          image: ghcr.io/observation-management-service/ewms-condor-benchmarking
          action: ${{ steps.set_action.outputs.action }}
          gh_cvmfs_token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          ghcr_token: ${{ secrets.GITHUB_TOKEN }}
          cvmfs_dest_dir: ewms/observation-management-service/
```
