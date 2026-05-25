# n8n-ffmpeg

A custom Docker image that extends the official [n8n](https://n8n.io) image with `ffmpeg` and `ffprobe` available on `PATH`, so that workflows can run audio/video processing nodes (or shell commands invoking ffmpeg) directly.

## Why this image exists

The official `n8nio/n8n` image **does not allow installing ffmpeg** through the normal route:

- The container runs as a non-root `node` user, so you cannot just `apk add ffmpeg` from inside.
- Even when switching to `root`, the package management tooling in the upstream image is restricted in ways that make a clean install fragile and version-dependent.
- Bind-mounting an ffmpeg binary from the host couples the image to the host's architecture/libraries and breaks portability.

Workflows that need ffmpeg (transcoding, thumbnail extraction, audio extraction, format conversion, etc.) therefore have no first-class way to use it.

## How this image solves it

Instead of fighting the package manager, this image **copies a pre-built statically linked ffmpeg binary** into the n8n image. The binary has no runtime dependencies, so it works regardless of what is or isn't installed in the base image.

The build uses a two-stage Dockerfile:

1. **Stage 1 — builder (`alpine:3.18`)**
   Downloads the static ffmpeg release from [johnvansickle.com](https://johnvansickle.com/ffmpeg/) (a well-known source of static linux ffmpeg builds) and extracts the `ffmpeg` and `ffprobe` binaries.

2. **Stage 2 — final (`n8nio/n8n:<version>`)**
   Starts from the official n8n image and **only** does a `COPY --from=builder` of the two binaries into `/usr/local/bin/`, then `chmod +x` and switches back to the `node` user. No package manager is invoked here. Because the binaries are statically linked, they run without any extra setup.

This means the build is completely **non-invasive** to the upstream image — you get vanilla n8n plus two extra files on `PATH`.

### Architectural note: amd64-only

The static tarball used is **amd64-only**. The build is pinned to `--platform=linux/amd64`. If you need an ARM image, you'll have to source a static `arm64` ffmpeg build and adjust the Dockerfile accordingly.

## Repository layout

| File                            | Purpose                                                                 |
| ------------------------------- | ----------------------------------------------------------------------- |
| `Dockerfile.template`           | Template Dockerfile. Copy to `Dockerfile` to use.                       |
| `update-image-amd64.sh.template`| Template build/push script. Copy to `update-image-amd64.sh` and edit.   |
| `.version`                      | Source of truth for the n8n version to build. Edit before each release. |

The actual `Dockerfile` and `update-image-amd64.sh` are git-ignored because they may contain your registry username. The repo ships templates; you create your own local copies.

## Usage

### 1. Initial setup (once)

```bash
git clone <this-repo> n8n-ffmpeg
cd n8n-ffmpeg

cp Dockerfile.template Dockerfile
cp update-image-amd64.sh.template update-image-amd64.sh
chmod +x update-image-amd64.sh
```

Open `update-image-amd64.sh` and set `IMAGE_NAME` to your registry path, e.g.:

```bash
IMAGE_NAME="yourdockerhubuser/n8n-ffmpeg"
```

### 2. Build a new version

Whenever you want to bump the n8n version:

1. Edit `.version` and write the desired n8n version (format `X.Y.Z`, e.g. `2.20.0`). You can find available versions on the [n8n Docker Hub page](https://hub.docker.com/r/n8nio/n8n/tags).
2. Run:

   ```bash
   ./update-image-amd64.sh
   ```

3. Confirm with `y`. The script will:
   - Build the image for `linux/amd64` using `n8nio/n8n:<.version>` as base
   - Tag both `:<version>` and `:latest`
   - Prompt for `docker login` if needed
   - Push both tags to your registry

### 3. Local-only build (no push)

```bash
docker build --platform=linux/amd64 \
  --build-arg N8N_VERSION=$(cat .version) \
  -t yourdockerhubuser/n8n-ffmpeg:test .
```

## Using the image

Drop-in replacement for `n8nio/n8n` in your `docker-compose.yml` or `docker run` command:

```yaml
services:
  n8n:
    image: yourdockerhubuser/n8n-ffmpeg:latest
    # ... rest of your n8n configuration
```

Inside any "Execute Command" node (or your own custom code), `ffmpeg` and `ffprobe` are now available on `PATH`.

## Verifying ffmpeg is present

```bash
docker run --rm yourdockerhubuser/n8n-ffmpeg:latest ffmpeg -version
docker run --rm yourdockerhubuser/n8n-ffmpeg:latest ffprobe -version
```

## License

The Dockerfile and scripts in this repo are MIT-licensed. The resulting image bundles:

- [n8n](https://github.com/n8n-io/n8n) — under its own [Sustainable Use License](https://docs.n8n.io/sustainable-use-license/).
- [ffmpeg](https://ffmpeg.org/) static build from [johnvansickle.com](https://johnvansickle.com/ffmpeg/) — see ffmpeg's own licensing (LGPL / GPL components depending on the build).
