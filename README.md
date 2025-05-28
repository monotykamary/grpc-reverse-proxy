# gRPC Reverse Proxy

This project provides a simple gRPC reverse proxy using Caddy. It is designed to be run as a Docker container and forwards gRPC traffic to a backend service.

## How it Works

The core of this project is the `Caddyfile` which configures Caddy to act as a reverse proxy. It listens on a specified port and forwards all incoming gRPC requests (using `h2c` - HTTP/2 Cleartext) to a designated backend gRPC service.

The `Dockerfile` creates a lightweight Docker image based on `caddy:alpine` and includes this Caddy configuration.

## Configuration

The reverse proxy is configured using the following environment variables when running the Docker container:

*   `PORT`: The port on which the Caddy server will listen for incoming connections *inside* the container.
*   `SERVICE_URL`: The URL of the backend gRPC service to which traffic will be forwarded (e.g., `my-grpc-service:50051`). This service must support `h2c` (HTTP/2 Cleartext).

## Running with Docker

1.  **Build the Docker Image:**
    Navigate to the directory containing the `Dockerfile` and run:
    ```bash
    docker build -t grpc-reverse-proxy .
    ```

2.  **Run the Docker Container:**
    To run the proxy, you need to set the `PORT` environment variable for the port inside the container that Caddy listens on, and the `SERVICE_URL` environment variable for your backend gRPC service. You'll also map a host port to the container's port.

    ```bash
    docker run -d \
      -p <HOST_PORT>:<CONTAINER_LISTENING_PORT> \
      -e PORT=<CONTAINER_LISTENING_PORT> \
      -e SERVICE_URL=<YOUR_BACKEND_GRPC_SERVICE_URL> \
      --name my-grpc-proxy \
      grpc-reverse-proxy
    ```

    *   `<HOST_PORT>`: The port on your host machine that will forward to the proxy (e.g., `8080`).
    *   `<CONTAINER_LISTENING_PORT>`: The port Caddy will listen on *inside* the container (e.g., `80`). This value is set via the `PORT` environment variable.
    *   `<YOUR_BACKEND_GRPC_SERVICE_URL>`: The address of your gRPC service, accessible from the proxy container. For example:
        *   `host.docker.internal:50051`: If your service runs directly on your Docker host machine (common for Docker Desktop on Mac/Windows).
        *   `your-service-container-name:50051`: If your service runs in another Docker container on the same user-defined Docker network.
        *   `172.17.0.1:50051`: If your service runs on the host and you are on Linux (this is often the IP of the host on the default `docker0` bridge).

## Example Usage

Let's say you have a gRPC service (e.g., a Greeter service) running on your host machine at `localhost:50051`. You want to expose this service through the reverse proxy on port `8080` of your host machine. Caddy inside the container will listen on port `80`.

1.  **Ensure your gRPC service is running and accessible at `localhost:50051`.**

2.  **Build the Docker image (if you haven't already):**
    ```bash
    docker build -t grpc-reverse-proxy .
    ```

3.  **Run the reverse proxy container:**
    ```bash
    docker run -d \
      -p 8080:80 \
      -e PORT=80 \
      -e SERVICE_URL="host.docker.internal:50051" \
      --name example-grpc-proxy \
      grpc-reverse-proxy
    ```
    *   This maps port `8080` on your host to port `80` inside the container.
    *   Caddy inside the container listens on port `80` (due to `-e PORT=80`).
    *   Caddy forwards requests to `host.docker.internal:50051`. (If you're on Linux and `host.docker.internal` doesn't work, try replacing it with `172.17.0.1:50051` or the appropriate IP of your host on the Docker bridge network).

4.  **Test the proxy:**
    You can now send gRPC requests to `localhost:8080`, and they will be proxied to your service at `localhost:50051`.

    For example, using `grpcurl` (a command-line tool for interacting with gRPC services):
    ```bash
    # Install grpcurl if you don't have it: https://github.com/fullstorydev/grpcurl
    # Assuming your Greeter service has a method like 'SayHello' 
    # in a package 'your.package.greeter'.
    grpcurl -plaintext localhost:8080 your.package.greeter.Greeter/SayHello
    ```
    (Replace `your.package.greeter.Greeter/SayHello` with the actual fully qualified service name and method for your gRPC service.)
