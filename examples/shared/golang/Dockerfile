FROM debian:bookworm-slim@sha256:3d5df92588469a4c503adbead0e4129ef3f88e223954011c2169073897547cac as os-base
RUN rm -f /etc/apt/apt.conf.d/docker-clean \
    && echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' | tee /etc/apt/apt.conf.d/keep-cache


FROM golang:1.22.2-bookworm@sha256:b03f3ba515751657c75475b20941fef47341fccb3341c3c0b64283ff15d3fb46 as golang-base


FROM golang-base as golang-control-plane-builder
ARG GO_RESOURCE=resource.go
RUN git clone https://github.com/envoyproxy/go-control-plane && cd go-control-plane && git checkout b4adc3bb5fe5288bff01cd452dad418ef98c676e
ADD "$GO_RESOURCE" /go/go-control-plane/internal/example/resource.go
RUN cd go-control-plane && make bin/example
WORKDIR /go/go-control-plane


FROM os-base as golang-control-plane
ENV DEBIAN_FRONTEND=noninteractive
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get -qq update \
    && apt-get -qq install --no-install-recommends -y netcat-traditional
COPY --from=golang-control-plane-builder /go/go-control-plane/bin/example /usr/local/bin/example


FROM golang-base as golang-base-builder
ARG GO_APP=.
ARG GO_APP_NAME=grpc-service
RUN echo "COPY $GO_APP -> ${GO_APP_NAME} ..."
COPY "$GO_APP" /app


FROM golang-base-builder as golang-grpc-auth-builder
RUN make -C "/app/${GO_APP_NAME}"


FROM os-base as golang-grpc-auth
COPY --from=golang-grpc-auth-builder /app/grpc-service/server /app/server
CMD ["/app/server", "-users", "/etc/users.json"]


FROM golang-base as golang-grpc-server-builder
WORKDIR /build
# Resolve and build Go dependencies as Docker cache
COPY go.mod /build/go.mod
COPY go.sum /build/go.sum
ENV GO111MODULE=on
RUN go mod download
COPY service.go /build/main.go
COPY kv/ /build/kv
# Build for linux
ENV GOOS=linux
ENV GOARCH=amd64
ENV CGO_ENABLED=0
RUN go build -o server


FROM os-base as golang-grpc-server
WORKDIR /root/
# Copy the linux amd64 binary
COPY --from=golang-grpc-server-builder /build/server /bin/
ENTRYPOINT /bin/server


FROM golang-base as golang-lrs-builder
COPY ./server /go/src/github.com/envoyproxy/envoy/example/load-reporting-service/server
COPY *.go /go/src/github.com/envoyproxy/envoy/example/load-reporting-service/
COPY go.sum /go/src/github.com/envoyproxy/envoy/example/load-reporting-service
COPY go.mod /go/src/github.com/envoyproxy/envoy/example/load-reporting-service
WORKDIR /go/src/github.com/envoyproxy/envoy/example/load-reporting-service
RUN go mod download \
    && go install /go/src/github.com/envoyproxy/envoy/example/load-reporting-service


FROM os-base as golang-lrs
COPY --from=golang-lrs-builder /go/bin/load-reporting-service /usr/local/bin/load-reporting-service
CMD ["load-reporting-service"]
