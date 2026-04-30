FROM alpine:3.20 AS artifact

ARG DEVTOOLS_FRONTEND_DIST_URL="https://code.lexmount.net/api/packages/jinghua.guo/generic/devtools-frontend-artifacts/dev-392a237/dist.zip"

RUN apk add --no-cache ca-certificates curl unzip

WORKDIR /artifact
RUN curl -fsSL "${DEVTOOLS_FRONTEND_DIST_URL}" -o dist.zip \
    && unzip -q dist.zip -d dist \
    && test -f dist/inspector.html

FROM nginx:1.27-alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=artifact /artifact/dist/ /usr/share/nginx/html/

EXPOSE 8080
