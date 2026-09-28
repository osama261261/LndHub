# AGENTS.md

## Cursor Cloud specific instructions

LndHub is a custodial Lightning wallet HTTP API (Express, Babel-transpiled) that wraps a
single `lnd` node and stores per-user accounts/balances in Redis. It is the backend behind
BlueWallet's custodial Lightning wallets.

The update script only runs `npm install`. The backing services below (Redis, bitcoind, lnd)
are installed once into the VM image and are NOT started by the update script — a future agent
must start them before running LndHub.

### Services and how to run them

| Service | Required? | Notes |
|---|---|---|
| Redis | Yes | Account/token/balance store. LndHub calls `process.exit(5)` on boot if it can't reach Redis. |
| lnd (regtest) | Yes | gRPC on `127.0.0.1:10009`. LndHub `process.exit`s on boot (codes 3/4) if lnd is unreachable, and `lightning.js` throws at require-time if `tls.cert`/`admin.macaroon` are missing. |
| bitcoind (regtest) | Backend for lnd | Chain backend for lnd. LndHub itself treats `config.bitcoind` as optional (omitted here), so on-chain tx listing falls back to lnd. |

Startup order matters: bitcoind → lnd → redis → LndHub. Long-running processes are kept in
tmux sessions (`bitcoind`, `lnd`, `redis`, `lndhub`). Config files already exist at
`~/.bitcoin/bitcoin.conf` and `~/.lnd/lnd.conf`.

Start the backing services (idempotent — skip any already running; check with
`redis-cli ping`, `bitcoin-cli getblockchaininfo`, `lncli --network=regtest getinfo`):

```bash
redis-server --save "" --appendonly no --daemonize yes
bitcoind -daemon                 # regtest, RPC on 127.0.0.1:18443
lnd &                            # regtest; wallet auto-unlocks via LndHub config password
```

The lnd wallet is already created (password `password01`). After an lnd restart the wallet is
locked; LndHub auto-unlocks it because the config `lnd.password` is set. If you ever need to
recreate the wallet, use lnd's REST init API (`GET /v1/genseed` then `POST /v1/initwallet`) —
`lncli create` needs a TTY and fails when its stdin is piped.

If bitcoind has 0 blocks (fresh chain), mine some so lnd can sync:
`bitcoin-cli -rpcwallet=miner getnewaddress` then `bitcoin-cli generatetoaddress 101 <addr>`.

### Running LndHub

Config is supplied via the `CONFIG` env var (JSON that fully replaces `config.js`; do not edit
`config.js`). `tls.cert` and `admin.macaroon` must sit in the repo root — copy them from
`~/.lnd/tls.cert` and `~/.lnd/data/chain/bitcoin/regtest/admin.macaroon` (both are gitignored).

```bash
cd /workspace
export CONFIG='{"redis":{"host":"127.0.0.1","port":6379,"db":0},"lnd":{"url":"127.0.0.1:10009","password":"password01"}}'
export HOST=::   # bind dual-stack; see note below
npm run dev      # nodemon hot-reload; listens on :::3000
```

Port-forwarding gotcha: `index.js` defaults `HOST` to `0.0.0.0` (IPv4 only). On this VM
`localhost` resolves to IPv6 `::1` first, and the Cursor browser port-forwarder dials
`localhost`, so an IPv4-only bind yields `ERR_CONNECTION_REFUSED` in the browser even though
`curl 127.0.0.1:3000` works. Start with `HOST=::` so the server binds dual-stack and is
reachable over both `127.0.0.1` and `[::1]`/`localhost`.

`npm run dev` requires `nodemon`, which is MISSING from `package.json` and is installed
globally in this VM instead. If `nodemon` is not found, either reinstall it globally
(`npm install -g nodemon`, needs the system Node so run as root) or just use `npm start`
(same babel-node source run, no auto-reload). `npm start` here is a source run, not a
production build — the production Docker image transpiles via `npm run dockerbuild` first.

Lint: `npm run lint` (eslint + prettier). Note it runs with `--fix` and will rewrite two files
that have pre-existing formatting drift (`controllers/api.js`, `scripts/show_user.js`); revert
those with `git checkout --` if you only wanted to check. There is no automated test suite
(`npm test` is a placeholder that exits 1); acceptance tests live in the BlueWallet repo.

### Smoke test (hello world)

Only `POST /create` and `POST /auth` are Redis-only; everything else hits lnd. Authenticated
requests use `Authorization: Bearer <access_token>`.

```bash
curl -sX POST 127.0.0.1:3000/create -d '{}' -H 'Content-Type: application/json'   # -> {login,password}
curl -sX POST 127.0.0.1:3000/auth -d '{"login":"..","password":".."}' -H 'Content-Type: application/json'  # -> {access_token,..}
curl -s 127.0.0.1:3000/getinfo -H "Authorization: Bearer <access_token>"
curl -sX POST 127.0.0.1:3000/addinvoice -H "Authorization: Bearer <access_token>" -d '{"amt":"1000","memo":"hi"}' -H 'Content-Type: application/json'
```
