# Agent instructions for signet-playground

## What this is

A pure Docker Compose stack (no application source code to build/test/lint).
All "development" here means editing `compose.yml`, `bitcoin.conf`, `fulcrum.conf`,
or `.env`/`compose.override.yml` files and validating behavior by running the stack.
There is no compiler, package manager, or test suite in this repo.

## Running / validating changes

```bash
docker compose config          # validate compose.yml syntax after edits
docker compose up --wait        # start the stack (waits for healthchecks)
docker compose rm -f wallet-setup   # remove the one-shot wallet-setup container after it completes
docker compose logs <service>   # inspect a single service, e.g. node, miner, wallet-setup
docker compose down -v          # tear down containers AND volumes (full reset, needed after
                                 # changing signetchallenge, descriptors, or genesis-affecting config)
```

There is no way to "test a single unit"; the closest equivalent is restarting/recreating
a single service and checking its logs/healthcheck, e.g.:

```bash
docker compose up -d --force-recreate node
docker compose exec node bitcoin-cli --version
```

## Architecture (why services depend on each other)

The stack is a chain of `depends_on` conditions — understanding the order matters more
than any single file:

1. **node** (Bitcoin Knots, `-chain=signet`) starts and is healthy once its RPC cookie
   file exists (`test -f /var/tmp/.cookie`).
2. **wallet-setup** runs once `node` is healthy. It creates a *blank* descriptor wallet
   named `BBO` and imports three Taproot (`tr()`) descriptors derived from the same
   master xprv (see Wallet Organization below), then exits. It is not `restart`ed and
   is meant to be manually removed (`docker compose rm -f wallet-setup`) after success.
3. **miner** starts after `wallet-setup` completes successfully. It mines a signet block
   every `MAX_INTERVAL` (60s) using `MINING_XPUB`, sending each reward to a
   height-indexed address (see below).
4. **frigate** and **fulcrum** depend on `node` (healthy) and `miner` (started); they
   index the chain and expose Electrum-protocol servers. `frigate` proxies/augments
   `fulcrum` with Silent Payments support and talks to `node`'s ZMQ sequence stream.
5. **mempool-api** depends on `node`, `fulcrum`, and `mariadb` (all healthy); **mempool-web**
   depends on `mempool-api`. **faucet** depends on `node` (healthy) and `valkey` (healthy).
6. **mariadb** and **valkey** are plain data-store dependencies for the mempool backend
   and faucet.

All inter-service communication happens over the internal Docker network using service
names as hostnames (`node`, `fulcrum`, `mariadb`, `valkey`, etc.) — never `localhost`
inside a container's environment/command. The RPC cookie file is shared via the
`cookie_dir` named volume, mounted at `/var/tmp` in every service that needs to
authenticate to `node`'s RPC (this is why cookie-based auth works across containers
without exchanging a password).

Only ports explicitly listed under a service's `ports:` are reachable from the host
(`127.0.0.1:<port>`). Everything else is internal-only by design; to expose more,
add/extend `ports:` in `compose.yml` or use a local (gitignored) `compose.override.yml`
rather than editing the checked-in ports table casually — update the README ports table
in that case.

## Wallet / descriptor conventions (critical, don't break)

The `wallet-setup` entrypoint is intentionally fragile-looking but precise — do not
"simplify" it:

* Wallet is created with `blank=true` (no default descriptors), then exactly three
  Taproot (`tr(...)`) descriptors are imported from the same master tprv, at derivation
  paths `86h/1h/0h/0/*` (receiving, external, active), `86h/1h/0h/1/*` (change, internal,
  active), and `86h/1h/0h/2/*` (mining rewards, internal, **inactive**).
* The mining descriptor is marked inactive/internal on purpose so `bitcoin-cli
  getnewaddress` never returns mining-reward addresses (which would otherwise get reused
  by the miner script). The `miner` service's `MINING_XPUB` env var must correspond to
  the tpub of that same `86h/1h/0h/2/*` path — obtainable via
  `bitcoin-cli listdescriptors` after import.
* `signetchallenge` in `compose.yml` (`node` command) is hardcoded to the Taproot
  scriptPubKey of address `86h/1h/0h/2/0` (the mining descriptor's index 0), because
  block 0's coinbase is unspendable. If you ever regenerate the master xprv, you must
  recompute and update `signetchallenge` to match, and a full `docker compose down -v`
  reset is required (the chain is bound to this challenge from genesis).
* Keep the three `*_XPRV` values in `wallet-setup.environment` consistent with the tpub
  used by `miner.environment.MINING_XPUB` — they must derive from the same master key.

## Conventions for config files

* `bitcoin.conf` only carries settings `bitcoin-cli` needs client-side (chain, cookie
  path); node-server-side flags live in `compose.yml`'s `node.command`, not in
  `bitcoin.conf`. Keep new node flags in `compose.yml` for consistency.
* Prefer editing/adding fees, RPC, and indexing flags in `node.command` (the big
  multi-line `>` block) rather than introducing a second config source.
* Pin third-party image versions by digest (`image@sha256:...`) with the human-readable
  tag as a trailing comment (see `fulcrum`, `mariadb`, `valkey` services) — this is the
  established pattern for non-`1maa/*` images; the `1maa/*` images intentionally track
  `:latest`.
