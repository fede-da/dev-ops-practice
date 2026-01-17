FROM jenkins/jenkins:lts-jdk17

USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates git \
 && rm -rf /var/lib/apt/lists/*

# --- Install Docker CLI (static) for linux/arm64 ---
RUN curl -L "https://download.docker.com/linux/static/stable/aarch64/docker-27.5.1.tgz" -o /tmp/docker.tgz \
 && tar -xzf /tmp/docker.tgz -C /tmp \
 && mv /tmp/docker/docker /usr/local/bin/docker \
 && chmod +x /usr/local/bin/docker \
 && rm -rf /tmp/docker /tmp/docker.tgz

# --- Install Docker Compose v2 (standalone) for linux/arm64 ---
RUN curl -L "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-aarch64" \
  -o /usr/local/bin/docker-compose \
 && chmod +x /usr/local/bin/docker-compose

USER jenkins