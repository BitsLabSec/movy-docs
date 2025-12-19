# Installtion

Depending on your usage, you could use movy in several ways.

## Use Movy as a Tool

It is the most common usage and we recommend using Movy via docker.

```bash
docker run --rm movyrs/movy --help
```

This shall print the help menu of Movy.

Additionally, you could also build `Movy` locally.

```bash
# Install dependencies
apt install -y libssl-dev libclang-dev

# Build movy
git clone https://github.com/BitsLabSec/movy
cd movy && cargo build --release
```

This takes several minutes even on a powerful machine.

## Use Movy as a Library

Add this to your `Cargo.toml`

```toml
movy = {git = "https://github.com/BitsLabSec/movy", branch = "master"}
```