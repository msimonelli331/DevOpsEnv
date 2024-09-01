FROM alpine:3.20.2

COPY ca.crt /usr/local/share/ca-certificates/
RUN apk --no-cache add ca-certificates \
    && update-ca-certificates

ARG BUILDKIT_VERSION=v0.15.2
RUN wget https://github.com/moby/buildkit/releases/download/\${BUILDKIT_VERSION}/buildkit-\${BUILDKIT_VERSION}.linux-amd64.tar.gz \
    && tar xvf buildkit-\${BUILDKIT_VERSION}.linux-amd64.tar.gz -C /usr \
    && rm -rf buildkit-\${BUILDKIT_VERSION}.linux-amd64.tar.gz
