# Open Data Hub - Data Connect Hub FIPS Cryptographic Backend

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-03 |
| Scope          | Data Connect Hub |
| Status         | Draft |
| Authors        | [Loic Marchal](@loicmarchal) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | ODH-ADR-0001-data-connect-hub, ODH-ADR-0002-bring-your-own-connector |

## What

Data Connect Hub's Rust services will perform their cryptography inside the FIPS 140-3 validated OpenSSL provider shipped in the RHEL/UBI9 base image, rather than in a cryptographic library statically linked into our binaries.

Mechanically, this means replacing the `aws-lc-rs` rustls crypto provider with [`rustls-ossl`](https://github.com/latchset/kryoptic/tree/main/rustls/ossl), a rustls `CryptoProvider` implementation that routes every cryptographic primitive to OpenSSL 3's provider layer. The TLS protocol implementation (rustls) and the gRPC stack (tonic, arrow-flight) are unchanged.

## Why

Data Connect Hub must be deployable on OpenShift clusters installed in FIPS mode. Running *on* such a cluster is not sufficient, and this is the single most commonly misunderstood point about the work:

* The kernel flag `/proc/sys/crypto/fips_enabled` governs the host kernel.
* Containers inherit the host kernel but **not** its userspace.
* A cryptographic library statically linked into our image is therefore entirely unaffected by it. It will keep using non-approved algorithms — correctly, silently, and without raising any error.

Both services are currently in exactly that state. They statically link `rustls` with the `aws-lc-rs` backend and install it explicitly at startup (`main()` in both service binaries, and `build_tls_acceptor()` in `services/flight/src/lib.rs`). None of that code path touches the system's OpenSSL.

A second constraint shapes which module is usable: all FIPS 140-2 certificates moved to the CMVP Historical list in September 2026, so only a module holding an **Active 140-3** certificate can support a new claim.

## Goals

* Cryptographic operations in `rest-service` and `flight-service` are performed by a FIPS 140-3 validated module.
* The validated module is Red Hat's, obtained from the base image — no third-party certificate is asserted by Data Connect Hub.
* No bespoke TLS transport. The existing rustls/tonic/arrow-flight stack is retained.
* The property is **provable** at runtime and in CI, not assumed. A build that cannot reach the validated module must be detectable before release.
* Services fail closed: a service that cannot establish FIPS operation when required must refuse to start rather than degrade silently.
* A single shipped image, containing only the connectors able to operate under the validated module. Excluded connectors remain available to customers through the bring-your-own-connector mechanism (ODH-ADR-0002) rather than through a second distributed build.

## Non-Goals

* **Claiming that Data Connect Hub is FIPS validated.** This is not a shortfall of the approach. FIPS 140-3 validates *cryptographic modules* — bounded artifacts with a defined cryptographic boundary and Security Policy — not the products that use them. RHEL and OpenShift are not themselves validated either; their constituent modules are, and they ship into FedRAMP High on that basis. Certificate #4857 belongs to Red Hat's OpenSSL provider, and asserting it as our own is what CMVP rules prohibit.

  The mandates require that a system *use* validated cryptography, not that every application hold a certificate, so the permitted claim discharges the requirement in full: what is excluded is the wording, not the outcome. See *Security and Privacy Considerations* for the permitted and prohibited phrasings.
* **Shipping a second, non-FIPS image carrying the full connector set.** The shipped image will contain only the connectors that can operate under the validated module; connectors that cannot are dropped rather than preserved in a parallel build. Maintaining two images means two dependency graphs, two sets of CI and container builds, and a per-release decision about which connectors belong in which — a standing cost for a configuration that no FIPS-mode customer can use. Customers needing a connector that is excluded can supply it through the bring-your-own-connector mechanism (ODH-ADR-0002). This is a deferral rather than a permanent position: a non-FIPS variant may be introduced later if demand warrants the duplication.
* **QUIC / HTTP/3 support.** `rustls-ossl` does not implement rustls' QUIC interface. Data Connect Hub uses gRPC over HTTP/2 and HTTP/1.1 only, with no QUIC or HTTP/3 anywhere in the workspace, so this is not applicable.

## How

### The provider architecture

OpenSSL 3 relocated its cryptographic implementations into swappable *providers*, one of which is the FIPS-validated provider. Reaching a provider requires calling OpenSSL through its modern `EVP_*` API. OpenSSL also retains the legacy OpenSSL 1.x low-level API (`RSA_*`, `EC_KEY_*`, `DH_*`) for compatibility, and those entry points dispatch to built-in implementations **without** passing through the provider layer.

rustls separates the TLS protocol from the cryptography and allows the latter to be replaced through its `CryptoProvider` interface. `rustls-ossl` implements that interface on top of the [`ossl`](https://crates.io/crates/ossl) crate, which binds only the modern EVP API:

```text
tonic / arrow-flight
  -> rustls            (TLS protocol: handshake, state machine, record layer)
    -> CryptoProvider  (pluggable)
      -> rustls-ossl
        -> ossl        (EVP-only bindings)
          -> OpenSSL 3 provider layer
            -> fips provider  (CMVP certificate #4857)
```

tonic cooperates with this without modification: it resolves `CryptoProvider::get_default()` first and only falls back to a compiled-in backend when no provider has been installed (tonic 0.14.6).

### Evidence that the module is present and enforcing

Verified directly, by inspecting the images and by running the provider checks on a FIPS-enabled OpenShift cluster:

| Claim | Evidence |
| --- | --- |
| A current validated module exists | CMVP certificate **#4857**, "Red Hat Enterprise Linux 9 — OpenSSL FIPS Provider", FIPS 140-3, **Active**, covering RHEL 9.2–9.8 |
| A UBI9 image carries the certified module | `ubi9/ubi-minimal:9.7` ships `openssl-fips-provider-3.0.7-8.el9.x86_64`, which reports module version `3.0.7-395c1a240fbfffd8` — an exact match against the certificate |
| The provider loads and enforces | `fips` provider active; MD5 rejected with `inner_evp_generic_fetch:unsupported` |
| The cluster kernel is in FIPS mode | `/proc/sys/crypto/fips_enabled` reports `1` |
| **The current runtime base carries the certified module** | **No.** See below — unresolved. |

Note that the RPM release and the module version are different identifiers. Red Hat rebuilds the provider package while keeping the certified binary module unchanged, so `-6.el9` and `-8.el9` both report `3.0.7-395c1a240fbfffd8`. **The module version string is the authoritative identifier**, not the package release.

#### Unresolved: the runtime base image does not carry the certified module

The runtime stage moved to `registry.redhat.io/ubi9/ubi-minimal-pqc` for post-quantum support. That image ships `openssl-fips-provider-3.0.7-11.el9_8`, which reports module version **`3.0.7-cda111b5812c30d4`** — a different binary module, appearing on no published certificate. A "RHEL 9 — OpenSSL FIPS Provider" entry is on CMVP's Modules In Process list, plausibly the revalidation for it, but *in process is not validated*. The certified build is not installable from the UBI repositories.

Two observations:

- **In FIPS mode the PQC base provides no benefit.** Post-quantum algorithms arrived in OpenSSL 3.5; the validated provider is frozen at 3.0.7 and cannot offer ML-KEM. Under FIPS mode the default property query routes to that provider, so PQC algorithms are unavailable regardless of base image. The PQC variant's value is for non-FIPS deployments.
- Pinning the runtime base to an image carrying a certified module — `ubi9/ubi-minimal:9.7` today — restores coverage with a one-line change, but is a holding position rather than a durable answer, since each new RHEL minor raises the question again.

Until this is settled, Data Connect Hub **cannot claim to utilize a validated module in the image it currently ships**, irrespective of the crypto backend decision recorded here. The decision itself is unaffected: it concerns how our code reaches the system provider, not which build of that provider the base image happens to carry.

Two further details are easy to get wrong and are recorded deliberately:

* **The validated boundary is `openssl-fips-provider`, not `openssl-libs`.** Red Hat ships them as separate packages on purpose: the provider is version-frozen at 3.0.7 to remain inside the certificate, while `openssl-libs` advances independently (3.5.1 in the image above). Any pin or runtime assertion must target the provider package.
* **An active `default` provider is not a failure signal.** Enforcement comes from the default property query (`fips=yes`) on the library context. The `base`, `default` and `fips` providers can all be loaded simultaneously while non-approved algorithms are still correctly refused.

### Changes to the services

**Dependencies.** `rustls-ossl` replaces `aws-lc-rs` as the crypto provider. The tonic TLS feature changes from `tls-aws-lc` to `tls-native-roots` (`services/rest/Cargo.toml`, `services/flight/Cargo.toml`), which enables tonic's TLS support without compiling in a Rust crypto backend — tonic declares `tokio-rustls` with `default-features = false`, so no backend is forced. The workspace `rustls` and `tokio-rustls` declarations drop their `aws_lc_rs` features. `arrow-flight` requires no change: it already declares tonic with `default-features = false` and no TLS features.

**Provider installation.** The two startup install sites — `main()` in each service binary — switch to `rustls_ossl::default_provider()`. The provider is restricted before installation so that every component it contains reports FIPS status — see *Key exchange groups* below.

**Flight listener.** `flight-service` already constructs its own `rustls::ServerConfig` through `builder_with_provider`, derives a provider to enforce the configured cipher suites, sets ALPN `h2`, and passes the resulting `tokio_rustls::TlsAcceptor` to tonic via `serve_with_incoming_shutdown` (`build_tls_acceptor()` and `start_server()` in `services/flight/src/lib.rs`). This is the pattern that gives us direct control over the rustls configuration, and it means no change to tonic's API surface is required. The legacy `ServerTlsConfig` path in `configure_tls()` is to be consolidated onto the same route.

**Builder image.** `openssl-devel` is added to the builder stage of both Containerfiles. The runtime stage already contains the provider by virtue of the UBI9 base image.

### Key exchange groups and Extended Master Secret

`CryptoProvider::fips()` is an `.all()` across every configured component (rustls 0.23.45): a single component reporting `false` makes the whole provider report `false`. `rustls-ossl` reports per-algorithm status by querying the live OpenSSL context for actual availability (`src/hash.rs`, `src/hmac.rs`, `src/keyex.rs`) rather than returning a single global flag, which is the behaviour we want — but the resulting group list must still be verified on-cluster rather than assumed.

This matters more than a reporting detail, because rustls derives its Extended Master Secret requirement from it:

```rust
// rustls 0.23.45: ServerConfig::builder() and ClientConfig::builder()
let require_ems = self.provider.fips();
```

EMS (RFC 7627) is mandatory under **FIPS 140-3 Implementation Guidance section D.Q, "Transition of the TLS 1.2 KDF to Support the Extended Master Secret"** — a TLS 1.2 connection without it uses a KDF that is no longer approved. Because the flag is derived from the provider, a provider that reports non-FIPS silently disables EMS as well. Getting the provider to report correctly is therefore a correctness requirement, not cosmetics.

### Proving it, rather than trusting it

The failure mode this decision must defend against is specific: cryptography that executes outside the validated module returns **correct results**. Handshakes succeed, tests pass, logs are clean, and FIPS status may still be reported as true. It is undetectable by functional testing.

Two controls address this:

1. **Negative test against the `base` provider.** CI runs the TLS test suite against an OpenSSL configuration in which only the `base` provider — which implements no algorithms — is active. Any operation correctly routed through the provider layer *must* fail under that configuration. An operation that succeeds was computed outside any provider and is a defect. This technique is borrowed from `rustls-openssl`'s own test suite and works regardless of which provider crate is used.
2. **Fail-closed startup assertion.** When FIPS operation is required, services assert `ServerConfig::fips()` / `ClientConfig::fips()` at startup and refuse to start otherwise. Note this is the correct assertion: `CryptoProvider::fips()` covers only the primitives, while the config-level call additionally accounts for TLS protocol-level requirements such as EMS.

`check-payload` image scanning provides a third, independent check at image level. Dynamic linkage to the system OpenSSL is the case it recognises natively as compliant.

## Open Questions

* Which runtime base image carries a certified module? The current base ships an uncertified provider build (see *Unresolved* above), and the PQC and FIPS goals appear mutually exclusive while the validated provider remains frozen at OpenSSL 3.0.7.
* Does `rustls-ossl`'s key exchange group list resolve to an all-FIPS set under RHEL's FIPS provider, or must groups be restricted explicitly before installing the provider? Its X25519 branch reports *availability* in the OpenSSL context as its FIPS answer, which conflates "the provider offers this" with "this is approved."
* Can `neo4rs` be redirected to the system TLS implementation, or must the `neo4j` connector be excluded from the shipped image?
* Does PostgreSQL password authentication remain usable under FIPS, or is client-certificate authentication required?

## Alternatives

### `aws-lc-rs` with the `fips` feature

rustls' own upstream-supported FIPS path: `rustls = { features = ["fips"] }` statically links AWS-LC-FIPS, covered by CMVP certificate #4816.

**Tradeoffs:** The smaller code change by some margin — tonic would need no modification at all, the crypto provider *is* the validated module so no binding layer can misroute calls, and the semantics are documented upstream. Rejected on compliance grounds rather than engineering grounds. Using a third-party validated module places the burden of evidence on that vendor, and it leaves an unanswered question about whether compiling the module from source in our own UBI9 builder keeps us inside a *tested operational environment* on certificate #4816 or drops us to vendor affirmation — a question that surfaces at audit rather than at build time. It would also require a `fips_certified_modules` entry to be accepted by `check-payload`, whereas system OpenSSL linkage is recognised natively. Red Hat compliance was consulted and advised against this option.

### `rustls-openssl`

An alternative rustls `CryptoProvider` backed by OpenSSL, maintained independently ([tofay/rustls-openssl](https://github.com/tofay/rustls-openssl)).

**Tradeoffs:** Considerably more mature — ten releases since November 2024 and roughly 354,000 downloads, against `rustls-ossl`'s single `0.0.1` release. However it is built on the `openssl` crate, which wraps both the modern EVP API and the deprecated OpenSSL 1.x low-level API, so provider misrouting is possible by construction. This is not hypothetical: until September 2026 its ECDHE key generation called `EC_KEY_generate_key`, generating private keys outside the module boundary on every P-256/P-384 handshake while handshakes completed normally. It has since migrated to provider APIs for OpenSSL 3 targets. Additionally, all of its FIPS status methods return the same process-global flag, so the same class of defect would not be reflected in its reporting. Retained as the fallback if `rustls-ossl` proves unsuitable; the two are interchangeable at the same interface.

### Hand-built `tokio-openssl` transport for tonic

Terminate TLS with OpenSSL directly and hand tonic pre-encrypted streams through `connect_with_connector` and `serve_with_incoming`.

**Tradeoffs:** Initially assessed as the only route. Substantially more work than any alternative considered, and it carries a permanent cost: implementing `tonic::transport::server::Connected`, negotiating ALPN `h2` manually, forgoing `ClientTlsConfig`/`ServerTlsConfig` entirely, and owning a transport layer whose internals have changed across recent tonic major versions. Unnecessary once the pluggable-provider route was identified: rustls remains the TLS implementation and only the cryptography is redirected.

### Platform TLS termination

Move TLS off the `rest-service` → `flight-service` hop entirely, terminating at a service mesh sidecar as `rest-service`'s inbound traffic already does.

**Tradeoffs:** Would remove tonic from the problem completely. Rejected as the primary mechanism because it reassigns responsibility rather than discharging it — the auditor's question becomes whether the mesh's cryptography is validated — and because it would require the deployment to mandate a sidecar, leaving any non-mesh installation carrying plaintext gRPC. Remains available as a defence-in-depth measure and is not precluded by this decision.

## Security and Privacy Considerations

* **Permitted claim language.** Data Connect Hub *utilizes* the Red Hat Enterprise Linux 9 OpenSSL FIPS provider (CMVP certificate #4857). It must not be described as "FIPS validated", "FIPS compliant", or "FIPS enabled" — the first asserts a certificate that belongs to Red Hat, and the latter two are prohibited outright by FedRAMP in authorization documentation. This constraint applies to release notes, user documentation and UI text, not only to formal compliance submissions. See *Non-Goals* for why validation of the product itself is not available.
* **The unqualified claim depends on removing statically linked cryptography.** Until the RustCrypto paths described under *Risks* are eliminated, it is not true that *all* of Data Connect Hub's cryptographic operations are performed by the validated module, and the claim must be qualified accordingly. Removing them is therefore what unlocks the strongest permissible statement, not merely a dependency cleanup.
* **Silent failure is the dominant risk.** Cryptography performed outside the validated module produces correct output and working connections. No functional test detects it. The negative-test control described under *How* is the primary mitigation and should be treated as a release gate rather than an optional check.
* **Extended Master Secret.** Because rustls derives `require_ems` from the provider's FIPS status, a provider that under-reports its status will also disable a mandatory protocol-level FIPS requirement. The two failures are coupled.
* **Reduced algorithm set.** Operating under the FIPS provider restricts the available cipher suites and key exchange groups. Interoperability with external data sources that cannot negotiate an approved suite will fail at connection time; this is the intended behaviour and should be documented per connector.

## Risks

* `rustls-ossl` is at version `0.0.1` with a single published release and a correspondingly small user base, so defects are more likely to surface here first than in a widely deployed crate. Accepted on the basis that its author is a Red Hat engineer who has confirmed it is suitable for production use, that the dependency is pinned to an exact version, and that `rustls-openssl` remains a drop-in fallback at the same interface.
* This decision does not itself remove statically linked Rust cryptography, which bypasses OpenSSL entirely and is unaffected by FIPS mode. That removal is required and is planned as separate work; it is called out here because adopting the validated module exposes the gap rather than closing it. Known paths:

| Path | Removal |
| --- | --- |
| `sha1` ← `actix-http` ← `actix-web` | A feature flag. `sha1` is an optional dependency of `actix-http`, gated behind `ws`, which `actix-web` enables by default. Nothing in the workspace uses WebSockets, so building `actix-web` with `default-features = false` and the default list minus `ws` removes it. |
| `sha1` ← `reqsign` ← `opendal` | Object-storage request signing, used by the `s3` and `uri` connectors. Removed by excluding those connectors from the shipped image, which this decision already anticipates. |
| `md-5` ← `sqlx-postgres`, `sha1` ← `sqlx-mysql` | The hard case. These are authentication implementations rather than TLS, so switching sqlx to `tls-native-tls` does not touch them. MD5 authentication is compiled in regardless of whether SCRAM-SHA-256 is used at runtime, and linked-but-unused may still be flagged by manifest-based scanning. `sqlx-mysql`'s presence appears to be incidental — only `postgres` and `sqlite` are declared — and may be removable. |

  An earlier revision of this record described the `actix-http` path as structural and the largest uncertainty in the estimate. That was wrong: it is gated behind an optional feature. The remaining uncertainty is sqlx.
* Dropping connectors from the shipped image is a user-visible regression for anyone relying on one today, and it shifts the build-and-maintain cost onto those customers via bring-your-own-connector. The set of excluded connectors must therefore be settled and communicated before the FIPS image becomes the default, and the bring-your-own-connector path must be usable in practice — not merely documented — by that point.
* `rustls-ossl` is not rustls' upstream-supported FIPS path. The two authorities point in opposite directions — rustls recommends certificate #4816, Red Hat recommends its own module — and this decision consciously follows Red Hat.
* From rustls 0.24, a crypto provider must be supplied explicitly when constructing `ClientConfig`/`ServerConfig` rather than installed process-wide. `rustls-ossl` currently constrains rustls to `>=0.23.37, <0.24`. The process-default installation pattern is therefore not a permanent shape.
* No build has yet been produced demonstrating Data Connect Hub binaries resolving their cryptography to the validated provider. Until that exists, every claim in this record is about the design rather than the artifact.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Data Connect Hub maintainers  |                  |            | Yes |
| ODH operator team             |                  |            | Yes |
| Red Hat compliance            | Jean-Philippe Jung | 2026-10-13 | Yes |
| Customers deploying on FIPS clusters |           |            | Yes |

## References

* ODH-ADR-0001-data-connect-hub — foundational architecture for Data Connect Hub
* ODH-ADR-0002-bring-your-own-connector — custom connector deployment model
* CMVP certificate #4857 — Red Hat Enterprise Linux 9 OpenSSL FIPS Provider
* <https://access.redhat.com/compliance/fips> — Red Hat FIPS compliance status
* <https://github.com/latchset/kryoptic/tree/main/rustls/ossl> — `rustls-ossl` crypto provider
* <https://docs.rs/rustls/latest/rustls/manual/_06_fips/index.html> — rustls FIPS guidance
* <https://github.com/openshift/check-payload> — OpenShift FIPS image scanner
* `docs/user-guide/connectors.md` (data-connect-hub) — per-connector FIPS-mode status

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Jean-Philippe Jung            | 2026-10-13 | Red Hat compliance. Advised against a third-party validated module; confirmed the RHEL OpenSSL module is validated and approved the approach. |
| Simo Sorce                    |            | Author of `ossl` and `rustls-ossl`. Confirmed the crate is suitable for this use; noted it does not support QUIC, which does not apply to Data Connect Hub. |
|                               |            |       |
