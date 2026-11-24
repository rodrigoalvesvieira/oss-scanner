# MagicMtn pins Rust 1.100.0 in rust-toolchain.toml. Trixie provides the Python 3.13 the sandbox runtime uses.
# Line-table debug info and four build jobs keep linking the test binaries within the memory of a small machine.
FROM docker.io/library/node:24-trixie-slim AS node

FROM docker.io/library/rust:1.100.0-trixie

ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_TERM_COLOR=never \
    CARGO_INCREMENTAL=0 \
    CARGO_BUILD_JOBS=4 \
    CARGO_PROFILE_DEV_DEBUG=line-tables-only \
    CARGO_PROFILE_TEST_DEBUG=line-tables-only \
    RUST_BACKTRACE=1 \
    RUSTFLAGS="-C force-frame-pointers=yes"

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential ca-certificates clang git openssh-client pkg-config python3 python3-venv sqlite3 \
    && rm -rf /var/lib/apt/lists/*

# Node.js 24 runs the Pi extension, OpenCode plugin, and sandbox runtime tests.
COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
    && node --version && npm --version

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

# Fetch every crate in both lockfiles so later builds work offline.
RUN cargo fetch --locked \
    && cargo fetch --locked --manifest-path sdk/examples/minimal-provider/Cargo.toml

# Build the engine, local server, CLI, SDK crates, and their tests as debug binaries.
# The `magic` CLI is at /src/target/debug/magic.
RUN cargo build --workspace --all-targets --locked \
    && cargo build --locked --all-targets --manifest-path sdk/examples/minimal-provider/Cargo.toml

# JavaScript and Python dependencies for the agent integrations and the in-sandbox runtime.
RUN npm ci --prefix interfaces/pi --ignore-scripts --no-audit --no-fund \
    && npm ci --prefix runtime --ignore-scripts --no-audit --no-fund \
    && python3 -m venv /opt/runtime-venv \
    && /opt/runtime-venv/bin/pip install --no-cache-dir -r runtime/requirements.txt

ENV CARGO_NET_OFFLINE=true

# Smoke-test the image. Failures are reported in the build log without aborting the build.
RUN (timeout 1500 cargo test --locked -p magicmtn-cli -p magicmtn-local -p magicmtn-engine --lib --bins --tests -- --test-threads=1 \
      || echo "WARNING: cargo tests reported failures") \
    && (npm --prefix interfaces/pi test || echo "WARNING: Pi extension tests reported failures") \
    && (npm --prefix runtime test || echo "WARNING: runtime JavaScript tests reported failures") \
    && (/opt/runtime-venv/bin/python -m unittest discover -s runtime/tests \
      || echo "WARNING: runtime Python tests reported failures")
