VERSION 0.6
FROM alpine

ARG BASE_IMAGE=quay.io/kairos/core-opensuse:latest
ARG IMAGE_REPOSITORY=quay.io/kairos

ARG LUET_VERSION=0.32.4
ARG GOLINT_VERSION=v1.46.2
ARG GOLANG_VERSION=1.18

ARG K3S_VERSION=latest
ARG BASE_IMAGE_NAME=$(echo $BASE_IMAGE | grep -o [^/]*: | rev | cut -c2- | rev)
ARG BASE_IMAGE_TAG=$(echo $BASE_IMAGE | grep -o :.* | cut -c2-)
ARG K3S_VERSION_TAG=$(echo $K3S_VERSION | sed s/+/-/)

build-cosign:
    FROM gcr.io/projectsigstore/cosign:v1.9.0
    SAVE ARTIFACT /ko-app/cosign cosign

go-deps:
    FROM golang:$GOLANG_VERSION
    WORKDIR /build
    COPY go.mod go.sum ./
    RUN go mod download
    RUN apt-get update && apt-get install -y upx
    SAVE ARTIFACT go.mod AS LOCAL go.mod
    SAVE ARTIFACT go.sum AS LOCAL go.sum

BUILD_GOLANG:
    COMMAND
    WORKDIR /build
    COPY . ./
    ARG BIN
    ARG SRC

    ENV CGO_ENABLED=0

    RUN go build -ldflags "-s -w" -o ${BIN} ./${SRC} && upx ${BIN}
    SAVE ARTIFACT ${BIN} ${BIN} AS LOCAL build/${BIN}

VERSION:
    COMMAND
    FROM alpine
    RUN apk add git

    COPY . ./

    RUN echo $(git describe --exact-match --tags || echo "v0.0.0-$(git log --oneline -n 1 | cut -d" " -f1)") > VERSION

    SAVE ARTIFACT VERSION VERSION

lint:
    FROM golang:$GOLANG_VERSION
    RUN wget -O- -nv https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s $GOLINT_VERSION
    WORKDIR /build
    COPY . .
    RUN golangci-lint run

build-provider:
    FROM +go-deps
    DO +BUILD_GOLANG --BIN=agent-provider-k3s --SRC=main.go

docker:
    DO +VERSION
    ARG VERSION=$(cat VERSION)

    FROM $BASE_IMAGE

    IF [ "$K3S_VERSION" = "latest" ]
    ELSE
        ENV INSTALL_K3S_VERSION=${K3S_VERSION}
    END

    ENV INSTALL_K3S_BIN_DIR="/usr/bin"
    RUN curl -sfL https://get.k3s.io > installer.sh \
        && INSTALL_K3S_SKIP_START="true" INSTALL_K3S_SKIP_ENABLE="true" bash installer.sh \
        && INSTALL_K3S_SKIP_START="true" INSTALL_K3S_SKIP_ENABLE="true" bash installer.sh agent \
        && rm -rf installer.sh

    COPY +build-provider/agent-provider-k3s /system/providers/agent-provider-k3s

    ENV OS_ID=${BASE_IMAGE_NAME}-k3s
    ENV OS_NAME=$OS_ID:${BASE_IMAGE_TAG}
    ENV OS_REPO=${IMAGE_REPOSITORY}
    ENV OS_VERSION=${K3S_VERSION_TAG}_${VERSION}
    ENV OS_LABEL=${BASE_IMAGE_TAG}_${K3S_VERSION_TAG}_${VERSION}
    RUN envsubst >/etc/os-release </usr/lib/os-release.tmpl

    SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-k3s:${K3S_VERSION_TAG}
    SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-k3s:${K3S_VERSION_TAG}_${VERSION}

cosign:
    ARG --required ACTIONS_ID_TOKEN_REQUEST_TOKEN
    ARG --required ACTIONS_ID_TOKEN_REQUEST_URL

    ARG --required REGISTRY
    ARG --required REGISTRY_USER
    ARG --required REGISTRY_PASSWORD

    DO +VERSION
    ARG VERSION=$(cat VERSION)

    FROM docker

    ENV ACTIONS_ID_TOKEN_REQUEST_TOKEN=${ACTIONS_ID_TOKEN_REQUEST_TOKEN}
    ENV ACTIONS_ID_TOKEN_REQUEST_URL=${ACTIONS_ID_TOKEN_REQUEST_URL}

    ENV REGISTRY=${REGISTRY}
    ENV REGISTRY_USER=${REGISTRY_USER}
    ENV REGISTRY_PASSWORD=${REGISTRY_PASSWORD}

    ENV COSIGN_EXPERIMENTAL=1
    COPY +build-cosign/cosign /usr/local/bin/

    RUN echo $REGISTRY_PASSWORD | docker login -u $REGISTRY_USER --password-stdin $REGISTRY

    RUN cosign sign $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-k3s:${K3S_VERSION_TAG}
    RUN cosign sign $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-k3s:${K3S_VERSION_TAG}_${VERSION}

docker-all-platforms:
     BUILD --platform=linux/amd64 +docker
     BUILD --platform=linux/arm64 +docker

cosign-all-platforms:
     BUILD --platform=linux/amd64 +cosign
     BUILD --platform=linux/arm64 +cosign
