SHELL := /bin/sh
.DEFAULT_GOAL := test
.DELETE_ON_ERROR:

GO ?= go
GOFMT ?= $(shell "$(GO)" env GOROOT)/bin/gofmt
GO_TEST_FLAGS ?= -count=1 -v -timeout=5m -json
# Replace this default with the project's primary service name during initialization.
SERVICE ?= service
VERSION ?= dev

.PHONY: build run dev fmt test coverage

# Shared prerequisite checks, expanded only when a public target runs.
define guard
	@test -f go.mod || { printf '%s\n' 'Run from the initialized Go module root.' >&2; exit 1; }
	@"$(GO)" env GOVERSION > /dev/null
endef

build:
	$(guard)
	@mkdir -p bin
	"$(GO)" build -trimpath -ldflags '-X main.version=$(VERSION)' -o 'bin/$(SERVICE)' './cmd/$(SERVICE)'

run: build
	'./bin/$(SERVICE)'

dev: build
	@set -e; \
	if [ -f .env ]; then \
	  set -a; \
	  . ./.env; \
	  set +a; \
	fi; \
	exec './bin/$(SERVICE)'

test:
	$(guard)
	"$(GO)" test $(GO_TEST_FLAGS) ./...

coverage:
	$(guard)
	"$(GO)" test $(GO_TEST_FLAGS) -coverprofile=coverage.out -covermode=atomic ./...
	"$(GO)" tool cover -func=coverage.out

fmt:
	$(guard)
	@test -x '$(GOFMT)' || { printf '%s\n' 'gofmt is required from the selected Go toolchain.' >&2; exit 1; }
	@find . -type d \( -name .git -o -name .tools -o -name vendor \) -prune -o -type f -name '*.go' -exec '$(GOFMT)' -w {} +
