# Dockerfile Instructions: ARG, ENV, ADD, CMD and ENTRYPOINT

## `ARG`: a setting used only while **building** the image

```dockerfile
ARG APP_VERSION=1.0
RUN echo "Building version $APP_VERSION"
```
Change it at build time:
```bash
docker build --build-arg APP_VERSION=2.5 -t myapp .
```
- Exists **only during `docker build`**
- **Gone** when the container runs

**Use it for:** values that change how the image is built, like a version number, a base image tag, or which package version to download.

**Don't use it for:** passwords. They can show up in the image history (`docker history`).

## `ENV`: a setting the app sees while **running**

```dockerfile
ENV ASPNETCORE_URLS=http://+:8080
ENV PYTHONUNBUFFERED=1
```
- Available during the build (after the line) **and** inside the running container
- Can be changed at run time: `docker run -e ASPNETCORE_URLS=http://+:9090 myapp`

**Use it for:** settings the app reads when it runs, like the port to listen on, the environment (`Production`), or logging behavior.

**Don't use it for:** passwords and secrets. Anyone with the image can read them. Pass those at run time with `-e`, or with a secrets tool.

### `ARG` vs `ENV` in one line
> **`ARG` = build time only. `ENV` = build time and run time.**

## `ADD`: like `COPY`, with extra powers

```dockerfile
COPY app.py /app/          # plain copy
ADD  app.tar.gz /app/      # copies AND automatically extracts the archive
ADD  https://example.com/file.txt /app/   # can also download from a URL
```
The two extra things `ADD` does:
1. **Auto-extracts** local `.tar`, `.tar.gz`, and similar archives
2. **Downloads from URLs**

**Use `COPY` almost always.** It's predictable: it just copies.

**Use `ADD` only when** you specifically want a local tar archive extracted into the image. Surprise extraction or downloads make `ADD` harder to reason about, which is why best practice says "prefer `COPY`."

## `CMD`: the **default** command, easy to replace

```dockerfile
CMD ["python", "app.py"]
```
- Runs when the container starts
- **Replaced** if you give another command: `docker run myapp bash` runs `bash` instead

**Use it for:** most apps, especially when you want to easily run something else for debugging.

## `ENTRYPOINT`: the **fixed** program the container always runs

```dockerfile
ENTRYPOINT ["dotnet", "ProjectOps.Api.dll"]
```
- Runs when the container starts
- Extra words after `docker run myapp ...` are **added as arguments**, not a replacement
- Replace it only with `--entrypoint`

**Use it for:** containers that should always run one specific program, like your Java and .NET backends.

### `CMD` vs `ENTRYPOINT` in one line
> **`CMD` = default command, easy to swap. `ENTRYPOINT` = always this program.**

## All five at a glance

| Instruction | Simple meaning | When it's used | Typical use |
|---|---|---|---|
| `ARG` | Build setting | Build only | Versions, build options |
| `ENV` | App setting | Build + run | Port, environment, app config |
| `COPY` | Copy files | Build | Code, config files (default choice) |
| `ADD` | Copy + extract/download | Build | Extracting a local `.tar.gz` |
| `CMD` | Default start command | Run | Most apps; easy to override |
| `ENTRYPOINT` | Fixed start program | Run | "This container always runs X" |

## One small Dockerfile using them all

```dockerfile
FROM python:3.13-slim

ARG APP_VERSION=1.0                 # build-time setting
ENV PORT=5000                       # runtime setting the app reads
ENV APP_VERSION=$APP_VERSION        # copy the ARG into ENV to keep it at run time

WORKDIR /app
COPY requirements.txt .             # COPY for normal files
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ADD static-assets.tar.gz /app/static/   # ADD only to auto-extract an archive

ENTRYPOINT ["python"]               # always run python...
CMD ["app.py"]                      # ...with app.py by default
```
- `docker run myapp` runs `python app.py`
- `docker run myapp other.py` runs `python other.py` (`CMD` replaced, `ENTRYPOINT` kept)

The `ENV APP_VERSION=$APP_VERSION` line is a common trick: it copies a build-time `ARG` into a runtime `ENV`, so the running app can still see the version.