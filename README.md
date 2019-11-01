# Configure a Holo HoloPortOS Instance

Deploying a basic configuration to a HoloPortOS instance requires key generation material and basic
identity + password to be collected and deployed.

## USB Configuration

The simplest and most direct method is to generate a configuration, and copy it onto a USB stick,
which is then inserted into the HoloPortOS instance.  When the device boots, it will:

- Use the data on the USB stick to create its Holochain and potentially other keys
- Authenticate itself to the Holo ZeroTier network, which will provision its DNS configuration
- Start the Holo services
- Eject the USB and blacklist the kernel modules

## Building & Generating a `holo-config.json`

We will generate a `Config` object in JSON form, to be saved into `holo-config.json`:

```
$ nix-build -A holo-config-generate-cli
$ ./target/debug/holo-config-generate-cli  --email "a@b.ca" --password "secret" | tee holo-config.json
```

Also available is the nix-shell and manual build approach:
```
$ nix-shell
$ cargo build --release --bin holo-config-generate-cli

$ ./target/release/holo-config-generate-cli --email "a@b.ca" --password "secret" | tee holo-config.json
https://hcscjzpwmnr6ezxybxauytg458vgr6t8nuj3deyd3g6exybqydgsz38qc8n3zfr.holohost.net/
{
  "v1": {
  "seed": "jYvZ70UkYJGjMzADb4PcQzHcECLfUHHXb9KMk6NY2fE",
  "admin": {
    "email": "a@b.ca",
    "public_key": "4sfPilERj9dPCkTADmJ8MfsUkfXOxWOlPHhhtVuzlt4"
  }
}
```

## Building a Web UI to Generate Config

Each UI can build and ship exactly the subset of the Rust `holo-config` project required to support
its functionality.  We do not ship a "standard" JS library, but instead allow the Web UI developer
to write a very small Rust API calling holo-config code, which is compiled to a small WASM static
asset included with and called by the Web UI project.

For example, the provided `generate-web` example generates a JSON string containing a holo-config,
from a supplied email and password.  The Rust code in [./generate-web/src/lib.rs] is compiled using
the Javascript `@wasm-tool/wasm-pack-plugin`.

A very simple Javascript `index.js` loads the compiled WASM package, in [./generate-web/index.js].

When the Webpack-compiled page is loaded, the DOM is configured by the above Javascript, and the
WASM code is invoked on the Generate button-click, producing the `holo-config.json` file.

### Building the WASM and JS

To build an example web UI, able to call a WASM-compiled function that can generate and return a
`Config` in JSON form suitable for saving to `holo-config.json`:

```
$ nix-shell
$ cd generate-web
$ npm install
$ npm run build
$ npm run serve
```

Go to `http://localhost:8080`, type in an email and password, and click `Generate`, and save the
file.  Will default to saving a file named `holo-config.json` to your downloads directory.

## Generating a Holochain Keystore from `holo-config.json`

To use the seed saved in `holo-config.json` from within a Holochain application (for example, upon
start-up of the Holochain Conductor on the HoloPort), the Config needs to be deserialized, and the
seed used in the standard Holochain cryptography routines.

Standard Rust Serialize/Deserialize functionality is provided:

```
use holo_config_core::{config::Seed, Config}
...
let Config::V1 { seed, .. } = serde_json::from_reader(stdin())?;
```

Generate a `holo-config.json`, and use `holo-config-derive` to load it and generate a Holochain
keystore:

```
$ nix-shell
$ cargo build --release --bin holo-config-derive < holo-config.json
$ ./target/release/holo-config-derive < holo-config.json
HcSCjwu4wIi4BawccpoEINNfsybv76wrqoJe39y4KNAO83gsd87mKIU7Tfjy7ci
{
  "passphrase_check": "eyJzY...0=",
  "secrets": {
    "primary_keybundle:enc_key": {
      "blob_type": "EncryptingKey",
      "seed_type": "Mock",
      "hint": "",
      "data": "eyJzY...0="
    },
    "primary_keybundle:sign_key": {
      "blob_type": "SigningKey",
      "seed_type": "Mock",
      "hint": "",
      "data": "eyJzYW...19"
    },
    "root_seed": {
      "blob_type": "Seed",
      "seed_type": "OneShot",
      "hint": "",
      "data": "eyJzYW...19"
    }
  }
}
```
