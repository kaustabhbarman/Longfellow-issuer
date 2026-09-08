# Longfellow-issuer

Standalone OpenID4VCI issuer for mdoc/SD-JWT credentials, extracted from
[Multipaz](https://github.com/openwallet-foundation/multipaz) at tag **0.99.0**.

The two issuer modules (`multipaz-openid4vci` library + `multipaz-openid4vci-server`)
are vendored as source; all other Multipaz modules are consumed as published
Maven artifacts at 0.99.0.

## Why pinned at 0.99.0

Multipaz [PR #1904](https://github.com/openwallet-foundation/multipaz/pull/1904)
(2026-08-13) makes the issuer add `keyAuthorizations` to every MSO's
`deviceKeyInfo`. The Google Longfellow ZK circuits (v6 and v7) cannot generate
proofs over such MSOs — ZKP presentment fails with
`MDOC_PROVER_GENERAL_FAILURE` (error code 6) while classic presentment keeps
working. Version 0.99.0 predates that change, so credentials issued by this
server are ZKP-compatible.

If you ever update the vendored modules past that commit, remove the
`deviceKeyAuthorizedNamespaces = listOf(PingTransaction.mdocResponseNamespace)`
argument in `multipaz-openid4vci/.../CredentialFactoryMdl.kt`.

## The `base_url` setting

The issuer embeds `base_url` in its metadata
(`/.well-known/openid-credential-issuer`) and in every credential offer, so it
must be **the URL the wallet device itself can reach** — not just a URL that
works on the machine running the server. Every setup below differs only in what
that URL is.

Without `base_url`, it defaults to `http://localhost:<server_port>`, which only
works for a client on the same machine. The server binds `0.0.0.0:8007`, so it
is reachable on loopback and over the LAN at the same time.

## Running with ngrok (https, works everywhere)

```sh
ngrok http 8007          # in one terminal
./run-local-issuer.sh https://YOUR-NGROK-URL
```

This is the simplest option for a phone that is not on your network, and the
only one that needs no wallet-side changes (see the cleartext note below).

## Running locally without ngrok

The server speaks **plain HTTP only** — it has no TLS support of its own. That
matters because Android blocks cleartext HTTP by default: any wallet targeting
API 28+ without an explicit exception (Multipaz's sample wallet included) will
refuse to talk to `http://` URLs. So a no-ngrok setup needs a one-time wallet
change, described at the end of this section.

### Desktop browser only (no wallet)

To browse the web UI, inspect metadata, or use the admin page:

```sh
./gradlew :multipaz-openid4vci-server:run
```

Then open <http://localhost:8007>. Provisioning into a wallet will not work with
this `base_url`, since `localhost` on the phone means the phone itself.

### Android emulator

The emulator reaches the host machine at the special address `10.0.2.2`:

```sh
./run-local-issuer.sh http://10.0.2.2:8007
```

Open `http://10.0.2.2:8007` in the emulator's browser to get a credential offer.

An alternative that keeps the URL as `localhost` is an adb reverse tunnel, which
forwards the device's port 8007 to the host's:

```sh
adb reverse tcp:8007 tcp:8007
./gradlew :multipaz-openid4vci-server:run     # base_url defaults to http://localhost:8007
```

The tunnel is dropped when the device disconnects, so re-run `adb reverse` after
a reconnect.

### Physical device on the same network

Use the host machine's LAN address, e.g. found with:

```sh
ipconfig getifaddr en0        # macOS, Wi-Fi
hostname -I                   # Linux
```

Then start the server with that address:

```sh
./run-local-issuer.sh http://192.168.0.237:8007
```

Both devices must be on the same network, with no client isolation on the
access point, and the host firewall must allow inbound connections on port 8007.

### Wallet-side prerequisite: allow cleartext HTTP

All of the HTTP setups above require the wallet to permit cleartext traffic to
that host. In the wallet project, add `composeApp/src/androidMain/res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>
        <domain includeSubdomains="false">localhost</domain>
        <domain includeSubdomains="false">192.168.0.237</domain>
    </domain-config>
</network-security-config>
```

and reference it from the `<application>` tag in
`composeApp/src/androidMain/AndroidManifest.xml`:

```xml
android:networkSecurityConfig="@xml/network_security_config"
```

Scope the exception to these development hosts rather than enabling
`android:usesCleartextTraffic="true"` globally, and keep it out of any build you
distribute.

## Notes

- Admin page: `/admin.html` (password `meinewallet-admin`, set in
  `run-local-issuer.sh`).
- On first credential issuance the server enrolls its document-signing (DS)
  certificate against `https://issuer.multipaz.org/records` (the Multipaz
  default `enrollment_server_url`), so the first run needs internet access even
  in a local setup. The DS private key never leaves the machine.
- Server state (keys, DS certificate, issuance sessions) lives in a local
  database created at runtime and is gitignored; deleting it makes the server
  re-enroll on next issuance.
- Changing `base_url` after credentials have been issued invalidates them for
  refresh purposes, since issuer URLs are recorded in the wallet.
