# --- build ---
FROM golang:1.25-alpine AS builder

WORKDIR /src

# Las dependencias primero: mientras go.mod/go.sum no cambien, Docker reutiliza
# esta capa y no vuelve a descargar módulos en cada build.
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# CGO apagado para obtener un binario estático que corra en una imagen sin libc.
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/api ./cmd/api

# --- runtime ---
FROM alpine:3.20

# ca-certificates para llamadas HTTPS salientes; tzdata si la app maneja
# zonas horarias de negocio.
RUN apk add --no-cache ca-certificates tzdata && adduser -D -u 10001 appuser

COPY --from=builder /out/api /app/api

USER appuser
WORKDIR /app
EXPOSE 3000

CMD ["/app/api"]
