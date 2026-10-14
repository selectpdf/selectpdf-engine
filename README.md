# SelectPdf engine

The native conversion engines of **SelectPdf.Universal** as an HTTP service: the Chromium engine that
converts HTML to PDF and to images, and the PDF tool that extracts text and renders PDF pages to images.

An application normally runs these engines itself, from the `SelectPdf.Universal.Native.<rid>` NuGet
package. With `RenderingEngine.ChromiumRemote` it sends that work to this container instead, which can run on
another machine, as another container, or beside the application in the same pod.

Everything else stays in the application and works as it does with a local engine: building the document,
headers and footers, bookmarks, PDF/A and PDF/UA, security and digital signatures, forms, Word to PDF,
redaction - and the license check. **The engine itself needs no license key.**

## Why use it

- **Alpine Linux.** The native engines cannot run on musl-based systems. With the remote engine, an application
  on `mcr.microsoft.com/dotnet/aspnet:10.0-alpine` converts like any other: this is the supported way to use
  the library there.
- **A small application.** The application references only the managed package (`SelectPdf.Universal`, or
  `SelectPdf.HtmlToPdf.Universal` for the free Community edition) - no native package, no fonts package, no
  Chromium in its image.
- **Isolation and scale.** Rendering runs apart from the application, scales on its own, and one engine can serve
  several applications.
- **Nothing to install.** The image carries Chromium with every library it needs, fonts for every language (the
  Noto family, color emoji included) and the PDF tool. No `apt-get`, no `--shm-size`, no extra capabilities, no
  display server.

## Tags

| tag | what it is |
|---|---|
| `26.5.0` | the engine for SelectPdf.Universal 26.5.0 |
| `26.5` | the latest build for 26.5 |

Each tag is a multi-architecture image for **linux/amd64** and **linux/arm64**; Docker picks the right one.
**Use the tag of the library version your application references.** There is no `latest` tag. When an engine
does not speak the protocol of the library calling it, the conversion fails with a message saying so.

The image is published as `ghcr.io/selectpdf/engine` and mirrored on Docker Hub as `selectpdf/engine`. Chromium
154.0.28.

## Quick start

**1. Start the engine** with an access token (at least 32 characters):

```sh
export SELECTPDF_ENGINE_TOKEN="$(openssl rand -base64 32)"

docker run -d --name selectpdf-engine -p 8080:8080 \
  --read-only --tmpfs /tmp \
  -e SELECTPDF_ENGINE_TOKEN \
  ghcr.io/selectpdf/engine:26.5
```

The engine reports healthy once it has done a first conversion, a second or two after it starts:

```sh
curl http://localhost:8080/health
```

**2. Install the managed package** in the application - no native package is needed:

```sh
dotnet add package SelectPdf.Universal
```

**3. Point the application at the engine**, once at startup. The conversion code does not change:

```csharp
using SelectPdf.Universal;

GlobalProperties.RenderingEngine = RenderingEngine.ChromiumRemote;
GlobalProperties.EngineEndpoint = "http://localhost:8080";
GlobalProperties.EngineAccessToken =
    Environment.GetEnvironmentVariable("SELECTPDF_ENGINE_TOKEN");

HtmlToPdf converter = new HtmlToPdf();
PdfDocument doc = converter.ConvertUrl("https://selectpdf.com");
doc.Save("test.pdf");
doc.Close();
```

```vb
Imports SelectPdf.Universal

GlobalProperties.RenderingEngine = RenderingEngine.ChromiumRemote
GlobalProperties.EngineEndpoint = "http://localhost:8080"
GlobalProperties.EngineAccessToken =
    Environment.GetEnvironmentVariable("SELECTPDF_ENGINE_TOKEN")

Dim converter As New HtmlToPdf()
Dim doc As PdfDocument = converter.ConvertUrl("https://selectpdf.com")
doc.Save("test.pdf")
doc.Close()
```

Or set the same three values as environment variables of the application, with no code at all - the usual way
in a container deployment:

```
SELECTPDF_RENDERING_ENGINE=ChromiumRemote
SELECTPDF_ENGINE_ENDPOINT=http://selectpdf-engine:8080
SELECTPDF_ENGINE_TOKEN=<one of the engine's tokens>
```

## Application settings

| property (`GlobalProperties`) | environment variable | meaning |
|---|---|---|
| `RenderingEngine` | `SELECTPDF_RENDERING_ENGINE` | `Chromium` (the default: the engines run in the application) or `ChromiumRemote` (this container). Every converter follows it unless its own `RenderingEngine` property is set; the PDF tools always follow it. |
| `EngineEndpoint` | `SELECTPDF_ENGINE_ENDPOINT` | The engine's url. Default `http://selectpdf-engine:8080` - an engine service named `selectpdf-engine` on the same Docker network or cluster. |
| `EngineAccessToken` | `SELECTPDF_ENGINE_TOKEN` | One of the tokens the engine was started with. |
| `EngineTimeout` | `SELECTPDF_ENGINE_TIMEOUT` (seconds) | How long one request may take. By default it follows the conversion's own navigation and print timeouts, plus time to wait for a free slot on a busy engine. |

## Engine settings

All settings are environment variables of the container. A setting that cannot be honoured stops the engine at
start, with a message naming it.

| variable | default | meaning |
|---|---|---|
| `SELECTPDF_ENGINE_TOKEN` | none | The access tokens the engine accepts, separated by commas. Several at once let a token be rotated without downtime: add the new one, move the clients, remove the old one. Each must be at least 32 characters. **Without a token, anyone who can reach the port can use the engine** (it logs a warning). |
| `SELECTPDF_ENGINE_TOKEN_FILE` | none | A file holding the tokens, one per line - a Docker or Kubernetes secret. Instead of `SELECTPDF_ENGINE_TOKEN`, not with it. |
| `SELECTPDF_MAX_CONCURRENCY` | number of CPUs | HTML conversions run at once. |
| `SELECTPDF_MAX_TOOL_CONCURRENCY` | number of CPUs | PDF to text / PDF to image operations run at once. |
| `SELECTPDF_QUEUE_TIMEOUT` | `30` | Seconds a request waits for a free slot. After that the engine answers 503 with `Retry-After: 1` and closes the connection, so a retry can reach another replica; the library retries such an answer up to three times. |
| `SELECTPDF_MAX_BODY_MB` | `100` | The largest request accepted, in MB. |
| `SELECTPDF_ENGINE_HTTP_CACHE` | `all` | The most of Chromium's HTTP cache a conversion may keep: `clear`, `thirdParty` or `all` (each conversion keeps what it asks for through `HttpCache`). **Set `clear` on an engine shared by different callers.** |
| `SELECTPDF_ENGINE_PORT` | `8080` | The port to listen on - change it when the engine shares a pod with an application on 8080. |
| `SELECTPDF_TEMP_DIR` | `/tmp/selectpdf-engine` | Where conversions work. Each request has a folder of its own, deleted when it ends. |

A single run is never allowed more than 10 minutes, whatever the client asks for.

## Endpoints

| endpoint | meaning |
|---|---|
| `GET /health` | 200 once the engine has done its first conversion, 503 before that or when it cannot convert. No token needed. The image's `HEALTHCHECK` uses it. `/healthz` answers the same. |
| `GET /v1/info` | The engine's version, Chromium version, platform and limits. Needs the token. |

The conversion endpoints are spoken by the library; there is nothing to call by hand.

## Docker Compose

Name the engine service `selectpdf-engine` and the application needs no endpoint setting at all:

```yaml
services:
  selectpdf-engine:
    image: ghcr.io/selectpdf/engine:26.5
    read_only: true
    tmpfs:
      - /tmp
    environment:
      SELECTPDF_ENGINE_TOKEN: ${SELECTPDF_ENGINE_TOKEN}
      # an engine shared by different callers:
      # SELECTPDF_ENGINE_HTTP_CACHE: clear
    mem_limit: 4g

  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SELECTPDF_RENDERING_ENGINE: ChromiumRemote
      SELECTPDF_ENGINE_TOKEN: ${SELECTPDF_ENGINE_TOKEN}
    depends_on:
      selectpdf-engine:
        condition: service_healthy
```

The engine is not published on the host: only the application can reach it. The SelectPdf samples include this
setup complete, with an ASP.NET Core application on Alpine (`docker/remote-engine`).

## Kubernetes

**With Helm** - the chart runs the engine non-root with a read-only root file system, `/health` probes, the token
in a Secret and a PodDisruptionBudget:

```sh
helm install selectpdf-engine \
  oci://ghcr.io/selectpdf/charts/selectpdf-engine --version 26.5.0 \
  --namespace pdf --create-namespace \
  --set auth.token="$(openssl rand -base64 32)"
```

Several tokens are separated by commas, which `--set` splits on: escape them (`--set auth.token='a\,b'`) or use a
values file.

**Without Helm** - the engine as a Deployment behind a Service named `selectpdf-engine`, the token from a Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: selectpdf-engine-token
stringData:
  token: "<at least 32 characters, e.g. openssl rand -base64 32>"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: selectpdf-engine
spec:
  replicas: 2
  selector:
    matchLabels: { app: selectpdf-engine }
  template:
    metadata:
      labels: { app: selectpdf-engine }
    spec:
      containers:
        - name: engine
          image: ghcr.io/selectpdf/engine:26.5
          ports:
            - containerPort: 8080
          env:
            - name: SELECTPDF_ENGINE_TOKEN_FILE
              value: /run/secrets/selectpdf/token
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
            periodSeconds: 10
          startupProbe:
            httpGet: { path: /health, port: 8080 }
            periodSeconds: 5
            failureThreshold: 24
          resources:
            requests: { cpu: "2", memory: 2Gi }
            limits: { memory: 4Gi }
          securityContext:
            runAsNonRoot: true
            runAsUser: 1654
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
            - { name: token, mountPath: /run/secrets/selectpdf, readOnly: true }
      volumes:
        - name: tmp
          emptyDir: {}
        - name: token
          secret:
            secretName: selectpdf-engine-token
---
apiVersion: v1
kind: Service
metadata:
  name: selectpdf-engine
spec:
  selector: { app: selectpdf-engine }
  ports:
    - port: 8080
      targetPort: 8080
```

The application then sets `SELECTPDF_RENDERING_ENGINE=ChromiumRemote` and the same token, and needs no
endpoint. To run the engine as a second container in the application's pod instead, give it another
`SELECTPDF_ENGINE_PORT` if the application uses 8080, and set `SELECTPDF_ENGINE_ENDPOINT=http://localhost:<port>`
in the application.

## In the cloud

Each of these was tested end to end with an application on `aspnet:10.0-alpine`; the documentation has the full
commands.

| platform | how the application reaches the engine |
|---|---|
| **Azure Container Apps** | the engine as an app with internal ingress named `selectpdf-engine`; the application sets `SELECTPDF_ENGINE_ENDPOINT=http://selectpdf-engine`. Keep `--min-replicas 1`. |
| **Azure Container Instances** | the engine as a second container in the application's group, `SELECTPDF_ENGINE_PORT=8081`; the application uses `http://localhost:8081`. Only the application's port is published. |
| **Amazon ECS on Fargate** | the engine as its own service with Service Connect, client alias `selectpdf-engine` on port 8080 (the library's default, no endpoint setting); or as a second container in the application's task on `localhost:8081`, started after the engine is healthy. With `readonlyRootFilesystem`, mount a task volume at `/tmp` - the image declares it, so the volume is writable by the engine. |
| **Google Cloud Run** | the engine as its own service, deployed with `--no-invoker-iam-check`; the application uses its HTTPS URL. Without that flag Cloud Run checks the `Authorization` header of a call from another Cloud Run service as a Google identity token and refuses the engine's token. Check its health from outside at `/health`: Cloud Run reserves paths ending in `z`, so `/healthz` is not served from outside there. |
| **Kubernetes** (AKS, EKS, GKE, ...) | the Helm chart above; the application uses `http://selectpdf-engine` in the same namespace. |

## Security

- **Use a token, and keep the engine off the public internet** - on the Docker network, the cluster network or a
  private subnet of the application.
- **The token is never sent in clear text to another network.** With a token set, the library requires an
  `https` endpoint unless the endpoint is local: a service name without a dot, `localhost`, a loopback or a
  private address. The engine serves plain HTTP; for an engine on another machine, put TLS in front of it (an
  ingress, a load balancer or a reverse proxy).
- **A page cannot read the engine's files:** local file access is always off in the engine, whatever a conversion
  asks for.
- **The engine fetches what the pages reference, from its own network.** When it converts HTML you do not control,
  limit where the engine's network can reach (internal services, cloud metadata addresses).
- **A shared engine:** set `SELECTPDF_ENGINE_HTTP_CACHE=clear`, so no conversion is served a response another
  caller's conversion fetched. Cookies and storage are cleared after every conversion in any case.
- **The container runs as a non-root user** (`app`, uid 1654), works with a read-only root file system and a
  writable `/tmp`, and needs no added capabilities.

## What differs from a local engine

- **Local files:** the engine cannot see the application's files, so a `file://` url or base url is refused
  before anything is sent. Convert the HTML as a string, or serve the files over HTTP.
- **`localhost`** in a url means the engine's machine, not the application's - except when the engine runs in the
  application's pod.
- **Fonts:** the engine renders with its own fonts (every language). Fonts in the application's own `fonts`
  folder are sent to the engine with each conversion.
- **Per-process settings** of a local engine - WebGL, a proxy, insecure content - are honoured per conversion: the
  engine runs such a conversion in a process of its own. A proxy must be reachable from the engine.
- **Errors** are the same exception types as with a local engine. When the engine could not be reached or refused
  the request, the message says so and names the request id the engine logs its runs under.
- **Cancellation** of an awaited conversion (`ConvertUrlAsync` and the other asynchronous methods) cancels it in
  the engine.

## Sizing

One engine converts as many pages at once as it has CPUs (by default) and queues the rest. Measured with a light
document of one or two pages, one engine limited to N CPUs:

| CPUs | one conversion | conversions per second, fully loaded |
|---|---|---|
| 1 | about 100 ms | about 7 |
| 2 | about 60 ms | about 10 |
| 4 | about 50 ms | about 18 |
| 8 | about 50 ms | about 24 |

Real pages take longer, mostly in fetching them. Throughput grows little beyond 4 to 8 CPUs per engine; for more,
run more replicas behind one service. Memory stayed under 500 MB with eight light conversions at once; heavy pages
(news sites, long reports) need more - give each engine 2 GB or more.

## More

- Documentation, Remote Engine topic:
  https://selectpdf.com/pdf-library/html/Remote-Engine.htm
- SelectPdf.Universal on NuGet: `SelectPdf.Universal` (Full) and `SelectPdf.HtmlToPdf.Universal` (Community, free)
- Licensing and support: https://selectpdf.com
