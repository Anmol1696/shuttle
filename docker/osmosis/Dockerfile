ARG CHAIN_NAME
ARG CHAIN_BIN
ARG CHAIN_VERSION
ARG CODE_REF

FROM golang:1.18-alpine3.16 AS go-builder

ENV CHAIN_NAME=${CHAIN_NAME:-"osmosis"}
ENV CHAIN_BIN=${CHAIN_BIN:-"osmosisd"}
ENV CHAIN_VERSION=${CHAIN_VERSION:-"12.1.0"}
ENV CODE_REF=${CODE_REF:-"https://github.com/osmosis-labs/osmosis/archive/refs/tags"}

# Set up dependencies
ENV PACKAGES curl make git libusb-dev libc-dev bash gcc linux-headers eudev-dev python3

# Install ca-certificates
RUN set -eux; apk add --no-cache ca-certificates build-base;

# Install minimum necessary dependencies
RUN apk add --no-cache $PACKAGES

# See https://github.com/CosmWasm/wasmvm/releases
ADD https://github.com/CosmWasm/wasmvm/releases/download/v1.0.0/libwasmvm_muslc.aarch64.a /lib/libwasmvm_muslc.aarch64.a
ADD https://github.com/CosmWasm/wasmvm/releases/download/v1.0.0/libwasmvm_muslc.x86_64.a /lib/libwasmvm_muslc.x86_64.a
RUN sha256sum /lib/libwasmvm_muslc.aarch64.a | grep 7d2239e9f25e96d0d4daba982ce92367aacf0cbd95d2facb8442268f2b1cc1fc
RUN sha256sum /lib/libwasmvm_muslc.x86_64.a | grep f6282df732a13dec836cda1f399dd874b1e3163504dbd9607c6af915b2740479

# Copy the library you want to the final location that will be found by the linker flag `-lwasmvm_muslc`
RUN cp /lib/libwasmvm_muslc.$(uname -m).a /lib/libwasmvm_muslc.a

# Set working directory for the build
WORKDIR /usr/local/app

# Add source files
ADD $CODE_REF/v$CHAIN_VERSION.zip /tmp
RUN cd /tmp && unzip v${CHAIN_VERSION}.zip && mv /tmp/${CHAIN_NAME}-${CHAIN_VERSION}/* /usr/local/app

# Force it to use static lib (from above) not standard libgo_cosmwasm.so file
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/root/go/pkg/mod \
    go build \
      -mod=readonly \
      -tags "netgo,ledger,muslc" \
      -ldflags "-X github.com/cosmos/cosmos-sdk/version.Name="osmosis" \
              -X github.com/cosmos/cosmos-sdk/version.AppName="osmosisd" \
              -X github.com/cosmos/cosmos-sdk/version.Version=latest \
              -X github.com/cosmos/cosmos-sdk/version.Commit=tester \
              -X github.com/cosmos/cosmos-sdk/version.BuildTags='netgo,ledger,muslc' \
              -w -s -linkmode=external -extldflags '-Wl,-z,muldefs -static'" \
      -trimpath \
      -o /usr/local/app/build/osmosisd \
      /usr/local/app/cmd/osmosisd/main.go

# Install cosmovisor
RUN go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@v1.0.0

FROM alpine:3.16

ENV CHAIN_NAME=${CHAIN_NAME:-"osmosis"}
ENV CHAIN_BIN=${CHAIN_BIN:-"osmosisd"}
ENV CHAIN_VERSION=${CHAIN_VERSION:-"12.1.0"}

COPY --from=go-builder /usr/local/app/build/$CHAIN_BIN /usr/bin/$CHAIN_BIN
COPY --from=go-builder /go/bin/cosmovisor /usr/bin/cosmovisor

# Set up dependencies
ENV PACKAGES curl make bash jq sed
ENV CHAIN_BIN /usr/bin/$CHAIN_BIN

# Install minimum necessary dependencies
RUN apk add --no-cache $PACKAGES

WORKDIR /opt

# rest server, p2p, rpc, grpc
EXPOSE 1317 26656 26657 9090

# Run persistenceCore by default, omit entrypoint to ease using container with cli
CMD ["/usr/bin/osmosisd", "version", "--long"]
