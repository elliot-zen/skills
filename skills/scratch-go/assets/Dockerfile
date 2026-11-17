FROM golang:1.27.3-alpine3.24 AS build

RUN apk add --no-cache ca-certificates tzdata
ENV CGO_ENABLED=0 GOTOOLCHAIN=local
WORKDIR /src

COPY go.mod go.sum* ./
RUN go mod download
COPY . .

ARG SERVICE=service
ARG VERSION=dev
RUN go build -trimpath -ldflags "-X main.version=${VERSION}" -o /out/service "./cmd/${SERVICE}"

FROM scratch
WORKDIR /app

COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=build /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=build /out/service ./service
COPY configs/ ./configs/

USER 1000:1000
ENTRYPOINT ["/app/service"]
