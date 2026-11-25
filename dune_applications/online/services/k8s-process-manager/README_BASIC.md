# Pocket Baseline and Process Manager

## Deploy and Connect

- From the Pocket root, create the Kind cluster if it does not already exist:

  ```sh
  make create-pocketdune-cluster
  ```

- Add the downloaded tools to `PATH` and deploy the baseline:

  ```sh
  eval "$(make env)"
  cd daq-kube
  kluctl deploy -t pocket-baseline --context kind-pocketdune --write-command-result=false
  ```

- Deploying the baseline does not remove services from an existing full Pocket
  deployment. Use a fresh cluster if those services need to be removed.

- Deploy the process manager separately:

  ```sh
  kluctl deploy -t pocket-k8s-pm --context kind-pocketdune \
    -a k8s_pm_image=ghcr.io/dune-daq/nightly-release-alma9:development_v5 \
    --write-command-result=false
  ```

- Wait for the process manager pod to show `1/1` ready. The image pull and
  nightly setup can take several minutes; stop watching with `Ctrl+C`:

  ```sh
  kubectl --context kind-pocketdune -n drunc-process-manager get pods -w
  ```

- Create a separate shell pod, then enter it once it is ready:

  ```sh
  kubectl --context kind-pocketdune -n drunc-process-manager run drunc-pm-shell \
    --image=ghcr.io/dune-daq/nightly-release-alma9:development_v5 \
    --image-pull-policy=IfNotPresent --restart=Never \
    --command -- /bin/bash -c 'exec tail -f /dev/null'
  kubectl --context kind-pocketdune -n drunc-process-manager \
    wait --for=condition=Ready pod/drunc-pm-shell --timeout=10m
  kubectl --context kind-pocketdune -n drunc-process-manager \
    exec -it drunc-pm-shell -- /bin/bash
  ```

- Inside the shell pod, activate the nightly and connect to the manager Service:

  ```sh
  export SPACK_ROOT=/cvmfs/dunedaq-development.opensciencegrid.org/nightly/last_fddaq/spack-installation
  source /cvmfs/dunedaq.opensciencegrid.org/setup_dunedaq.sh
  setup_dbt latest
  dbt-setup-release -n last_fddaq
  drunc-process-manager-shell grpc://drunc-process-manager:10054
  ```

- At the process-manager prompt, list managed processes:

  ```text
  ps
  ```

- The manager currently has read-only Kubernetes permissions; starting or
  deleting DAQ sessions requires additional permissions and configuration.
- The gRPC endpoint is insecure. Other workloads in the cluster may be able
  to connect to it; restrict access in a shared cluster.

## Useful Commands

- List all pods, or watch the process-manager namespace (`Ctrl+C` stops watching):

  ```sh
  kubectl --context kind-pocketdune get pods -A
  kubectl --context kind-pocketdune -n drunc-process-manager get pods -w
  ```

- Follow process-manager logs, or inspect a previous container after a restart:

  ```sh
  kubectl --context kind-pocketdune -n drunc-process-manager logs -f deploy/drunc-process-manager
  kubectl --context kind-pocketdune -n drunc-process-manager logs deploy/drunc-process-manager --previous
  ```

- Inspect pod status and events, including image-pull failures:

  ```sh
  kubectl --context kind-pocketdune -n drunc-process-manager describe pod <pod-name>
  kubectl --context kind-pocketdune -n drunc-process-manager get events --watch
  ```

- List Services and their ports:

  ```sh
  kubectl --context kind-pocketdune get services -A
  ```

- Restart the manager after changing its ConfigMap:

  ```sh
  kubectl --context kind-pocketdune -n drunc-process-manager rollout restart deployment/drunc-process-manager
  ```

- Exit the process-manager shell, then run `exit` in Bash. Remove the separate
  shell pod when finished:

  ```sh
  kubectl --context kind-pocketdune -n drunc-process-manager delete pod drunc-pm-shell
  ```

- Delete the manager Deployment and its pods without deleting the baseline:

  ```sh
  kubectl --context kind-pocketdune -n drunc-process-manager delete deployment drunc-process-manager --cascade=foreground
  ```

- Remove the Kind cluster from the Pocket root. Pods and data stored only
  inside its nodes are lost; host-mounted data may remain:

  ```sh
  make remove-pocketdune-cluster
  ```
