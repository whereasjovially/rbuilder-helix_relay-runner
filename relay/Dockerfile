# Stage 1: Build the Helix binary
FROM rust:1.72.0 as helix

RUN apt update -y && apt install -y clang protobuf-compiler wget

# Install sccache for caching Rust builds
RUN wget https://github.com/mozilla/sccache/releases/download/v0.3.1/sccache-v0.3.1-x86_64-unknown-linux-musl.tar.gz \
    && tar xzf sccache-v0.3.1-x86_64-unknown-linux-musl.tar.gz \
    && mv sccache-v0.3.1-x86_64-unknown-linux-musl/sccache /usr/local/bin/sccache \
    && chmod +x /usr/local/bin/sccache

# Copy project files
ADD ./ /app/
RUN ls -lah /app
WORKDIR /app/

# Fetch dependencies and build the Helix project
RUN --mount=type=cache,target=/root/.cargo \
    --mount=type=cache,target=/usr/local/cargo/registry \
    cargo fetch

RUN --mount=type=cache,target=/root/.cargo \
    --mount=type=cache,target=/usr/local/cargo/registry \
    RUSTC_WRAPPER=/usr/local/bin/sccache cargo build -p helix-cmd --release

RUN mv /app/target/release/helix-cmd /app/helix-cmd

# Stage 2: Create the final container
FROM debian:stable-slim

RUN apt-get update && apt-get install -y ca-certificates

WORKDIR /app

# Copy the compiled binary and config file
COPY --from=helix /app/helix-cmd* ./
COPY config.yml ./config.yml
COPY network-config.yaml ./network-config.yaml  

# Download `wait-for-it.sh` from GitHub
RUN wget -q -O /usr/local/bin/wait-for-it.sh https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh \
    && chmod +x /usr/local/bin/wait-for-it.sh

    
# Copy the wait-for-it script from the host
COPY wait-for-it.sh /usr/local/bin/wait-for-it.sh
RUN chmod +x /usr/local/bin/wait-for-it.sh

# Set environment variables
ENV CONFIG_PATH="/app/config.yml"

# Ensure TimescaleDB extension is loaded (optional, for extra safety)
RUN echo "shared_preload_libraries = 'timescaledb'" >> /etc/postgresql/16/main/postgresql.conf || true

# Set the entrypoint to run the helix-cmd binary directly without arguments
ENTRYPOINT ["sh", "-c", "
    echo 'Waiting for PostgreSQL to be ready...';
    until pg_isready -h postgres -p 5432 -U postgres; do
      echo 'PostgreSQL not ready yet... retrying in 2 seconds...';
      sleep 2;
    done;
    echo 'PostgreSQL is ready!';
    /app/helix-cmd
"]

# Use the CMD to pass the config as an argument
CMD ["--config", "/app/config.yml"]