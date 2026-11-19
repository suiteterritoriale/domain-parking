# The page builder: one static Go binary on distroless. The server is a
# separate image, built from caddy/.
#
# The `s3dev` target is the S3 client for local development and the
# end-to-end test (see cmd/s3dev). It is not in the builder image.

# Digests resolved on 2026-11-18. GO_IMAGE is also pinned in the Makefile.
ARG GO_IMAGE=golang:1.27.4@sha256:205c7636d74e0ffbe3d007162df7d74f859f5a8c8322dfb9f907977686c126cf
ARG DISTROLESS_STATIC_IMAGE=gcr.io/distroless/static-debian13:nonroot@sha256:aafe993665bf6200db38a02c11a9351e782a84b6a45e7f33b6695638c88e3931

FROM ${GO_IMAGE} AS build

WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

COPY . ./
# CGO stays off: the binaries have to run on distroless static.
ENV CGO_ENABLED=0
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    go build -trimpath -ldflags='-s -w' -o /out/builder . \
    && go build -trimpath -ldflags='-s -w' -o /out/s3dev ./cmd/s3dev

# distroless has no shell, so the output directory is created and owned here.
RUN mkdir -p /state/sites && chown -R 65532:65532 /state

FROM ${DISTROLESS_STATIC_IMAGE} AS s3dev
COPY --from=build /out/s3dev /usr/local/bin/s3dev
ENTRYPOINT ["/usr/local/bin/s3dev"]

FROM ${DISTROLESS_STATIC_IMAGE} AS builder
COPY --from=build /out/builder /usr/local/bin/builder
COPY --from=build --chown=65532:65532 /state/sites /srv/sites
ENV OUTPUT_DIR=/srv/sites
ENTRYPOINT ["/usr/local/bin/builder"]
