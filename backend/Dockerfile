# -------------------------
# Build stage
# -------------------------
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Copy go.mod and go.sum first (layer caching)
COPY go.mod go.sum ./
RUN go mod download

# Copy the rest of the backend code
COPY . .

# Build the Go binary
RUN go build -o server ./cmd/server

# -------------------------
# Runtime stage
# -------------------------
FROM alpine:3.19

WORKDIR /app

# Copy the compiled binary
COPY --from=builder /app/server .

# Copy migrations (if the binary needs them)
COPY migrations ./migrations

EXPOSE 8080

CMD ["./server"]
