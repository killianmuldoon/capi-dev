# Developing CAPI on MacOS

# Unit test panics

When running unit test debugging configurations, a `panic` sometimes leads to a stuck debugger. Solutions:

* Enable the `Go error breakpoint / Fatal error` breakpoint (in `View Breakpoint`)
* Use run configurations to find and fix the panic

# Run local controllers including webhooks via Telepresence

Patch remote package to get workload cluster kube-apiserver IPs from Docker to be able to communicate with workload clusters. Telepresence must be used to get certificates and send webhook traffic to
local controllers.

Patch: [local-controllers.patch](./patches/local-controllers.patch)

Telepresence:

```bash
# TODO: complete cleanup before: telepresence uninstall --everything + quit
# Import kubeconfig
kind_kubeconfig test-<tab>
# Intercept webhook service (capi, capd, capi-kubeadm-bootstrap, capi-kubeadm-control-plane, capo, capv, capz)
controller=capi
telepresence connect
# Disable manager
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/readinessProbe"},{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"},{"op": "replace", "value": "k8s.gcr.io/pause:3.5", "path": "/spec/template/spec/containers/0/image"},{"op": "replace", "value": ["/pause"], "path": "/spec/template/spec/containers/0/command"}]'
telepresence intercept -n ${controller}-system ${controller}-controller-manager --port 9443 --env-file /tmp/local-controller-env --mount "${HOME}/telepresence/${controller}-controller/volumes"

# Un-intercept webhook service
telepresence uninstall -n ${controller}-system --agent ${controller}-controller-manager
# Enable manager (Note: image must be adjusted)
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "replace", "value": "gcr.io/k8s-staging-cluster-api/cluster-api-controller:master", "path": "/spec/template/spec/containers/0/image"},{"op": "replace", "value": ["/manager"], "path": "/spec/template/spec/containers/0/command"}]'
# Remove telepresence traffic-manager
telepresence uninstall --everything
```

Run controllers with: `--webhook-cert-dir=${HOME}/telepresence/${controller}-controller/volumes/tmp/k8s-webhook-server/serving-certs/`

# Run local controllers without webhooks

The patched code generates webhook certs if missing (so the controller is able to start). Also patches remote package to get a workload cluster kube-apiserver IP from Docker to be able to communicate
with workload clusters. Controllers in cluster have to keep running to serve webhooks (Note: this means the controllers are running twice, which can lead to unintended side effects).

Patch: [local-controllers-self-gen-certs.patch](./patches/local-controllers-self-gen-certs.patch)

```bash
kind get kubeconfig --name=$(kind get clusters | grep test) | k8s-ctx-import; kctx kind-$(kind get clusters | grep test)
```

Run controllers with: `--leader-elect=false`

## Retry e2e test apply

Sometimes webhooks are not available in time. This patch retries apply until the webhooks are ready.

Patch: [e2e-test-retry.patch](./patches/e2e-test-retry.patch)

Patch: [0.3-e2e-test-retry.patch](./patches/0.3-e2e-test-retry.patch) (for release-0.3 branch)

Note: I didn't had this issue the last view weeks, so it maybe has been fixed somehow (I had it very frequently before).

## Prepare e2e test

Execute the following to build e2e images and generate templates before running e2e tests via the IDE:

```bash
export REGISTRY=gcr.io/k8s-staging-cluster-api
export TAG=dev
export ARCH=amd64
export PULL_POLICY=IfNotPresent

make docker-build
make -C test/infrastructure/docker docker-build

make -C test/e2e cluster-templates
```

For upgrade tests:

```bash
source $CAPI_HOME/scripts/ci-e2e-lib.sh
# e.g.:
k8s::resolveVersion KUBERNETES_VERSION ci/latest-1.22
kind::prepareKindestImage "$resolveVersion"
```
