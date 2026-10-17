FROM golang:1.27-alpine AS builder

WORKDIR /src

COPY go.mod go.sum ./

# Download Go module dependencies
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /out/simupi .

FROM alpine:3.20

WORKDIR /app

RUN apk add --no-cache ca-certificates && \
    adduser -D -H -u 10001 simupi && \
    mkdir -p /app/data/scenarios && \
    chown -R simupi:simupi /app

COPY --from=builder /out/simupi /app/simupi

ENV PORT=5000

EXPOSE 5000 1883

USER simupi

CMD ["/app/simupi"]