# DevTools Frontend Nginx Image

This directory keeps a small Docker image definition for serving the DevTools
frontend artifact through nginx.

The image downloads the artifact zip at build time, extracts it into nginx's web
root, and serves `inspector.html` plus its sibling static resources over HTTP.

## Build

```bash
docker build -t devtools-frontend-nginx .
```

To override the artifact:

```bash
docker build \
  --build-arg DEVTOOLS_FRONTEND_DIST_URL=https://code.lexmount.net/api/packages/jinghua.guo/generic/devtools-frontend-artifacts/dev-392a237/dist.zip \
  -t devtools-frontend-nginx .
```

## Run

```bash
docker run --rm -p 8080:8080 devtools-frontend-nginx
```

Then open:

```text
http://127.0.0.1:8080/inspector.html
```
