+++
authors = ["Jan Ligudziński"]
title = "In which I fix a nasty production system: #2 - More CI and Docker Compose cleanup"
description = "We might be getting somewhere."
date = 2025-10-10
[taxonomies]
tags = ["nasty system", "programming", "rust", "devops", "docker", "compose", "war story"]
+++

## Where we left off

In the [previous post in this series](../1-nasty-system-1), I introduced you to a nasty system I'd inherited from past me, and we managed to make it slightly less nasty by CI-ifying its simplest component - an almost-static landing page and overhauling the reverse proxy infrastructure routing different subdomains of our business' main domain to different components - we yote Nginx and replaced it with the much simpler Caddy[^1].
This leaves us with some more things to do before we can move on to proper CD:
- Push the main web app and its AI agent companion into our org's Docker registry on GitHub as we did the landing page
- Have our docker-compose refer to the images in the registry rather than locally building them from source and dockerfiles we push via SSH
- Move all the service definitions into the main `docker-compose.yml` we made the last time, and nuke the old source/dockerfile directories

## Agent app

### Dockerfile optimization

Let's start with the AI agent app. It doesn't directly depend on the database or the Redis cache[^2], so as long as it can access the API, we can extract into the main dockerfile as we please. Let's take a look at how it gets built:

```
FROM rust:1.89-slim-bookworm

RUN apt-get update && \
    apt-get install -y \
    pkg-config \
    libssl-dev \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY Cargo.toml Cargo.lock ./
COPY api/Cargo.toml ./api/
COPY domain/Cargo.toml ./domain/

# Create dummy source files to build dependencies
RUN mkdir -p api/src domain/src
RUN echo "fn main() {}" > api/src/main.rs && \
    echo "pub const dummy: i32 = 0;" > domain/src/lib.rs

# Build dependencies (this layer will be cached unless Cargo files change)
RUN cargo build --release --bin api

RUN rm -rf api/src domain/src

# Copy actual source code
COPY . .

# Touch the files to bust cache

RUN touch api/src/main.rs domain/src/lib.rs

# Build the application with real source code
RUN cargo build --release --bin api

ENV RUST_LOG=info
EXPOSE ${PORT:-5099}

CMD ["/usr/src/app/target/release/api"]
```

This is honestly kind of suboptimal - some low-hanging fruit here would be separating the build stage and the runtime stage to minimize the size.

>*What's with the `touch` command and the manual deletion and creation of the dummy files?*

There's a little war story behind that. As I hinted at in the first post, I try to keep my backend apps segregated into clean-architecture-ish layers: a `domain` that defines the basic entities and rules in play, an `application` that actually does stuff with them, an `infrastructure` that contains all the ugly external-world-touching stuff like database drivers or HTTP clients, and an `api` that exposes the application to actual users. I could do that all as modules of one big Rust crate, but having multiple crates that share a *workspace* to cache common dependencies is much better for compile times.
The problem is that Cargo doesn't really have a dedicated command for "just download and build the dependencies our actual app will use, which rarely change" and if my Dockerfile just said "copy all the source code before building", changing just our own code would trigger a redownload and rebuild of absolutely everything involved.

This we would not want. So, after we pull in all the `Cargo.toml` manifests that specify our deps, we create dummy `lib.rs` and `main.rs` files so Cargo can say "this is a valid Rust project that will compile to actual libraries and executables, we can go ahead with `build`", then after the dependency fetching and building we swap in the real source code. Why the `touch`? Docker checks if a file has changed by looking at its modification time, not its size or actual contents, so if we don't manually update that, Docker will just keep the dummy files in its cache and build useless binaries again.

>*That sounds a lot more complicated than it should be. Haven't they fixed this in Cargo by now?*

As far as I know, no. But now that you point it out, there *is* a more efficient way to do this thing, and it's called [Chef](https://github.com/LukeMathWalker/cargo-chef).
Why wasn't I using it already? Your guess is as good as mine, I think I was just too fatigued after finally figuring out a flow that worked to do things all over again. Another thing you might notice is that we don't do a multi-stage build where the final container is just the executable and whatever it needs at runtime. Let's maybe remedy both of these problems at once:

```
# ---------- Build stage ----------
FROM rust:1-slim-bookworm AS chef

# Get cargo-chef
RUN cargo install cargo-chef

WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
# Build deps
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  pkg-config \
  libssl-dev \
  ca-certificates \
  binutils \
  && rm -rf /var/lib/apt/lists/*

COPY --from=planner /app/recipe.json recipe.json
# Build dependencies
RUN cargo chef cook --release --recipe-path recipe.json

# Build actual app
COPY . .
RUN cargo build --release --bin api

# ---------- Runtime stage ----------
FROM debian:bookworm-slim AS runtime

# Only the runtime bits you need for an OpenSSL-linked binary
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  libssl3 \
  ca-certificates \
  && rm -rf /var/lib/apt/lists/*

# Run as non-root
RUN useradd -u 10001 -r -g nogroup -s /usr/sbin/nologin -d / nonroot

WORKDIR /app
COPY --from=builder /app/target/release/api /app/api

ENV RUST_LOG=info

# EXPOSE doesn't support default values directly; use ARG + ENV for flexibility
ARG PORT=5099
ENV PORT=${PORT}
EXPOSE ${PORT}

USER 10001
ENTRYPOINT ["/app/api"]

```

### CI

Then we get to copy-paste the build and push job we used in the landing page repo:

```yaml
name: build and push

on:
  push:
    branches: ["master"]
  pull_request:
    branches: ["master"]

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: element-group-com-pl/goldenhand-golem

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/master' }}
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: false
```

We push this to origin and, because this time we didn't pre-create the `goldenhand-golem` package in GHCR by manually pushing, it gets automatically created with the repo's action already allowed to push to it.
Unfortunately, one advantage of our (horrible, very bad, no good) deployment flow with the images getting built locally was that we wouldn't have to re-pull the Rust images and dependencies on every build thanks to the caching layer tricks we'd used and now Cargo Chef, and this advantage is now lost to us, but that's not a bad price to pay for having actual CI. *Maybe* in the future we'll reimplement that in some more civilized way if we care enough.

### Integration into central compose file

Now, we can add compose entries for the two instances of `golem` we want to run:

```yaml
golem-prod:
  image: ghcr.io/element-group-com-pl/goldenhand-golem:latest
  pull_policy: always
  env_file:
    - ./golem/.env.prod
  ports:
    - "5100:5100"
  restart: unless-stopped
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:5100/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
  networks:
    - webnet
# ... same for golem-demo, but with another port and env file
```


And after pushing the new `main-machine` compose file and the requisite .envs to said machine we can scrap the old source directory and its compose:

```
root@ubuntu-8gb-fsn1-1:~# cd goldenhand-golem
root@ubuntu-8gb-fsn1-1:~/goldenhand-golem# docker-compose down
Stopping goldenhand-golem_agent-demo_1 ... done
Stopping goldenhand-golem_agent-prod_1 ... done
Removing goldenhand-golem_agent-demo_1 ... done
Removing goldenhand-golem_agent-prod_1 ... done
Removing network goldenhand-golem_default
root@ubuntu-8gb-fsn1-1:~/goldenhand-golem# cd ../main-machine
root@ubuntu-8gb-fsn1-1:~/main-machine# docker-compose up -d
Creating network "main-machine_default" with the default driver
Pulling golem-prod (ghcr.io/element-group-com-pl/goldenhand-golem:latest)...
latest: Pulling from element-group-com-pl/goldenhand-golem
5c32499ab806: Pull complete
f7dca50eac3a: Pull complete
2a2d4acc790a: Pull complete
8d9507874b7c: Pull complete
8aefda335d2e: Pull complete
Digest: sha256:7dd9ae7be4ea7690ef4deeca27cd069571bb3153045a25ec27fe7cef686847dd
Status: Downloaded newer image for ghcr.io/element-group-com-pl/goldenhand-golem:latest
main-machine_caddy_1 is up-to-date
main-machine_landing-page_1 is up-to-date
Creating main-machine_golem-prod_1 ... done
Creating main-machine_golem-demo_1 ... done
root@ubuntu-8gb-fsn1-1:~/main-machine# http GET https://call.ets-group.pl/health
HTTP/1.1 200 OK
Alt-Svc: h3=":443"; ma=2592000
Content-Length: 33
Content-Type: application/json
Date: Fri, 10 Oct 2025 18:53:22 GMT
Via: 1.1 Caddy

{
    "status": "ok",
    "version": "0.1.0"
}


root@ubuntu-8gb-fsn1-1:~/main-machine# http GET https://call-demo.ets-group.pl/health
HTTP/1.1 200 OK
Alt-Svc: h3=":443"; ma=2592000
Content-Length: 33
Content-Type: application/json
Date: Fri, 10 Oct 2025 18:53:26 GMT
Via: 1.1 Caddy

{
    "status": "ok",
    "version": "0.1.0"
}


root@ubuntu-8gb-fsn1-1:~/main-machine# cd ..
root@ubuntu-8gb-fsn1-1:~# rm -r goldenhand-golem
root@ubuntu-8gb-fsn1-1:~# ls
goldenhand-rs  goldenhand-rs-demo  main-machine
```

One down, two to go. Next up: the main web app.


[^1]: Behind the scenes: Traefik was also a possible solution, since it's apparently tailor-made for our use case of routing to multiple Docker containers, but Caddy's easy one-file config won out. Also, I can see a way I could use it locally for development when I need HTTPS for something (currently, we have a compile-time switch that makes the API serve over HTTPS in development; outsourcing that concern to Caddy would mean we get to delete code, which is what every developer loves most).
[^2]: The early prototype I'd built in C# before the Realtime API had even dropped functioned purely over multiple HTTP requests where Twilio would call a "start_call" endpoint and we'd respond with a TwiML `<Gather>` asking it to collect user audio and give us a text transcription at another endpoint, "advance_conversation", which then responded with a `<Say>` containing the LLM's response and another `<Gather>` asking to do the same thing again. In between the different requests we'd store the conversation state in Redis. Currently, as the conversation is a persistent connection, we can do this stuff in memory (and as the LLM can use tools like "hang up", we also don't need to do hacky things like asking the LLM to output a blob of everything-JSON containing its next sentence, a "conversation should end" flag, etc).
