---
name: Send a searcher bundle to Harmonic
description: Authenticate with a Solana keypair and submit an atomic transaction bundle to Harmonic's block engine, with revert protection and no protocol fee.
api: grpc/harmonic-gg-searcher-searcher.proto
operations: [GenerateAuthChallenge, GenerateAuthTokens, RefreshAccessToken, SendBundle]
---

# Send a searcher bundle to Harmonic

Harmonic is an open block-building system for Solana. Searchers submit atomic
transaction bundles over a gRPC interface that is backwards-compatible with Jito's
searcher protos. Use this skill to authenticate and land a bundle.

## Prerequisites

- Your searcher **pubkey must be whitelisted** by the Harmonic team first
  (apply: https://form.typeform.com/to/aUd0FiwD). Un-whitelisted keys are rejected
  with `PERMISSION_DENIED`.
- The protobuf definitions: `git submodule add https://github.com/harmonic/searcher-protos.git`.
- Pick the closest regional Block Engine endpoint (e.g. `https://fra.be.harmonic.gg`);
  Harmonic cross-forwards to all regions, so send to one region only.

## Steps

1. **Request a challenge.** Call `auth.AuthService/GenerateAuthChallenge` with
   `role = Role.SEARCHER (3)` and your 32-byte pubkey. Note: Harmonic's Searcher
   role is `3`, not `1` as in jito-protos — if reusing jito-protos, authenticate
   with the ShredstreamSubscriber role (= 3).
2. **Sign and exchange.** Sign `"{pubkey}-{challenge}"` with your keypair, then call
   `GenerateAuthTokens` with the challenge, your `client_pubkey`, and the 64-byte
   `signed_challenge`. You receive an `access_token` and a longer-lived `refresh_token`.
3. **Attach the Bearer token.** Send the access token as a gRPC Bearer credential on
   every request. Refresh it with `RefreshAccessToken` ~5 minutes before `expires_at_utc`;
   re-run steps 1–2 when the refresh token expires.
4. **Build the bundle.** Construct a `bundle.Bundle` with an optional `shared.Header`
   timestamp and a list of `packet.Packet` (each `data` = a serialized transaction).
   Tips are Solana **compute-unit priority fees** on the transactions themselves — do
   NOT add a Jito-style tip-transfer instruction; 100% of the tip goes to the validator.
5. **Optionally protect transactions.** To prevent frontrunning, embed a Bundle Control
   Account: attach a pubkey whose base58 address starts with `dontfront` (or `dontbund1e`)
   as the first account on a Compute Budget instruction in the transaction to protect.
6. **Submit.** Call `searcher.SearcherService/SendBundle` with the bundle. The response
   returns a server-assigned `uuid`.

## Error handling

- `PERMISSION_DENIED` — pubkey not whitelisted or wrong Searcher role.
- `UNAUTHENTICATED` — missing/expired access token; refresh and retry.
- **Bundle dropped** — if any transaction reverts, the whole bundle is dropped
  (revert protection) and no fees are charged; fix and resubmit.
- **Bundle rejected** — a Bundle Control Account directive was violated (e.g. a
  `dontfront`-protected transaction was not placed at index 0).

See `conventions/harmonic-gg-conventions.yml` and `errors/harmonic-gg-error-codes.yml`.
