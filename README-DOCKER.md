# Docker Deployment Guide

## Build and Run

### Using Docker directly:

```bash
# Build the image
docker build -t pictochat .

# Run the container
docker run -p 8090:8090 pictochat
```

### Using Docker Compose (recommended):

```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

## Configuration

The server will be accessible on:
- `http://localhost:8090` from your local machine
- `http://YOUR_LOCAL_IP:8090` from other devices on your network

## Environment Variables

Set environment variables in a `.env` file or pass them directly:

```bash
docker run -p 8090:8090 \
  -e PICTOJAVA_SECRET="your_secret" \
  -e PICTOJAVA_TRIPCODE_SECRET="your_tripcode_secret" \
  pictochat
```
