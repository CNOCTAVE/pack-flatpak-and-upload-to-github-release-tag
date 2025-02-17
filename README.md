<div align="center">

# Pack Flatpak and Upload to Github Release Tag

Fork of [flatpak/flatpak-github-actions](https://github.com/flatpak/flatpak-github-actions) to pack Flatpak and upload to Github release tag.

Build and deploy your Flatpak application using GitHub Actions

<img src="https://github.com/flatpak/flatpak/raw/main/flatpak.png?raw=true" alt="Flatpak logo" />

</div>

## How to use

### Building stage

Add a new workflow by creating a `.yml` file under `.github/workflows` with this content

```yaml
on:
  push:
    branches: [main]
  pull_request:
name: CI
jobs:
  flatpak:
    name: "Flatpak"
    runs-on: ubuntu-latest
    container:
      image: bilelmoussaoui/flatpak-github-actions:gnome-47
      options: --privileged
    steps:
    - uses: actions/checkout@v4
    - uses: CNOCTAVE/pack-flatpak-and-upload-to-github-release-tag@v1
      with:
        bundle: palette.flatpak
        manifest-path: org.gnome.zbrown.Palette.yml
        cache-key: pack-flatpak-and-upload-to-github-release-tag-${{ github.sha }}
```

#### Inputs

| Name | Description | Required | Default |
| ---     | ----------- | ----------- |----|
| `manifest-path` | The relative path of the manifest file  | Required | - |
| `stop-at-module` | Stop at the specified module, ignoring it and all the following ones. Using this option disables generating bundles. | Optional | Build all modules from the manifest file |
| `bundle` | The bundle name  | Optional | `app.flatpak` |
| `build-bundle` | Whether to build a bundle or not | Optional | `true` |
| `repository-name` | The repository name, used to fetch the runtime when the user download the Flatpak bundle or when building the application  | Optional | `flathub` |
| `repository-url` | The repository url, used to fetch the runtime when the user download the Flatpak bundle or when building the application  | Optional | `https://flathub.org/repo/flathub.flatpakrepo` |
| `run-tests` | Enable/Disable running tests. This overrides the `flatpak-builder` option of the same name, which invokes `make check` or `ninja test`. Network and X11 access is enabled, with a display server provided by `xvfb-run`.  | Optional | `false` |
| `branch` | The default flatpak branch  | Optional | `master` |
| `cache` | Enable/Disable caching `.pack-flatpak-and-upload-to-github-release-tag` directory | Optional | `true` |
| `restore-cache` | Enable/Disable cache restoring. If caching is enabled. | Optional | `true` |
| `cache-key` | Specifies the cache key. CPU arch is automatically added, so there is no need to add it to the cache key. | Optional | `pack-flatpak-and-upload-to-github-release-tag-${sha256(manifestPath)}` |
| `arch` | Specifies the CPU architecture to build for | Optional | `x86_64` |
| `mirror-screenshots-url` | Specifies the URL to mirror screenshots | Optional | - |
| `gpg-sign` | The key to sign the package | Optional | - |
| `verbose` | Enable verbosity | Optional | `false` |
| `upload-artifact` | Whether to upload the resulting bundle or not as an artifact | Optional | `true` |
| `github-tag` | If this value is set, the flatpak bundle will be uploaded to this tag of current GitHub repository | Optional | "" |

#### Building for multiple CPU architectures

To build for CPU architectures other than `x86_64`, the GitHub Actions workflow has to either natively be running on that architecture (e.g. on an `aarch64` self-hosted GitHub Actions runner), or the container used must be configured to emulate the requested architecture (e.g. with QEMU).

For example, to build a Flatpak for both `x86_64` and `aarch64` using emulation, use the following workflow as a guide:

```yaml
on:
  push:
    branches: [main]
  pull_request:
name: CI
jobs:
  flatpak:
    name: "Flatpak"
    runs-on: ubuntu-latest
    container:
      image: bilelmoussaoui/flatpak-github-actions:gnome-47
      options: --privileged
    strategy:
      matrix:
        arch: [x86_64, aarch64]
      # Don't fail the whole workflow if one architecture fails
      fail-fast: false
    steps:
    - uses: actions/checkout@v4
    # Docker is required by the docker/setup-qemu-action which enables emulation
    - name: Install deps
      if: ${{ matrix.arch != 'x86_64' }}
      run: |
        dnf -y install docker
    - name: Set up QEMU
      if: ${{ matrix.arch != 'x86_64' }}
      id: qemu
      uses: docker/setup-qemu-action@v2
      with:
        platforms: arm64
    - uses: CNOCTAVE/pack-flatpak-and-upload-to-github-release-tag@v1
      with:
        bundle: palette.flatpak
        manifest-path: org.gnome.zbrown.Palette.yml
        cache-key: pack-flatpak-and-upload-to-github-release-tag-${{ github.sha }}
        arch: ${{ matrix.arch }}
        # if you want to upload Flatpak bundle to a release tag e.g. 1.2.3
        github-tag: 1.2.3
```

#### Building for Automated Tests

As described in the [Inputs](#inputs) documentation, specifying `run-tests: true` will amend the Flatpak manifest to enable Network and X11 access automatically. Any other changes to the manifest must be made manually, such as building the tests (e.g. `-Dtests=true`) or any other options (e.g. `--buildtype=debugoptimized`).

Most developers will want to run tests on pull requests, before merging into the main branch of a repository. To ensure your manifest is building the correct code, you should set the `sources` entry for your project to `"type": "dir"` with the `path` key relative to the manifest's location in the repository.

In the example below, the manifest is located at `/build-aux/flatpak/org.gnome.zbrown.Palette.json`, so the `path` key is set to `../../`. If the manifest were in the project root instead, the correct usage would be `"path": "./"`.

```json
{
    "app-id" : "org.gnome.zbrown.Palette",
    "runtime" : "org.gnome.Platform",
    "runtime-version" : "master",
    "sdk" : "org.gnome.Sdk",
    "command" : "org.gnome.zbrown.Palette",
    "finish-args" : [
        "--share=ipc",
        "--device=dri",
        "--socket=fallback-x11",
        "--socket=wayland"
    ],
    "modules" : [
        {
            "name" : "palette",
            "buildsystem" : "meson",
            "config-opts" : [
                "--prefix=/app",
                "--buildtype=debugoptimized",
                "-Dtests=true"
            ],
            "sources" : [
                {
                    "name" : "palette",
                    "buildsystem" : "meson",
                    "type" : "dir",
                    "path" : "../../"
                }
            ]
        }
    ]
}
```
