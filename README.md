# castle

**Read the logs of services running in secured environments, from your own terminal.**

Secured environments such as pilot, pre-production and production run on private AKS clusters. For security reasons their Kubernetes API is not reachable from the internet, so `kubectl` on your laptop cannot read their logs. The usual workaround is to sign in to a jump box inside the cloud network and run commands there.

castle makes that hop for you. It connects through Azure Bastion and the jump box with your own Entra ID sign-in, and lets you view, follow and search those services' logs in your local terminal. The clusters stay private. `clogs` is the short alias.

```
clogs orders-service -f                        # follow live
clogs payments-api --env preprod --since 1h --grep ERROR
clogs apps --env pilot                         # every service, as copyable commands
clogs help                                     # usage + the saved service lists
```

## Connection

- The first command connects in the background (about 6 s). Every later command, from any terminal, reuses that connection (about 2 s).
- It closes itself after 15 idle minutes (`idle_minutes`) and reconnects on the next command if it drops.

## Security

- **The clusters stay private.** castle opens nothing to the internet, and the connection can only be used from your own machine.
- **Your identity, your permissions.** Every request carries your own Entra ID token, so you see only what your Kubernetes RBAC allows. There are no shared accounts and no stored passwords or tokens.
- **Short-lived keys.** The SSH key and certificate Bastion needs are issued for one hour, kept in a private folder, and deleted when the connection closes.
- **Read-only.** castle itself only lists deployments and reads logs. Anything else you run through `clogs kubeconfig` is limited by your own RBAC.
- **Nothing left behind.** The connection closes after the idle timeout; nothing is installed or stored on the jump box.

## Requirements

**Azure access**

- An Azure Bastion with native client support (Standard or Premium SKU, tunnelling enabled), and Reader on it.
- A Linux VM with Entra ID SSH login to use as the jump box: Reader on the VM and its network interface, and Virtual Machine User Login (or Administrator Login).
- Permission to read pod logs in the namespaces you need, for example Azure Kubernetes Service RBAC Reader.

**Tools.** castle checks for these on every run and prints the install command for anything missing. It installs the Azure CLI extensions it needs (`bastion`, `ssh`) itself.

| Tool | macOS | Windows | Linux |
|---|---|---|---|
| Azure CLI | `brew install azure-cli` | `winget install -e --id Microsoft.AzureCLI` | [aka.ms/azure-cli](https://aka.ms/azure-cli) |
| kubectl | `brew install kubectl` | `winget install -e --id Kubernetes.kubectl` | `az aks install-cli` |
| kubelogin | `brew install Azure/kubelogin/kubelogin` | `winget install -e --id Microsoft.Azure.Kubelogin` | `az aks install-cli` |
| OpenSSH client | built in | Settings > System > Optional features > OpenSSH Client | your distribution's `openssh-client` |

## Install

| Platform | Command |
|---|---|
| macOS / Linux (Homebrew) | `brew install evicoach/tap/castle` |
| Windows (Scoop) | `scoop bucket add evicoach https://github.com/evicoach/scoop-bucket` then `scoop install castle` |
| Manual | Download the archive for your OS from [Releases](https://github.com/evicoach/castle-releases/releases) and put `castle` and `clogs` on your PATH |

Check: `clogs version` prints the version.

## Set up

1. Sign in to Azure: `az login --tenant <your tenant domain>`
2. Set up castle, with a team config if your team has one (see "Share a team config"):
   - `clogs init --from <team config file or URL>`, or
   - `clogs init` to discover everything from your own access, or
   - `clogs init --jumpbox <vm-name>` when several VMs could be the jump box.
3. Check everything: `clogs doctor` ends with `All good.`

`init` takes about 30 seconds. It finds the Bastion, the jump box, the private clusters the jump box can reach, and the app namespaces you may read logs in, then groups them into environments from their names (`pilot`, `dev`, `prod`, …). A deployment whose name carries another environment, such as `orders-service-preprod` in a dev cluster, becomes its own environment.

## Use

| To | Run |
|---|---|
| Read the last 100 lines | `clogs orders-service` |
| Follow live (Ctrl+C to stop) | `clogs orders-service -f` |
| Another environment | `clogs orders-service --env preprod` |
| Only recent lines | `clogs orders-service --since 30m` |
| Only matching lines | `clogs orders-service --grep ERROR` |
| More lines per pod | `clogs orders-service --tail 500` |
| One namespace only | `clogs orders-service -n backend-ns` |
| List services as copyable commands | `clogs apps`, `clogs apps --env dev`, `clogs apps kyc` |
| Usage and the saved service lists | `clogs help` |
| Connection status / close it now | `clogs status` / `clogs stop` |
| Use the connection with kubectl, k9s or Lens | `clogs kubeconfig` |
| Check and fix your setup | `clogs doctor` |

A service name can be the full deployment name (`invoice-sync-deployment`), the name without `-deployment` (`invoice-sync`), or any unique part of it (`invoice`). When a name matches several services, castle lists them. Run castle in as many terminals as you like; they share one connection.

## Config

`~/.config/castle/config.yaml` (Windows: `%APPDATA%\castle\config.yaml`). Discovery fills it in; edit anything and your values win.

```yaml
connection:
  bastion: auto            # or a Bastion name / resource ID
  jumpbox: auto            # or a VM name / resource ID
default_environment: pilot
idle_minutes: 15
environments:
  dev:
    namespaces:            # cluster/namespace: group label shown by `clogs apps`
      my-dev-aks/backend-ns: Backend
    exclude: preprod       # leave out deployments whose name matches (regex)
  preprod:
    namespaces:
      my-dev-aks/backend-ns: Backend
    only: preprod          # keep only deployments whose name matches (regex)
```

- `clogs init` again after infrastructure changes; it keeps your edits and lists newly found namespaces.
- `clogs init --reset` rebuilds `environments` from discovery.
- `discovered.yaml` and `kubeconfig` next to it are written by castle; don't edit them.

## Share a team config

1. One person who is set up runs `clogs export team.yaml`. The file holds the connection, the environments and each cluster's host name and certificate authority, so teammates also reach clusters Azure does not list for them.
2. Share the file. It holds no secrets; everyone still signs in with their own Entra ID account and permissions.
3. Each teammate saves it in their home folder and runs `clogs init --from ~/team.yaml` (Windows: `$HOME\team.yaml`).

`--from` also accepts a URL that needs no sign-in to download.

## Update

| Platform | Command |
|---|---|
| macOS / Linux | `brew update && brew upgrade castle` |
| Windows | `scoop update castle` |

## Troubleshooting

| You see | Do this |
|---|---|
| `missing tools castle needs:` and a list | Run each `install:` command shown, open a new terminal, then run the command again. |
| `Azure login expired, opening browser sign-in...` | Sign in in the browser; the command continues. |
| `AADSTS…` error | `az login --tenant <your tenant domain>`, then run the command again. |
| `No module named …` | `clogs doctor`; it reinstalls the broken Azure CLI extension. |
| `Which jump box?` or `found several to choose from as the jump box` | Pick your team's jump box, or run `clogs init --jumpbox <vm-name>` or `clogs init --from <team config>`. |
| `could not open the Bastion connection` | Check the Azure access in "Requirements". |
| `local port … is in use` | `clogs stop`, then run the command again. |
| `no service matching "…"` | `clogs apps --env <name>` and copy the right line. |
| `doctor`: `you may not read logs here` | Ask for log access to that cluster. |

## License

MIT
