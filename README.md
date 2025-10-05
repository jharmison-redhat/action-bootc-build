# bootc-image-plugin

A GitHub Actions plugin for performing a bootc[^1] image build, rechunk, and publish.

This plugin will perform a rootful podman build into containers-storage, rechunk that image using
`bootc-base-imagectl rechunk`[^2] from inside the image itself, and optionally push the image to a container registry.
It doesn't handle anything related to installation media or disk images, focusing only on building an image suitable for
following for the smallest possible transactional updates.

[^1]: [bootc](https://bootc-dev.github.io/bootc/)

[^2]:
    [bootc-base-imagectl](https://docs.fedoraproject.org/en-US/bootc/building-from-scratch/#_optimizing_container_images)

## Usage

In the simplest case, you could build an image as a step that gets consumed in a follow-on action:

```yaml
name: ci
on:
  push: # will do a build without push on every push on every branch

concurrency: # prevent multiple builds at once
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-rechunk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - id: build
        uses: jharmison-redhat/action-bootc-build@v1
```

Here, with a step id of `build`, you can access the tag or unique Image ID in a follow-on step by using
`${{ steps.build.outputs.tag }}` or `${{ steps.build.outputs.iid }}`, or register them as outputs for the job.

If you want to have the job perform builds and push to a registry on a schedule and after changes to the main branch,
define a workflow with a job definition that includes logging into a registry with secrets and enabling push:

```yaml
name: Build, rechunk, and push image
on:
  schedule:
    - cron: 45 21 */2 * * # every other day, at 21:45 UTC
  push:
    branches:
      - main # you can do more complex branch to tag mapping, but this is the simple case

concurrency: # prevent multiple builds at once
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Log in to quay
        uses: redhat-actions/podman-login@v1
        with:
          registry: quay.io # or any other registry
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - uses: actions/checkout@v5

      - name: Build, rechunk, and push the image
        uses: jharmison-redhat/action-bootc-build@v1
        with:
          push: "true"
          tag: quay.io/myquayorg/bootc-image:latest
          cache-repo: quay.io/myquayorg/bootc-image/cache # optional repository reference to store tags for layer cache
```

## Inputs

> `List` type is a newline-delimited string
>
> ```yaml
> labels: |
>   org.opencontainers.image.title=My Bootc Image
>   org.opencontainers.image.description=My personal image for booting my bare metal systems
> ```

| Name            | Type    | Default                   | Description                                                                                                                                                                                |
| --------------- | ------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `annotations`   | List    |                           | Annotations to apply to the image, only applicable for OCI format images (silently ignored for docker format)                                                                              |
| `arch`          | String  | `amd64`                   | Alternate architecture to build for using qemu-static, regardless of the architecture of the runner                                                                                        |
| `authfile`      | String  |                           | Alternative authfile location to use for podman push and/or layer cache, if using an alternate method to sign in                                                                           |
| `build-args`    | List    |                           | Build arguments to pass to the build, for `ARG` instructions in the Containerfile                                                                                                          |
| `cache-repo`    | String  |                           | An OCI repository reference to use for intermediate layer cache, shortening successive build times                                                                                         |
| `context`       | String  |                           | The build context for the initial image                                                                                                                                                    |
| `file`          | String  |                           | The path to an alternate Containerfile, otherwise auto-detect with podman                                                                                                                  |
| `graphroot`     | String  | `/mnt/containers-storage` | The location to bind-mount over top of the rootful podman graphroot, to provide more storage for a build in the normal GitHub hosted runner configuration                                  |
| `labels`        | List    |                           | Additional labels to apply during the build                                                                                                                                                |
| `max-layers`    | Integer |                           | The maximum number of layers in the rechunked image, otherwise uses whatever is default in `bootc-base-imagectl`                                                                           |
| `no-cache`      | Bool    | `false`                   | Disable the layer cache when building the image                                                                                                                                            |
| `pull`          | String  | `newer`                   | The pull policy for the base image of the initial build                                                                                                                                    |
| `push`          | Bool    | `false`                   | Whether to push the image to the specified tag after rechunking                                                                                                                            |
| `rechunk-image` | String  |                           | An image to use to perform the rechunk, instead of the initial unchunked image itself (must have `bootc-base-imagectl` installed in it!)                                                   |
| `secret`        | List    |                           | A list of secrets, of type file or env, to provide to the build for `RUN` instructions which reference them ("id=mysecret,src=MYSECRET,type=env" or "id=mysecret,src=.mysecret,type=file") |
| `shm-size`      | String  |                           | Override the default size of `/dev/shm` provided on the initial build                                                                                                                      |
| `tag`           | String  |                           | The tag to set for the final image - required when `push` is `true`                                                                                                                        |

## Outputs

| Name       | Type   | Description                                                                                                                                            |
| ---------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `iid`      | String | The podman Image ID of the final chunked image, left in root `containers-storage` after the step runs                                                  |
| `tag`      | String | The tag applied to the final chunked image, left in root `containers-storage` after the step runs - mostly useful if `tag` was unspecified in `inputs` |
| `manifest` | Path   | The path to a JSON dump of the manifest of the final chunked image, useful for follow-on reporting                                                     |
