# flux2-kustomize-helm-example

<a href="https://opensource.newrelic.com/oss-category/#new-relic-example"><picture><source media="(prefers-color-scheme: dark)" srcset="https://github.com/newrelic/opensource-website/blob/main/src/images/categories/dark/Example_Code.png"><source media="(prefers-color-scheme: light)" srcset="[https://github.com/newrelic/opensource-website/raw/main/src/images/categories/Example.png](https://github.com/newrelic/opensource-website/blob/main/src/images/categories/dark/Example_Code.png)"><img alt="New Relic Open Source example code project banner." src="[https://github.com/newrelic/opensource-website/raw/main/src/images/categories/Example.png](https://github.com/newrelic/opensource-website/blob/main/src/images/categories/dark/Example_Code.png)"></picture></a>



For this example we assume a scenario with two clusters: staging and production.
The end goal is to leverage Flux and Kustomize to manage both clusters while minimizing duplicated declarations.

We will configure Flux to install, test and upgrade and patch the opentelemtry-kube-stack operator using
`HelmRepository` and `HelmRelease` custom resources.
Flux will monitor the Helm repository, and it will automatically
upgrade the Helm releases to their latest chart version based on semver ranges.

## Prerequisites

You will need a Kubernetes cluster version 1.28 or newer.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on macOS or Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **infrastructure** dir contains common infra and mointoring tools such as `opentelemetry-kube-stack` and `cert-manager`
- **clusters** dir contains the Flux configuration per cluster

```
├── clusters
│   └── staging
└── infrastructure
    ├── config
    └── controllers
```

### Infrastructure

The infrastructure is structured into:

- **infrastructure/controllers/base** dir contains namespaces and Helm release definitions for Kubernetes controllers
- **infrastructure/controllers/clusters** dir contains cluster specific Kustomize Overlays
- **infrastructure/configs/** dir would contains Kubernetes custom resources such as servicemonitor policies

```
./infrastructure/
├── config
└── controllers
    ├── base
    │   ├── cert-manager
    │   │   ├── helmrelease.yaml
    │   │   ├── helmrepository.yaml
    │   │   ├── kustomization.yaml
    │   │   └── namespace.yaml
    │   └── monitoring
    │       ├── kustomization.yaml
    │       └── opentelemetry-kube-stack
    │           ├── helmrelease.yaml
    │           ├── helmrepository.yaml
    │           ├── kustomization.yaml
    │           └── namespace.yaml
    └── clusters
        └── staging
            ├── kustomization.yaml
            └── monitoring
                ├── kustomization.yaml
                ├── secret.enc.yaml
                └── opentelemetry-kube-stack
                    └── kustomization.yaml
```

In **infrastructure/controllers/base/** dir we have the Flux `HelmRepository` and `HelmRelease` definitions such as:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: opentelemetry-stack
  namespace: opentelemetry-operator-system
spec:
  chart:
    spec:
      chart: opentelemetry-kube-stack
      sourceRef:
        kind: HelmRepository
        name: open-telemetry
      version: "0.4.1"
  interval: 15m
  dependsOn:
    - name: cert-manager
      namespace: cert-manager
  releaseName: opentelemetry-kube-stack
  values:
    clusterName: cluster_name
    opentelemetry-operator:
      admissionWebhooks:
        certManager:
          enabled: true
        autoGenerateCert:
          enabled: false
    collectors:
      daemon:
        enabled: true
        env:
          - name: NEW_RELIC_LICENSE_KEY
            valueFrom:
              secretKeyRef:
                name: newrelic-key-secret
                key: newrelic-license-key
        scrape_configs_file: ""
        targetAllocator:
          enabled: true
          image: ghcr.io/open-telemetry/opentelemetry-operator/target-allocator:main
          allocationStrategy: per-node
          prometheusCR:
            enabled: true
            podMonitorSelector: {}
            scrapeInterval: "10s"
            serviceMonitorSelector: {}
        presets:
          logsCollection:
            enabled: false
          kubeletMetrics:
            enabled: false
          hostMetrics:
            enabled: false
          kubernetesAttributes:
            enabled: false
        config:
          receivers:
            prometheus/podmonitor:
              config:
                scrape_configs:
                  - job_name: kubernetes-pods
                    scrape_interval: 30s
                    kubernetes_sd_configs:
                      - role: pod
                      ...
```

The `dependsOn` directive ensures `opentelemetry-stack` is only deployed after `cert-manager`. The chart is pinned to version `0.4.1` for consistency across bootstraps and deployments.

Check Interval: Set with `interval: 15m`, Flux periodically verifies that the deployed state matches the defined configuration.

The **infrastructure/controllers/cluster/staging/kustomization.yaml** serves as the main config for reference paths to infrastructure resources in the staging environment.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/cert-manager
  - monitoring
```


Located in `monitoring/opentelemetry-kube-stack/kustomization.yaml`, patching allows for dynamic, environment-specific adjustments:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/monitoring/opentelemetry-kube-stack
patches:
  # cluster_name patch
  - patch: |
      - op: replace
        path: /spec/values/clusterName
        value: test_cluster
    target:
      kind: HelmRelease
      name: opentelemetry-stack
      namespace: opentelemetry-operator-system
  # metrics to keep
  - patch: |
      - op: replace
        path: /spec/values/collectors/daemon/config/receivers/prometheus~1podmonitor/config/scrape_configs/0/metric_relabel_configs
        value: 
          - action: replace
            target_label: job_label
            replacement: kubernetes-apps
          - source_labels: [ __name__ ]
            regex: ^(istio.*|awscni_eni_max|awscni_build_info|envoy_cluster_bind_errors)$
            action: keep
    target:
      kind: HelmRelease
      name: opentelemetry-stack
      namespace: opentelemetry-operator-system
```

Cluster Name Patch: Changes `/spec/values/clusterName` to `test_cluster`.

Metrics Configuration Patch: Adjusts the `prometheus/podmonitor` configuration to retain specific metrics, as defined by custom labels and selections regex.


The patching mechanism leverages Kustomize overlays, enabling teams to maintain base configurations in one set of files and environment-specific tweaks in overlay files. This approach is efficient for creating multi-tenant environment configurations (e.g., staging, production) without duplicating the entire configuration set and expose complex configs.

## Bootstrap staging and production



The clusters dir contains the Flux configuration:

```
./clusters/
└── staging
    └── infrastructure.yaml
```

In **clusters/staging/** dir we have the Flux Kustomization definitions, where we instruct flux to utilize the stagging path as the main entry point for the staging environment's infrastructure configuration. 

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infra-controllers
  namespace: flux-system
spec:
  interval: 15m
  retryInterval: 5m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/controllers/clusters/staging
  prune: true
  wait: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```sh
flux check --pre

► checking prerequisites
✔ Kubernetes 1.32.0 >=1.30.0-0
✔ prerequisites checks passed
```

Set the kubectl context to your staging cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=staging \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/staging
```

The bootstrap command commits the manifests for the Flux components in `clusters/staging/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being installed on staging:

```console
$ watch flux get helmreleases --all-namespaces

NAMESPACE    	NAME         	REVISION	SUSPENDED	READY	MESSAGE 
cert-manager 	cert-manager 	v.17.1   	False    	True 	Release reconciliation succeeded
ingress-nginx	ingress-nginx	0.4.1   	False    	True 	Release reconciliation succeeded
```

Watch the stagging reconciliation:

```console
$ flux get kustomizations --watch

NAME                REVISION                SUSPENDED       READY   MESSAGE                              
flux-system         main@sha1:c99824a3      False           True    Applied revision: main@sha1:c99824a3
infra-controllers   main@sha1:c99824a3      False           True    Applied revision: main@sha1:c99824a3
```

## Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```sh
mkdir -p clusters/dev
```

Copy the sync manifests from staging:

```sh
cp clusters/staging/infrastructure.yaml clusters/dev
cp clusters/staging/apps.yaml clusters/dev
```

You could create a dev overlay inside `apps`, make sure
to change the `spec.path` inside `clusters/dev/apps.yaml` to `path: ./apps/dev`. 

Push the changes to the main branch:

```sh
git add -A && git commit -m "add dev cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev
```

## Identical environments

If you want to spin up an identical environment, you can bootstrap a cluster
e.g. `production-clone` and reuse the `production` definitions.

Bootstrap the `production-clone` cluster:

```sh
flux bootstrap github \
    --context=production-clone \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production-clone
```

Pull the changes locally:

```sh
git pull origin main
```

Create a `kustomization.yaml` inside the `clusters/production-clone` dir:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - ../production/infrastructure.yaml
```

Push the changes to the main branch:

```sh
git add -A && git commit -m "add production clone" && git push
```

Tell Flux to deploy the production workloads on the `production-clone` cluster:

```sh
flux reconcile kustomization flux-system \
    --context=production-clone \
    --with-source 
```

## Testing

Any change to the Kubernetes manifests or to the repository structure should be validated in CI before
a pull requests is merged into the main branch and synced on the cluster.
