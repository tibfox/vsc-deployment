# Security notes (review2)

## What changed in `docker-compose.yml`

### #12 Network segmentation (keystone, no operator action)
Previously every container shared one flat bridge, so compromising any one
(or watchtower) gave a path to the unauthenticated Mongo and the btcd RPC.
Now:

| Network | Internet | Members |
|---------|----------|---------|
| `dbnet` | no (`internal`) | go-vsc-node, init, mongo |
| `btcnet` | yes (btcd needs BTC P2P) | go-vsc-node, btcd |
| `proxynet` | no (`internal`) | watchtower, docker-socket-proxy |

mongo can no longer be reached from btcd/watchtower; watchtower has no
network path to the app containers at all.

### #11 watchtower no longer mounts the docker socket
A raw read-write `/var/run/docker.sock` bind in watchtower is a full
host-takeover primitive. Watchtower now talks to a `docker-socket-proxy`
(deny-by-default; only `CONTAINERS/IMAGES/NETWORKS` reads + `POST` for
container recreation) over the internal `proxynet`. Only the proxy mounts
the socket, read-only.

### #10 MongoDB host port removed
The `127.0.0.1:27021:27017` publish exposed an **unauthenticated** MongoDB
to every local user/process on the host. The node uses `mongo:27017` over
`dbnet`, not this port, so it was removed with no functional impact. For
admin access: `docker compose exec mongo mongosh`.

**Recommended further hardening — enable MongoDB auth.** Not forced here
because turning on `--auth` against an existing `./data/vsc-db` locks the
node out until a user exists. Migration for a fresh deploy:

1. Add to `.env`: `MONGO_USER=…`, `MONGO_PASS=…`
2. Add to the `mongo` service: `command: ["--auth"]` and
   `environment: MONGO_INITDB_ROOT_USERNAME/PASSWORD` (only seeds on an
   empty data dir).
3. Update `MONGO_URL` for `init` + `go-vsc-node` to
   `mongodb://<user>:<pass>@mongo:27017/?authSource=admin`.
   For an existing data dir, first create the user via
   `docker compose exec mongo mongosh` while still unauthenticated, then
   apply steps 2–3 and recreate (`docker compose up -d`, not `restart`).

### #9 / #56 btcd RPC
RPC was `-rpcallowip=0.0.0.0/0` with `vsc-node-user:vsc-node-pass`
hardcoded on the command line (readable via `ps` / `docker inspect`). Now
`-rpcallowip` is scoped to the pinned `btcnet` subnet (`172.28.7.0/24`,
only go-vsc-node), and the credentials are env-overridable. **Set unique
values in `.env`:**

```
BTC_RPC_USER=<random>
BTC_RPC_PASS=<random>
```

and mirror them in the node's BTC RPC configuration. Defaults are kept only
so existing deployments don't break on upgrade — they are not safe to leave.

## Host-level items (NOT in this repo)

- **#55 `PermitRootLogin yes`** is in the host's `/etc/ssh/sshd_config`,
  outside this deployment repo. Set `PermitRootLogin no`, use a non-root
  sudo user + key-only auth. Run the stack as a non-root user; never place
  `./data` on a world-writable path (see README step 5).

## #51 Image pinning vs. watchtower auto-update

`vscnetwork/go-vsc-node:main` is a **mutable tag by design** — watchtower
auto-pulling new node releases is the intended deployment workflow for
testnet/mainnet nodes. Digest-pinning it would disable that, so it is an
**operational policy decision for the operator**, not a blanket fix:

- Keep `:main` + watchtower **only if** you trust the registry/publisher
  and want hands-off node upgrades. Risk: a compromised/rolled `:main`
  is auto-deployed.
- For change-controlled environments, pin `go-vsc-node` to an immutable
  `@sha256:<digest>` (or a versioned tag) and update deliberately;
  watchtower will then leave it alone.

What this branch **does** fix: `containrrr/watchtower` was itself
unpinned (floating `:latest`) — the component that controls all updates
must not silently update itself. It is now pinned to `1.7.1`. The
remaining images are version-tagged (`mongo:8.0.17`,
`docker-socket-proxy:0.3.0`, `bitcoin/bitcoin:29.3`); pin them to
digests too if your threat model requires byte-for-byte reproducibility.
