# RIC-PLT Near-RT RIC New Installer

This is a fork to update the installation process of [RIC-PLT's Near-RT RIC](https://github.com/o-ran-sc/ric-plt-ric-dep/blob/m-release/new-installer/README.md). The installer leverages a single Helm chart for installation. The components deployed during installations are determined by flags present in the `values.yaml` file of the chart, which can be overridden using an override file.

The code is organized as follows:

```
  |
  + helm
  | |
  | + charts
  |   |
  |   + Makefile
  |   + nearrtric/
  |     |
  |     + Makefile
  |     + a1mediator/
  |     + appmgr/
  |     + dbaas/
  |     + e2mgr/
  |     + e2term/
  |     + nearrt-ric-common/
	|		...
  |
  + helm-overrides/
    |
    + nearrtric/
      |
      + minimal-nearrt-ric.yaml
```

All sub-chart dependencies are resolved using Helm's native `file://` repository support.

## Supported Versions

+ `microk8s` with a supported Kubernetes version.
+ Kubernetes 1.22 and above.
+ Helm 4.2 and above.

This setup is tested on Ubuntu 22.04 with the following `microk8s` and `helm` versions.

```bash
$ uname -mvp
#40~22.04.2-Ubuntu SMP PREEMPT_DYNAMIC x86_64 x86_64

$ microk8s version
MicroK8s v1.32.3 revision 7647

$ microk8s kubectl version --client
Client Version: v1.32.3
Server Version: v1.32.3

$ helm version
version.BuildInfo{Version:"v4.2.3", GitCommit:"912ebc1cd10d38d340f048efaf0abda047c3468e", ...}
```

## Getting Started

### Setup and Pre-requisites

These instructions assume that `microk8s` and `helm` are installed. If you are behind an HTTP proxy, set `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` before proceeding so that Kubernetes can pull container images.

### Installing `microk8s`

```bash
sudo snap install microk8s --classic --channel=1.32
sudo usermod -aG microk8s $USER
newgrp microk8s
```

Enable the add-ons required by the NearRT RIC:

```bash
microk8s enable dns ingress storage
```

Verify the cluster is up:

```bash
microk8s kubectl get pods -A
```

For convenience you can alias `kubectl`:

```bash
alias kubectl='microk8s kubectl'
```

### Installing Helm

Download the latest Helm 4 release from [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/) or with `snap`:

```bash
sudo snap install helm --classic
helm version
```

### Creating Platform and xApp Namespaces

Create the namespaces once per new cluster (they persist across restarts):

```bash
microk8s kubectl create ns ricplt
microk8s kubectl create ns ricxapp
```

## Running NearRT RIC

### Preparing the Charts

Build all sub-charts and the umbrella chart with a single `make` command. No external chart server is needed — all local dependencies are resolved directly from the source tree via `file://` paths.

```bash
cd ric-plt-ric-dep/new-installer/helm/charts
make nearrtric
```

The packaged umbrella chart is written to:

```
helm/charts/dist/packages/nearrtric-0.1.0.tgz
```

### Installing the Charts

Install using the packaged chart and an override file (noting the provided override file points to Alexandre Huff's updated A1 Mediator for enhanced xApp compatibility):

```bash
helm install nearrtric -n ricplt \
  helm/charts/dist/packages/nearrtric-0.1.0.tgz \
  -f helm-overrides/nearrtric/minimal-nearrt-ric.yaml
```

Refer to `helm-overrides/nearrtric/minimal-nearrt-ric.yaml` for the full set of configurable options including image registries, tags, namespaces, and ingress IPs.

#### Verifying the Installation

```bash
microk8s kubectl get pods -n ricplt
```

Expected output :

```
NAME                                              READY   STATUS    RESTARTS   AGE
deployment-ricplt-a1mediator-84fc865778-x846h     1/1     Running   0          2m
deployment-ricplt-appmgr-57cc4d665b-lb8dg         1/1     Running   0          2m
deployment-ricplt-e2mgr-9748f9585-mg2zl           1/1     Running   0          2m
deployment-ricplt-e2term-alpha-5ffb57bf9f-slmrz   1/1     Running   0          2m
deployment-ricplt-rtmgr-57f7c7797f-mpkg9          1/1     Running   0          2m
deployment-ricplt-submgr-74f67bf444-qh5rn         1/1     Running   0          2m
statefulset-ricplt-dbaas-server-0                 1/1     Running   0          2m
```

```bash
microk8s kubectl get svc -n ricplt
```

### Deploying the E2 Simulator

The simulator is available from the [sim-e2-interface](https://gerrit.o-ran-sc.org/r/admin/repos/sim/e2-interface,general) repository. When building on Ubuntu 22.04, update the `Dockerfile` in `e2sim/e2sm_examples/kpm_e2sm` to use the Ubuntu 22.04 builder image:

```Dockerfile
ARG CONTAINER_PULL_REGISTRY=nexus3.o-ran-sc.org:10004
FROM ${CONTAINER_PULL_REGISTRY}/o-ran-sc/bldr-ubuntu22-c-go:0.1.0 as buildenv
```

Also update the E2 term SCTP service IP in the `Dockerfile` CMD line to match the `CLUSTER-IP` shown by `microk8s kubectl get svc service-ricplt-e2term-sctp-alpha -n ricplt`:

```Dockerfile
CMD kpm_sim <service-ricplt-e2term-sctp-alpha-cluster-ip> 36422
```

With `microk8s`, container images are managed by `containerd`. Build the image with Docker and import it into the microk8s runtime:

```bash
docker build -t e2sim:latest .
docker save e2sim:latest | microk8s ctr images import -
```

Then install the simulator chart:

```bash
helm install e2sim -n ricplt helm/
```

### Deploying xApps

xApps are packaged as Helm charts and installed directly into the `ricxapp` namespace. If you have a pre-packaged chart tarball:

```bash
helm install <release-name> -n ricxapp <chart-name-version.tgz>
```

If you need to use the `xapp_onboarder` tool to generate the chart from an xApp descriptor, the tool requires a Helm-compatible chart repository for storage. The recommended approach with modern Helm is to use an OCI-based registry. The microk8s registry add-on provides a local one:

```bash
microk8s enable registry
# Registry is available at localhost:32000
```

Then point `CHART_REPO_URL` at the registry when running `dms_cli` (note the typo on `--shcema_file_path` flag):

```bash
git clone https://gerrit.o-ran-sc.org/r/ric-plt/appmgr
cd appmgr/xapp_orchestrater/dev/xapp_onboarder
python3 -m venv venv
. venv/bin/activate
pip install -r requirements.txt .

CHART_REPO_URL=http://localhost:32000 dms_cli onboard \
  --config-file-path <path-to-app-config> \
  --shcema_file_path <path-to-schema-json>

CHART_REPO_URL=http://localhost:32000 dms_cli download_helm_chart <chart-name> <version>

helm install <release-name> -n ricxapp <chart-name-version.tgz>
```

You can check the logs of individual pods using `microk8s kubectl logs ...` to verify xApps are running as expected.
