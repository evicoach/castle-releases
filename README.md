# castle releases

**castle** reads the logs of services running in secured environments, such as pilot, pre-production and production on private AKS clusters, from your own terminal. Those clusters' Kubernetes API is not reachable from the internet for security reasons, so `kubectl` on your laptop cannot read their logs. castle connects through Azure Bastion and a jump box with your own Entra ID sign-in, and keeps the clusters private. `clogs` is the short alias.

This repository only hosts the release downloads.

## Install

| Platform | Command |
|---|---|
| macOS / Linux (Homebrew) | `brew install evicoach/tap/castle` |
| Windows (Scoop) | `scoop bucket add evicoach https://github.com/evicoach/scoop-bucket` then `scoop install castle` |
| Manual | Download the archive for your OS from [Releases](https://github.com/evicoach/castle-releases/releases) and put `castle` and `clogs` on your PATH |

## Get started

```
az login
clogs init --from <team config>    # or: clogs init
clogs <service> -f                 # follow a service's logs
clogs help
```

You also need the Azure CLI, `kubectl` and `kubelogin` (`az aks install-cli`). Run `clogs doctor` to check your setup.
