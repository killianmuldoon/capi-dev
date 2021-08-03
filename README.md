
# MacOS

When running unit tests debugging configurations, `panic`s sometimes lead to a stuck debugger. Workarounds:
* Enable the `Go error breakpoint / Fatal error` breakpoint (in `View Breakpoint`)
* Use run configurations to find the panic

# main

## Patch: local controllers for Telepresence

Patches remote package to get kube-apiserver IP from Docker to be able to communicate with workload clusters. Telepresence must be used to get certificates and send webhook traffic to local controllers.

Patch: `local-controllers.patch`

Telepresence:

```bash
# Import kubeconfig
kind get kubeconfig --name=$(kind get clusters | grep test) | k8s-ctx-import; kctx kind-$(kind get clusters | grep test)
# Intercept webhook service (capi, capd, capi-kubeadm-bootstrap, capi-kubeadm-control-plane, capo, capv, capz)
controller=capi
telepresence intercept -n ${controller}-system ${controller}-controller-manager --port 9443 --env-file /tmp/local-controller-env --mount "${HOME}/telepresence/${controller}-controller/volumes"
# Disable manager
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/readinessProbe"}]'
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "replace", "value": "k8s.gcr.io/pause:3.5", "path": "/spec/template/spec/containers/0/image"}]'
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "replace", "value": ["/pause"], "path": "/spec/template/spec/containers/0/command"}]'

# Un-intercept webhook service
telepresence uninstall -n ${controller}-system --agent ${controller}-controller-manager
# Enable manager (Note: image must be adjusted)
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "replace", "value": "gcr.io/k8s-staging-cluster-api/cluster-api-controller:master", "path": "/spec/template/spec/containers/0/image"}]'
k -n ${controller}-system patch deployment ${controller}-controller-manager --type json -p='[{"op": "replace", "value": ["/manager"], "path": "/spec/template/spec/containers/0/command"}]'
# Remove telepresence traffic-manager
telepresence uninstall --everything
```

## Patch: local controllers (including self gen certificates)

Generate webhook certs if missing. Patches remote package to get kube-apiserver IP from Docker to be able to communicate with workload clusters. Controllers in cluster have to keep running to serve webhooks.

Patch: `local-controllers-self-gen-certs.patch`

```bash
kind get kubeconfig --name=$(kind get clusters | grep test) | k8s-ctx-import; kctx kind-$(kind get clusters | grep test)
```

## Patch: envtest with kind

Patches `WebhookInstallOptions` so the kube-apiserver running in Docker for Mac can reach the webhooks running locally.

Patch: `envtest-kind.patch`

## Patch: retry e2e test apply

Webhooks are not available in time. This patch just retries apply until the webhooks are ready.

Patch: `e2e-test-retry.patch`

## Prepare e2e

Build images and generate templates.

```bash
make docker-build
make -C test/infrastructure/docker docker-build

make -C test/e2e cluster-templates
```


# v0.3

## Patch: retry e2e test apply

Webhooks are not available in time. This patch just retries apply until the webhooks are ready.

Patch: `0.3-e2e-test-retry.patch`

## Prepare e2e

Build images and generate templates.

```bash
make docker-build
make -C test/infrastructure/docker docker-build

make -C test/e2e cluster-templates
```
