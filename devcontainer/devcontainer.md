# How the dev container sandbox works

This dev container gives you a .NET 10 environment where you can run the GitHub Copilot CLI in
`--yolo` mode (see `prompt.md`) with less risk. The agent can do whatever it likes inside the
container, but outbound network traffic, root access and host credentials are restricted.

## Files

| File | Purpose |
|------|---------|
| `.devcontainer/devcontainer.json` | Container definition: features, capabilities, mounts, startup command, VS Code settings |
| `.devcontainer/Dockerfile` | Image build: firewall tooling, sudo lockdown, Copilot config directory |
| `.devcontainer/init-firewall.sh` | Default-deny egress firewall, run as root on every container start |
| `.devcontainer/devcontainer-lock.json` | Pins the `copilot-cli` feature to a specific version and SHA-256 digest |

## Lifecycle

1. **Build.** The `Dockerfile` starts from `mcr.microsoft.com/devcontainers/dotnet:1-10.0-noble`
   (Ubuntu 24.04 with the .NET 10 SDK), installs `dnsutils`, `ipset`, `iptables` and `jq`, and
   copies `init-firewall.sh` to `/usr/local/bin` (owned by root, mode `0755`).
2. **Feature install.** The `copilot-cli` feature is added. The lock file pins it to version
   `1.1.4` by digest, so a changed upstream feature does not get pulled in silently.
3. **Start.** The container runs with the `NET_ADMIN` and `NET_RAW` capabilities, which
   `iptables` and `ipset` need.
4. **Post-start.** `postStartCommand` runs `sudo /usr/local/bin/init-firewall.sh`.
   `"waitFor": "postStartCommand"` makes VS Code wait for the firewall before it treats the
   container as ready. The firewall runs on every start, not only on the first one.
5. **Work.** You (and the agent) work as the unprivileged `vscode` user.

## What it restricts

### 1. Network egress (`init-firewall.sh`)

The script sets up a default-deny firewall:

1. **Reset.** Flushes all `iptables` rules and recreates the `allowed-domains` ipset.
2. **GitHub ranges.** Fetches `https://api.github.com/meta` and adds every IPv4 CIDR from its
   `web`, `api` and `git` lists. Each entry is checked against an IPv4 regex before it is added.
3. **Allowlisted domains.** Resolves the A records of each domain in `ALLOWED_DOMAINS` once and
   adds the IPs:
   - **Copilot CLI:** `api.githubcopilot.com`, `api.{individual,business,enterprise}.githubcopilot.com`,
     telemetry endpoints, `copilot-proxy.githubusercontent.com`, `github.githubassets.com`,
     `avatars.githubusercontent.com`, `default.exp-tas.com`, and others
   - **NuGet restore:** `api.nuget.org`, `globalcdn.nuget.org`
   - **VS Code server and extensions:** `update.code.visualstudio.com`,
     `marketplace.visualstudio.com`, `vscode.blob.core.windows.net`
4. **Policy.**

   | Traffic | Result |
   |---------|--------|
   | Loopback (`lo`) | Allowed |
   | Replies on established/related connections | Allowed |
   | DNS (UDP/TCP 53) to the resolvers in `/etc/resolv.conf` | Allowed |
   | TCP 443 to an IP in `allowed-domains` | Allowed |
   | Any other outbound traffic (HTTP on 80, SSH on 22, other hosts on 443, ...) | Rejected (`icmp-admin-prohibited`, so it fails fast instead of timing out) |
   | New inbound connections | Dropped |
   | Forwarding | Dropped |
   | IPv6 | Everything dropped except loopback. If `ip6tables` is missing, a warning is printed and IPv6 is **not** filtered |

5. **Verification.** The script exits with an error if `https://example.com` is reachable, or if
   `https://api.github.com` is not. The container start then fails visibly instead of
   continuing with a broken firewall.

In practice the agent can talk to Copilot, restore NuGet packages and use GitHub over HTTPS. It
cannot download arbitrary scripts, call other APIs, or send data to an arbitrary server.

### 2. Root access (`Dockerfile`)

The base image gives `vscode` passwordless `sudo` for everything. The Dockerfile:

- deletes `/etc/sudoers.d/vscode`;
- adds `/etc/sudoers.d/vscode-firewall`, which allows `vscode` to run **only**
  `/usr/local/bin/init-firewall.sh` as root without a password;
- checks the sudoers syntax with `visudo -c`, and fails the build if `sudo -l -U vscode` still
  shows an `(ALL` grant.

The agent therefore cannot become root to flush the firewall, install packages with `apt`, or
change system files. The capabilities `NET_ADMIN`/`NET_RAW` are only usable by root, so the
`vscode` user cannot use them to change firewall rules.

### 3. Host credentials (`devcontainer.json`)

VS Code normally forwards several host credentials into a dev container. This one turns them off:

| Setting | Effect |
|---------|--------|
| `"remoteEnv": { "SSH_AUTH_SOCK": "" }` | No SSH agent forwarding, so the host's SSH keys cannot be used |
| `"dev.containers.copyGitConfig": false` | Your host `~/.gitconfig` is not copied in |
| `"dev.containers.gitCredentialHelperConfigLocation": "none"` | No Git credential helper that proxies to host credentials |

### 4. The sandbox configuration itself

```json
"source=${localWorkspaceFolder}/.devcontainer,target=${containerWorkspaceFolder}/.devcontainer,type=bind,readonly"
```

The `.devcontainer` folder is mounted again on top of itself, **read-only**. The rest of the
workspace stays writable, but the agent cannot edit `devcontainer.json`, the `Dockerfile` or the
firewall script to weaken the sandbox on the next rebuild.

### 5. Copilot login persistence

```json
"source=copilot-cli-config,target=/home/vscode/.copilot,type=volume"
```

Copilot CLI configuration and login are stored in a named Docker volume, so you do not have to
log in again after a rebuild. The Dockerfile creates `/home/vscode/.copilot` owned by `vscode`
so the volume gets the right ownership.

## What it does not restrict

The sandbox limits the impact of an agent's actions. It does not prevent all of them. Keep these
in mind:

- **The workspace is writable.** The agent can create, change and delete any file in the
  project, except `.devcontainer/`. Use Git to review and undo changes.
- **GitHub is fully reachable.** All of GitHub's web/API/git ranges are allowed, and the Copilot
  token lives in `/home/vscode/.copilot`. An agent could push code, create a gist or open an
  issue using that token, so data could leave through GitHub.
- **Shared IPs open more than the listed domains.** The firewall filters on IP address, not on
  host name. Hosts behind the same CDN or cloud IPs (for example other Azure Blob Storage
  accounts that share an IP with `vscode.blob.core.windows.net`) are reachable too.
- **DNS is open to the configured resolver.** Data could in principle be tunneled through DNS
  queries.
- **IP lists are a snapshot.** Domains are resolved once at start. If a CDN rotates its IPs,
  a service can stop working until you restart the container (which reruns the firewall).
- **Rerunning the script briefly opens the firewall.** `reset_rules` sets every policy to
  `ACCEPT` before the new rules are applied. Because `vscode` may run the script through `sudo`,
  there is a short window during a rerun where egress is unrestricted.
- **No resource limits.** CPU, memory and disk are not capped.
- **Code runs in the container.** Anything the agent builds (for example the ASP.NET Core API in
  `prompt.md`) runs with the `vscode` user's permissions. The container is the boundary, not the
  app.
- **The container shares the host kernel.** This is container isolation, not a VM. It protects
  against mistakes and opportunistic misuse, not against a kernel exploit.

## Changing the allowlist

1. Add the domain to `ALLOWED_DOMAINS` in `.devcontainer/init-firewall.sh`. From inside the
   container you cannot do this, because the folder is read-only there; edit it on the host.
2. Rebuild or restart the container (**Dev Containers: Rebuild Container**).
3. Check the post-start output for `init-firewall: egress restricted to allowlist`, and for any
   `no A records` warnings about domains that could not be resolved.

To check the firewall by hand inside the container:

```bash
curl -sS --max-time 5 https://example.com     # should fail
curl -sS --max-time 5 https://api.github.com/zen  # should succeed
sudo iptables -S                               # fails: only init-firewall.sh is allowed via sudo
```
