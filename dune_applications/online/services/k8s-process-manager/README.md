# Kubernetes Process Manager

This is an experimental, opt-in Kluctl target for running the drunc Kubernetes
process manager as a pod. It is not included in the normal `pocket` target.

## Changes

- Added the `pocket-k8s-pm` target and `k8s_pm_image` argument to `.kluctl.yaml`.
- Updated the root `deployment.yml` to select this target independently from
  the rest of Pocket.
- Added a namespace, a single-replica Deployment, and a ClusterIP gRPC Service
  on port 10054.
- Added a ConfigMap with a minimal Pocket process-manager configuration and a
  kubeconfig. The kubeconfig obtains the pod's projected service-account token
  through a Python exec credential plugin, so drunc's existing
  `load_kube_config()` can be used without changes to drunc source. The plugin
  advertises a short credential expiry so the client can reread rotated tokens.
- Added a service account with cluster-wide read-only access to namespaces and
  pods, needed for startup and the manager's pod watcher. It cannot create or
  delete sessions or pods. Additional RBAC and configuration are required for
  that later milestone.
- No changes to drunc source were retained. An initial authentication change
  was reverted in favour of the manifest approach.

## Runtime Image

Supply an image containing `drunc-process-manager`, `python3`, and all drunc
and DAQ dependencies. It must allow the manager to run with `/tmp` as its
working and log directory.

The CI image `ghcr.io/dune-daq/nightly-release-alma9:development_v5` contains
drunc in its nightly environment. It is AMD64-only, but a disposable Docker
container ran successfully under emulation on this ARM64 Mac. The executable
is not on the default `PATH`. The following activation succeeded in that
container, including running `drunc-process-manager --help`:

```sh
export SPACK_ROOT=/cvmfs/dunedaq-development.opensciencegrid.org/nightly/last_fddaq/spack-installation
source /cvmfs/dunedaq.opensciencegrid.org/setup_dunedaq.sh
setup_dbt latest
dbt-setup-release -n last_fddaq
```

The Deployment invokes these activation steps before starting the process
manager. The nightly image occupies approximately 13.2 GB in Docker. A Kind
node may also pull and store its own copy. The Docker image cache can be
removed with:

```sh
docker image rm ghcr.io/dune-daq/nightly-release-alma9:development_v5
```

CI also creates a separate work area and installs the tested drunc checkout;
the image's nightly may contain a different revision. Pocket's existing
run-registry image does not contain drunc. Pocket's CVMFS integration is
conditional on `with_dune_readout` and was not present in the cluster checked
here.

The included minimal configuration is for starting the manager only. It uses
`/tmp` for `home_path_base` and the default managed-pod image; it is not a DAQ
session configuration and has not been validated for spawning workloads.

## Deploy And Connect

These steps assume the `pocketdune` Kind cluster already exists. From the
Pocket repository root, add the downloaded tools to your shell's `PATH`, set
the hostname used by Kafka, and deploy Pocket:

```sh
eval "$(make env)"
export HOSTNAME="$(hostname)"
cd daq-kube
kluctl deploy -t pocket --context kind-pocketdune --write-command-result=false
```

The hostname advertised by Kafka must be resolvable by its external clients.
The `--write-command-result=false` option avoids the Kubernetes Secret size
limit encountered when Kluctl saves a large deployment result, but disables
the saved result record.

Before deploying the process manager, check that the `process-manager.json`
inside `manager/process-manager.yaml` is valid JSON and includes the OpMon
settings. Its final fields should be:

```json
"opmon_uri": {"path": "/tmp/drunc-opmon.json", "type": "file"},
"opmon_conf": {"level": "info", "interval_s": 10.0},
"settings": {"home_path_base": "/tmp"}
```

There must be only one `settings` entry. The OpMon data above is written to a
file in the container; this does not configure a Kafka OpMon stream.

From `daq-kube`, deploy the separate manager target:

```sh
kluctl deploy -t pocket-k8s-pm --context kind-pocketdune \
  -a k8s_pm_image=ghcr.io/dune-daq/nightly-release-alma9:development_v5 \
  --write-command-result=false
```

For an existing deployment, a ConfigMap change alone does not necessarily
restart the container. After updating its configuration, restart it:

```sh
kubectl --context kind-pocketdune -n drunc-process-manager \
  rollout restart deployment/drunc-process-manager
```

Watch the pod and inspect its logs:

```sh
kubectl --context kind-pocketdune -n drunc-process-manager get pods -w
kubectl --context kind-pocketdune -n drunc-process-manager \
  logs -f deploy/drunc-process-manager
```

Use separate terminals for the watch and logs, and press `Ctrl+C` to stop
each. The initial image pull is large, and nightly setup can take several
minutes. Wait for the pod to show `1/1` ready and the manager to report its
gRPC address. If it is not ready, inspect its events and previous logs:

```sh
kubectl --context kind-pocketdune -n drunc-process-manager describe pod <pod-name>
kubectl --context kind-pocketdune -n drunc-process-manager \
  logs deploy/drunc-process-manager --previous
```

The previous-log command requires a prior container instance; its logs may no
longer be available. If startup reports `Missing opmon configuration`, check
the embedded JSON above and redeploy the target.

Once the manager is ready, open Bash in its pod:

```sh
kubectl --context kind-pocketdune -n drunc-process-manager exec -it \
  deploy/drunc-process-manager -- /bin/bash
```

Inside the pod, activate the nightly environment and start the shell:

```sh
export SPACK_ROOT=/cvmfs/dunedaq-development.opensciencegrid.org/nightly/last_fddaq/spack-installation
source /cvmfs/dunedaq.opensciencegrid.org/setup_dunedaq.sh
setup_dbt latest
dbt-setup-release -n last_fddaq
drunc-process-manager-shell grpc://127.0.0.1:10054
```

Try `ps` in the process-manager shell. `127.0.0.1` refers to this pod because
the shell is running in the manager's container. Exit the process-manager
shell, then use `exit` to leave Bash. In a separate pod, connect using the
`drunc-process-manager` Service name instead of `127.0.0.1`.

The gRPC service is insecure and the manager uses a dummy authoriser. The
ClusterIP service is not externally exposed, but other cluster workloads may
be able to reach it. Restrict access before using it in a shared cluster.

## Verification And Limits

- The target rendered successfully with a placeholder image; the rendered
  Kubernetes resources passed `kubectl create --dry-run=client`.
- The rendered kubeconfig was parsed by `kubectl config view`, and the existing
  `pocket-baseline` target still rendered successfully.
- A server-side Kluctl dry run without an image rejected the Deployment as
  invalid, as intended. It did not deploy the target.
- The nightly image's process-manager `--help` ran after activating the release
  with an explicit `SPACK_ROOT`. In the first pod attempt, the manager reported
  a missing OpMon configuration; the correction and shell connection have not
  yet been verified. Exec credential refresh also remains unverified.
- This target does not create, delete, or test a DAQ session. It must not be
  treated as a complete process-manager deployment.


