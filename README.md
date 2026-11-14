# helm-chart-audiobookshelf

Deploys [audiobookshelf](https://github.com/advplyr/audiobookshelf), a self-hosted audiobook and podcast server, from its official image.

The chart runs one replica:

- A Deployment that stops the old pod before starting the new one, since SQLite allows one server and the claims are `ReadWriteOnce`.
- Two PersistentVolumeClaims: `<release>-config` for the SQLite database and `<release>-metadata` for covers, cache, logs and backups.
  audiobookshelf's database mustn't be on network storage, so the config claim's class needs to be local or block storage.
- The libraries listed in `libraries`, each mounted from its own volume, read-only only when asked.
- A ClusterIP Service on 80, and an optional Gateway API `HTTPRoute` to it.
- Probes on `/healthcheck`, with 5 minutes for a first start, since database migrations run before the server listens.

audiobookshelf runs as the image's root user: it has no setting to run as another, and that lets it write library folders whoever owns them.
It exposes no Prometheus metrics, so the chart has no `ServiceMonitor` or dashboards.

## Values

| Value | Default |
| --- | --- |
| `image.repository` | `ghcr.io/advplyr/audiobookshelf` |
| `image.tag` | the chart's `appVersion` |
| `resources` | requests `cpu: 25m`, `memory: 128Mi`; no limits |
| `persistence.config` | `size: 1Gi`, the cluster's default class |
| `persistence.metadata` | `size: 5Gi`, the cluster's default class |
| `libraries` | none |
| `affinity` | none |
| `httpRoute.enabled` | `false` |
| `httpRoute.parentRefs` | none; required when enabled |
| `httpRoute.hostnames` | none |

Each `persistence` entry also takes `labels`, added to its claim's own, for example to opt it into a storage system's backups.

Each entry in `libraries` has a `name`, a `mountPath`, an optional `readOnly`, and a `volume`, any Kubernetes volume source.
audiobookshelf asks that no mount is inside another, so keep `mountPath` outside `/config` and `/metadata`.

## Requirements

- A storage class for the claims, the cluster's default unless named.
- The Gateway API's `HTTPRoute` CRD, when `httpRoute.enabled` is set.

## Usage

```sh
helm repo add audiobookshelf https://tonylea.github.io/helm-chart-audiobookshelf
helm install audiobookshelf audiobookshelf/audiobookshelf --values my-values.yaml
```

```yaml
libraries:
  - name: audiobooks
    mountPath: /audiobooks
    readOnly: true
    volume:
      hostPath:
        path: /srv/audiobooks
        type: Directory
```

## Development

Pull requests run:

- `helm lint --strict`, and the rendered chart checked by kubeconform against Kubernetes 1.37.
- helm-unittest's tests in `tests/`.
- An install and upgrade with chart-testing in a kind cluster, with the values in `ci/kind-values.yaml`.

Commits follow Conventional Commits.
Releases are versioned and tagged automatically from `main`, and published to the Helm repository on this repo's GitHub Pages.
