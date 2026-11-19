# CNE Client Extensions: Next Steps

What comes after the first CNE client extension release, as found while testing it on AWS. [`cne-cx-feature-demo.md`](cne-cx-feature-demo.md) sets up the environment these notes refer to.

## Private Registries

**Status:** works, tested. A client extension published to private GitHub Container Registry packages deployed, and both pulls failed without credentials.

A private registry is used twice, and each pull gets its credentials in a different place:

| Pull | Who pulls | Credentials |
| --- | --- | --- |
| Descriptor | Argo CD's repo server (the `cxdescriptor` Helm plugin) | The optional `argocd-cxdescriptor-registry-credentials` Secret in `argocd-system`, type `kubernetes.io/dockerconfigjson`. The repo server reads it when it starts. |
| Workload image | The kubelet | A pull Secret in the client extension's namespace, named in the entry's `values.imagePullSecrets` in `liferay.yaml` |

For example:

```bash
kubectl create secret docker-registry argocd-cxdescriptor-registry-credentials \
	--docker-password="${TOKEN}" \
	--docker-server=ghcr.io \
	--docker-username=<user> \
	--namespace argocd-system

kubectl rollout restart deployment argocd-repo-server --namespace argocd-system

kubectl create secret docker-registry ghcr-pull \
	--docker-password="${TOKEN}" \
	--docker-server=ghcr.io \
	--docker-username=<user> \
	--namespace liferay-<project>-<environment>
```

```yaml
clientExtensions:
  liferay-sample-etc-node:
    values:
      imagePullSecrets:
        - name: ghcr-pull
```

Use a token that can only read packages.

**Next:** there is no gitops-managed way to create either Secret. Each needs an ExternalSecret, for example from AWS Secrets Manager: one in `argocd-system` from the platform chart, and one in each client extension namespace.

## Client Extensions In A Separate Namespace

**Status:** works for client extensions without a route; on the release branch, not yet in master.

An environment's `liferay.yaml` can set `clientExtensionsSeparateNamespace: true`, or one entry can set `separateNamespace: true` (an entry's value wins). The ApplicationSet then:

- deploys into `liferay-<project>-cx-<environment>`, a name derived from the environment's path, so a `liferay.yaml` cannot place client extensions in another environment's namespace;
- creates that namespace with the `pod-security.kubernetes.io/enforce: restricted` label;
- points the chart at DXP's namespace through `clientExtension.dxpNamespace`, which makes the chart add a RoleBinding for the DXP operator.

DXP must still grant the namespace, in the same `liferay.yaml`:

```yaml
liferay-default:
  clientExtensions:
    policy:
      allowedNamespaces:
        - liferay-<project>-cx-<environment>
```

Until it does, the ClientExtension reports `NamespaceNotPermitted`, with a message that says what to add. Revoking a namespace later stops delivery but leaves what was already delivered, and the workload keeps running.

Tested by hand on `etc-cron`: the operator delivered its configuration to DXP's namespace and mirrored DXP's metadata ConfigMaps into the client extension namespace, and a run from there got its OAuth2 token and called DXP and the Spring Boot backend.

**Next:**

1. **Routed client extensions.** The environment's Gateway accepts routes only from its own namespace (`allowedRoutes.namespaces.from: Same`), and the chart's HTTPRoute has no `parentRef` namespace. This needs a namespace selector on the Gateway listeners in the infrastructure composition, such as a `liferay.com/gateway: <project>-<environment>` label that the ApplicationSet puts on the namespace it creates, and a `network.gatewayNamespace` value in the chart that the ApplicationSet sets to DXP's namespace.
1. **Per-team namespaces,** if they are needed: the same derived pattern with a suffix, such as `liferay-<project>-cx-<environment>-<team>`, never a namespace named in full.
1. **Moving a client extension recreates its OAuth2 application with a new client ID.** A backend that resolved client IDs at startup, such as the Spring Boot sample's `liferay.oauth.application.external.reference.codes`, rejects the moved client extension's tokens with `401` until it restarts. Document this, or make the sample's client library resolve the IDs again.

## Backends On The Cluster Network Only

**Status:** not tested; notes from reading the charts and the operator.

**Goal:** a backend that only DXP calls, such as one with object or workflow actions and no browser traffic, gets no public route or DNS record, and DXP calls it on its cluster address, `http://<service>.<namespace>.svc.cluster.local:<port>`.

**What exists:**

- The chart renders a Service for a client extension that runs as a Deployment, and its HTTPRoute only when `network.enabled` is true.
- DXP builds a client extension's address from `.serviceScheme` and `.serviceAddress` in its configurations, which Workspace takes from `client-extension.yaml`, such as `http` and `localhost:58081`.
- The operator passes `clientExtension.domain` to DXP as the client extension's main domain, and DXP calls the object actions through it: the JWT audience is that domain. How DXP combines the main domain with `.serviceAddress` has not been checked.
- The chart merges `clientExtension.configs` from `values` over the descriptor's `configs`.

**What to try:** leave out `network` and `clientExtension.domain`, and override `.serviceScheme` and `.serviceAddress` through `values.clientExtension.configs` with the Service's cluster address. Then check that:

- the object actions still fire and reach the backend;
- the JWT audience DXP signs, now the cluster address, is the one the backend accepts;
- the ClientExtension's status is sensible without a URL;
- DXP's NetworkPolicy lets it reach the client extension's pod.

If overriding each configuration is clumsy, the follow-up is a chart or operator setting, such as `clientExtension.internal: true`, that sets the address from the Service. Backends a browser calls, such as custom-element-2's, still need a public route.
