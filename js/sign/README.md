# Signing with Integrity Block

This is a Node.js module for signing
[Web Bundles](https://wpack-wg.github.io/bundled-responses/draft-ietf-wpack-bundled-responses.html)
with [integrityblock](../../explainers/integrity-signature.md).

The module takes an existing bundle file and an ed25519 private key, and emits a
new bundle file with cryptographic signature added to the integrity block.

## Installation

Using npm:

```bash
npm install wbn-sign
```

## Requirements

This plugin requires Node v14.0.0+.

## Usage

Please be aware that the APIs are not stable yet and are subject to change at
any time.

Signing a web bundle file:

```javascript
import * as fs from 'fs';
import * as wbnSign from 'wbn-sign';

// Read an existing web bundle file or generate using "wbn" npm package.
const webBundle = fs.readFileSync('./path/to/webbundle.wbn');

const privateKey = wbnSign.parsePemKey(
  fs.readFileSync('./path/to/privatekey.pem', 'utf-8')
);

// Option 1: With the default (`NodeCryptoSigningStrategy`) signing strategy.
const { signedWebBundle } = await new wbnSign.IntegrityBlockSigner(webBundle, {
  key: privateKey,
}).sign();

// Option 2: With specified signing strategy.
const { signedWebBundle } = await new wbnSign.IntegrityBlockSigner(
  webBundle,
  new wbnSign.NodeCryptoSigningStrategy(privateKey)
).sign();

// Option 3: With ones own CustomSigningStrategy class implementing
// ISigningStrategy.
const { signedWebBundle } = await new wbnSign.IntegrityBlockSigner(
  webBundle,
  new (class {
    async sign(data: Uint8Array): Promise<Uint8Array> {
      // E.g. connect to one's external signing service that signs the payload.
    }
    async getPublicKey(): Promise<KeyObject> {
      /** E.g. connect to one's external signing service that returns the public
       * key.*/
    }
  })()
).sign();

fs.writeFileSync(signedWebBundle);
```

This library also exposes a helper class to calculate the Web Bundle's ID (or
App ID) from the private or public ed25519 key, which can then be used when
bundling
[Isolated Web Apps](https://github.com/WICG/isolated-web-apps/blob/main/README.md).

Calculating the web bundle ID for Isolated Web Apps:

```javascript
import * as fs from 'fs';
import * as wbnSign from 'wbn-sign';

const privateKey = wbnSign.parsePemKey(
  fs.readFileSync('./path/to/privatekey.pem', 'utf-8')
);

// Web Bundle ID only:
const webBundleId = new wbnSign.WebBundleId(privateKey).serialize();

// With origin, meaning "isolated-app://" combined with Web Bundle ID:
const webBundleIdWithIWAOrigin = new wbnSign.WebBundleId(
  privateKey
).serializeWithIsolatedWebAppOrigin();
```

## CLI

This package also includes a CLI tool `wbn-sign` which lets you sign a web
bundle easily without having to write any additional JavaScript.

Example command:

```bash
wbn-sign \
-i ~/path/to/webbundle.wbn \
-o ~/path/to/signed-webbundle.swbn \
-k ~/path/to/ed25519key.pem
```

An unencrypted ed25519 private key can be generated with:

```
openssl genpkey -algorithm Ed25519 -out ed25519key.pem
```

For better security, one should prefer using passphrase-encrypted ed25519
private keys. To encrypt an unencrypted private key, run:

```
# encrypt the key (will ask for a passphrase, make sure to use a strong one)
openssl pkcs8 -in ed25519key.pem -topk8 -out encrypted_ed25519key.pem
# delete the unencrypted key
rm ed25519key.pem
```

If you use an encrypted private key, you will be prompted for its passphrase as
part of the signing process. If you want to use the CLI tool programmatically,
then you can bypass the passphrase prompt by storing the passphrase in an
environment variable named `WEB_BUNDLE_SIGNING_PASSPHRASE`.

## Release Notes

### v0.1.1

- Add support for bypassing the passphrase prompt for encrypted private keys by
  reading the passphrase from the `WEB_BUNDLE_SIGNING_PASSPHRASE` environment
  variable if it is set when using the CLI tool directly.

### v0.1.0

- BREAKING CHANGE: Introducing the support for using different types of signing
  strategies. Will enable users to initialize a SigningStrategy class
  (implementing the newly introduced `ISigningStrategy` interface). Also `sign`
  changes to be an async function.
- Add support for using passphrase-encrypted Ed25519 private keys.

### v0.0.1

- Support for signing web bundles with
  [integrity block](https://github.com/WICG/webpackage/blob/main/explainers/integrity-signature.md)
  added.
