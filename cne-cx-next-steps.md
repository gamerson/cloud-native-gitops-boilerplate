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

**Status:** in review as cloudnative-team/liferay-portal#606 (LCD-53966), and on the `release/LCD-53904` release candidate.

Two settings turn it on for an environment, one in each of its gitops files. `infrastructure.yaml` creates the namespace:

```yaml
clientExtensions:
  separateNamespace:
    enabled: true
```

`liferay.yaml` chooses which client extensions use it. `clientExtensionDefaults` merges under every entry of `clientExtensions`, and a key an entry sets replaces the default's, so one entry can stay in DXP's namespace:

```yaml
clientExtensionDefaults:
  separateNamespace: true

clientExtensions:
  liferay-sample-global-css-1:
    enabled: true
    separateNamespace: false
```

The platform then:

- names the namespace `liferay-<project>-<environment>-cx`, derived from the environment's path, so a `liferay.yaml` cannot name another namespace;
- creates it from the infrastructure chart with the `pod-security.kubernetes.io/enforce: restricted` and `liferay.com/observable: "true"` labels, so the observability stack still collects from it, and removes it with the environment;
- lets DXP's `client-extension-policy` accept it, with no grant in `liferay.yaml`;
- binds the DXP operator in it, with the RoleBinding's subject pinned to the platform's operator;
- lets DXP's Gateway accept routes from it, so client extensions that serve HTTP keep their domains.

An environment ID of `cx`, or one that ends with `-cx`, is refused, since its namespace could be another environment's client extension namespace.

Setting `clientExtensions.separateNamespace.enabled` in the base `infrastructure.yaml` turns it on for every environment, and an environment can turn it off with `false`. `clientExtensionDefaults` belongs in each environment's `liferay.yaml` instead: the base and environment `liferay.yaml` are merged in a way that treats `false` as unset, so an environment cannot turn off a `true` from the base.

**Lifecycle:**

- Turning the setting on for a running client extension deletes it from DXP's namespace and creates it in the separate one. Expect a short outage, and a new OAuth2 client ID for its applications: a backend that resolved client IDs at startup, such as the Spring Boot sample's `liferay.oauth.application.external.reference.codes`, rejects the moved client extension's tokens with `401` until it restarts.
- A private registry's pull Secret has to exist in the separate namespace too. `clientExtensionDefaults.values.imagePullSecrets` names it for every client extension.

**Next:**

1. **Per-team namespaces,** if they are needed: the same derived pattern with another suffix, never a namespace named in full.

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
