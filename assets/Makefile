ENV_FILE ?= .env

.PHONY: build test fmt tidy run air verify db-up db-down docker-up docker-down cover

build:
	go build ./...

test:
	go test ./...

cover:
	go test -cover ./...

fmt:
	gofmt -w ./cmd ./internal

tidy:
	go mod tidy

run:
	ENV_FILE=$(ENV_FILE) go run ./cmd/api

# Requiere `air` instalado: go install github.com/air-verse/air@latest
air:
	ENV_FILE=$(ENV_FILE) air

# Compuerta antes de commitear: formato, dependencias, compilación y tests.
verify: fmt tidy build test

# Solo la base de datos (para correr la API con `make run` desde el host).
db-up:
	docker compose up -d db

db-down:
	docker compose down

# API + base de datos.
docker-up:
	docker compose up -d --build

docker-down:
	docker compose down -v
