# CNE Client Extension Feature Demo

Step-by-step instructions to stand up a Liferay Cloud Native (CNE) environment on AWS that runs one nonproduction DXP pod with six client extensions, and to demonstrate them working end to end:

| Client extension | Descriptor type | Workload | What it shows |
| --- | --- | --- | --- |
| `liferay-sample-batch` | `batch` | Job | Imports the `Sample` object definition, with four object actions, into DXP |
| `liferay-sample-custom-element-2` | `frontend` | Deployment | A widget served from its own domain that calls both backends |
| `liferay-sample-etc-cron` | `service` | CronJob | A scheduled job that calls DXP's headless API every 15 minutes |
| `liferay-sample-etc-node` | `service` | Deployment | Node backend: a REST endpoint and two object actions |
| `liferay-sample-etc-spring-boot` | `service` | Deployment | Spring Boot backend: a REST endpoint and two object actions |
| `liferay-sample-global-css-1` | `frontend` | Deployment | A global stylesheet served from its own domain |

The flow: Workspace publishes each client extension as a workload image plus a small descriptor image. The gitops repository's `liferay.yaml` lists the descriptors and versions. Argo CD turns each entry into an Application, the `liferay-client-extension` chart deploys it, and the DXP operator delivers its configuration into DXP.

The platform comes from the `release/LCD-53904` release candidate on `cloudnative-team/liferay-portal`.

## Variables

| Variable | Meaning | Example |
| --- | --- | --- |
| `${RELEASE_BRANCH}` | Release branch on `cloudnative-team/liferay-portal` | `release/LCD-53904` |
| `${RC_SHA7}` | First 7 characters of the release branch commit CI built | `c526abe` |
| `${BOOTSTRAP_VERSION}` | `liferay-aws-bootstrap` in `cloud/scripts/versions.json` on that commit | `4.0.1` |
| `${ACCOUNT_ID}` | AWS account | (yours) |
| `${REGION}` | AWS region | `ca-central-1` |
| `${DEPLOYMENT}` | `deployment_name` in `config.json` | `cne-cx-demo` |
| `${DOMAIN}` | Domain you control, with a wildcard TLS certificate | `example.com` |
| `${CERT_SLUG}` | The domain with the dots removed, used in secret names | `examplecom` |
| `${GITOPS_REPO}` | Your copy of this gitops repository | `https://github.com/<your-user>/cne-cx-demo-gitops` |
| `${REGISTRY}` | Registry path you push the client extensions to | `ghcr.io/<your-user>/cne-cx-demo` |
| `${NS}` | Environment namespace | `liferay-able-nonproduction` |

The project is `able` and the environment is `nonproduction` throughout, as in this repository. The files in this repository use `runray.xyz`, `runrayxyz`, and `ghcr.io/gamerson/lcd-53904`; replace them with your `${DOMAIN}`, `${CERT_SLUG}`, and `${REGISTRY}`.

## Prerequisites

- **Tools:** `aws`, `gcloud`, `gh`, `git`, `helm`, `jq`, `kubectl`, `terraform`, `docker`, and JDK 21.
- **AWS:** an account where you have administrator access through SSO.
- **GCP:** read access to `gs://liferay-cloud-native-bootstrap-nonprd`, where CI uploads the release candidate bootstrap tarballs.
- **A domain and DNS provider** where you can create CNAME records, and a wildcard TLS certificate for the domain.
- **A container registry** you can push to and that the cluster can pull from anonymously, such as GitHub Container Registry (`ghcr.io`) with public packages.
- **AWS Secrets Manager secrets** in `${REGION}`, which External Secrets reads:
  - `liferay-certificates-${CERT_SLUG}`: JSON with the keys `tls.crt` and `tls.key`. The name must start with `liferay-certificates-` and contain no dots.
  - `liferay-credentials-<name>`: JSON with the keys `git_machine_user_id` and `git_access_token`, the HTTPS login Argo CD uses for `${GITOPS_REPO}`.

## Step 1: Copy The Gitops Repository

Argo CD watches this repository, and you push changes to it, so you need your own copy:

```bash
gh repo create <your-user>/cne-cx-demo-gitops --public --clone

cd cne-cx-demo-gitops

git remote add upstream https://github.com/gamerson/cloud-native-gitops-boilerplate
git pull upstream master
git push --set-upstream origin master
```

It contains:

- `liferay/`: the gitops values that Argo CD reads (Step 3).
- `liferay-workspace/`: a Liferay Workspace with the six sample client extensions. Argo CD reads only paths under `liferay/`, so this folder never becomes an Application.
- `.github/workflows/publish-client-extensions.yaml`: a GitHub Actions workflow that publishes the client extensions (Step 2).
- `cne-cx-feature-demo.md`: this document.

The Workspace plugin that publishes a client extension as an image plus a descriptor is not released yet. A snapshot of it, `com.liferay.gradle.plugins.workspace` `17.2.0-SNAPSHOT`, is checked in under `liferay-workspace/maven-repository/`, and `liferay-workspace/settings.gradle` loads it from there, so nothing needs to be built first.

## Step 2: Publish The Client Extensions

Each client extension's version is in its own `liferay-workspace/client-extensions/<name>/gradle.properties`, such as `version=1.0.2`. Every client extension needs one; without it, publishing fails with `Set the "liferay.workspace.client.extension.registry" property and a version to publish "<name>"`.

Each publish pushes two images per client extension, each under two tags:

- `${REGISTRY}/<name>:<version>` and `${REGISTRY}/<name>:<version>.<12 hex>`: the workload image.
- `${REGISTRY}/<name>-descriptor:<version>` and `${REGISTRY}/<name>-descriptor:<version>.<12 hex>`: the descriptor, a one-file image holding `client-extension-descriptor.json` (Step 11).

The `<version>.<12 hex>` tag, the version plus the start of the descriptor's digest, pins one exact build.

### From GitHub Actions

1. Set the registry in `liferay-workspace/gradle.properties` to `ghcr.io/<owner of your gitops repository>/<path>`. The workflow can push only to packages under the account that owns the repository.

	```properties
	liferay.workspace.client.extension.registry=${REGISTRY}
	```

1. Commit and push. Every push to `master` that changes `liferay-workspace/` runs **Publish Client Extensions**, logged in with the workflow's own `GITHUB_TOKEN`. It publishes every client extension whose version is not in the registry yet and leaves the rest alone. **Run workflow** in the **Actions** tab runs the same check by hand.

1. The run's summary lists each newly published descriptor under its digest tag, a `liferay.yaml` snippet with each one's `descriptor` and pinned `version`, the client extensions it skipped because their version is already published, and a link to each new package's settings page. The snippet is also attached to the run as the `client-extension-versions-<run number>` artifact.

A version that is already published is never published again, because an environment that uses it would silently get different code. To ship a change, raise that client extension's `version` in the same commit.

If a package already exists because it was first published from someone's machine, the workflow cannot push to it until you open that package's settings, find **Manage Actions access**, click **Add Repository**, and give the gitops repository the **Write** role.

### From Your Machine

```bash
echo "${GITHUB_TOKEN}" | docker login ghcr.io --username <your-user> --password-stdin

cd liferay-workspace

./gradlew publishClientExtension
```

Use a personal access token with `write:packages`. This publishes every client extension at its current version, whether or not that version exists already.

### Make The Packages Public

New packages under a personal account are private, and the cluster pulls anonymously. GitHub has no API for a package's visibility, so do it once per package in the browser; later versions keep it:

1. Open `https://github.com/users/<your-user>/packages/container/<path>/settings`, with the `/` in `<path>` written as `%2F`, for example `cne-cx-demo%2Fliferay-sample-batch-descriptor`. For an organization, use `orgs/<org>` instead of `users/<your-user>`.
1. Under **Danger Zone**, click **Change visibility**, choose **Public**, and type the package name to confirm.

There are twelve packages: six workload images and six `-descriptor` images. Check one anonymously:

```bash
docker logout ghcr.io

docker pull "${REGISTRY}/liferay-sample-batch-descriptor:1.0.2"
```

## Step 3: Edit The Gitops Values

### `liferay/projects/able/base/infrastructure.yaml`

The environment's HTTPS endpoint, which the DXP route and the client extension routes attach to, and the search engine version:

```yaml
endpoints:
  runray.xyz:
    hostname: "*.runray.xyz"
    port: 443
    protocol: HTTPS
    tls:
      externalSecretKeys:
        - "liferay-certificates-runrayxyz"

forceDestroy: true

search:
  opensearch:
    version: "2.19"
```

- The endpoint key (`runray.xyz`) is the name every `endpointRef` below points at.
- `forceDestroy: true` lets Crossplane delete the database and buckets on teardown.
- DXP 2026.q3 needs OpenSearch 2.19 or later. Keep the quotes; the value must be a string.

### `liferay/projects/able/environments/nonproduction/infrastructure.yaml`

```yaml
enabled: true
```

`uat` and `production` stay at `enabled: false`.

### `liferay/projects/able/base/liferay.yaml`

Every client extension, keyed by name, at the version in its `gradle.properties`, and turned off. Each environment turns on the ones it runs:

```yaml
clientExtensions:
  liferay-sample-batch:
    descriptor: oci://${REGISTRY}/liferay-sample-batch-descriptor
    enabled: false
    version: 1.0.2
  liferay-sample-custom-element-2:
    descriptor: oci://${REGISTRY}/liferay-sample-custom-element-2-descriptor
    enabled: false
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
    version: 1.0.2
  liferay-sample-etc-cron:
    descriptor: oci://${REGISTRY}/liferay-sample-etc-cron-descriptor
    enabled: false
    values:
      workload:
        kind: CronJob
        schedule: "*/15 * * * *"
    version: 1.0.2
  liferay-sample-etc-node:
    descriptor: oci://${REGISTRY}/liferay-sample-etc-node-descriptor
    enabled: false
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
      workload:
        containerPort: 3001
    version: 1.0.2
  liferay-sample-etc-spring-boot:
    descriptor: oci://${REGISTRY}/liferay-sample-etc-spring-boot-descriptor
    enabled: false
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
      workload:
        containerPort: 58081
    version: 1.0.5
  liferay-sample-global-css-1:
    descriptor: oci://${REGISTRY}/liferay-sample-global-css-1-descriptor
    enabled: false
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
    version: 1.0.0

liferay-default:
  clientExtensions:
    delivery:
      enabled: true
  image:
    repository: liferay/dxp
    tag: 2026.q3.5
```

- `enabled: false` keeps a client extension out of every environment that does not turn it on.
- `descriptor` and `version` pick the build. `version` is a plain version such as `1.0.2`, or a digest tag such as `1.0.2.8c080d14bb0e` from the publish summary to pin one exact build.
- `values` is passed to the `liferay-client-extension` chart and wins over what the descriptor says.
- `network` gives the four client extensions that serve HTTP a route on the environment's gateway.
- The descriptor does not carry a project's `LCP.json`, so its workload settings go here:
  - `containerPort`: the port a backend listens on. Without it the chart uses port 80, and the route answers `503`.
  - `kind: CronJob` and `schedule` for the cron job. Without them it runs as a Deployment. The CronJob's `concurrencyPolicy` is always `Forbid`.
- `liferay-default.clientExtensions.delivery.enabled: true` lets the DXP operator deliver each client extension's configuration into this environment's DXP. It is `false` by default, and without it no client extension appears in DXP.

The rest of `liferay-default` in this repository's file, such as licensing and marketplace, is not needed for client extensions. Keep what your environment needs, including your usual license setup.

### `liferay/projects/able/environments/nonproduction/liferay.yaml`

The client extensions nonproduction runs, each HTTP one's public hostname, one DXP pod, and what DXP needs behind the gateway:

```yaml
clientExtensions:
  liferay-sample-batch:
    enabled: true
  liferay-sample-custom-element-2:
    enabled: true
    values:
      clientExtension:
        domain: custom-element-2-able-nonproduction.runray.xyz
  liferay-sample-etc-cron:
    enabled: true
  liferay-sample-etc-node:
    enabled: true
    values:
      clientExtension:
        domain: etc-node-able-nonproduction.runray.xyz
  liferay-sample-etc-spring-boot:
    enabled: true
    values:
      clientExtension:
        domain: etc-spring-boot-able-nonproduction.runray.xyz
  liferay-sample-global-css-1:
    enabled: true
    values:
      clientExtension:
        domain: global-css-1-able-nonproduction.runray.xyz

liferay-default:
  customEnv:
    x-web-server-protocol:
      - name: LIFERAY_WEB_PERIOD_SERVER_PERIOD_PROTOCOL
        value: https
  network:
    enabled: true
    endpointRef: runray.xyz
    forceHttpsRedirect: true
    hostnames:
      - "liferay-able-nonproduction.runray.xyz"
  replicaCount: 1
  resources:
    limits:
      memory: 12Gi
    requests:
      memory: 8Gi
  startupProbe:
    failureThreshold: 240
```

- `enabled: true` turns on a client extension that `base` defines.
- `clientExtension.domain` is the hostname a client extension is served on, and the address DXP uses for it. A client extension that serves HTTP with no domain still runs, but has no URL, so a browser cannot reach it.
- `LIFERAY_WEB_PERIOD_SERVER_PERIOD_PROTOCOL=https`: the gateway redirects HTTP to HTTPS, so DXP must advertise `https` to client extensions, or their calls back to DXP get a `301`.
- `resources`: EKS Auto Mode sizes the node from the request, and the chart's 4Gi default gets a node that cannot fit the 12Gi limit.
- `startupProbe.failureThreshold: 240`: the first boot against an empty database runs past the chart's default 15 minute startup budget.

### `liferay/projects/able/environments/uat/liferay.yaml` And `production/liferay.yaml`

These environments run no DXP, so they turn on no client extensions, and need no `clientExtensions` entries at all.

### How Base And Environment Values Combine

For each environment, the client extension ApplicationSet merges the environment's `liferay.yaml` over its project's `base/liferay.yaml`, key by key, with the environment winning. Each entry under `clientExtensions` then becomes one Application. So an environment can override one field and inherit the rest, for example running a newer, pinned build in nonproduction only:

```yaml
# base/liferay.yaml: every environment's default
  liferay-sample-etc-spring-boot:
    descriptor: oci://${REGISTRY}/liferay-sample-etc-spring-boot-descriptor
    enabled: false
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
      workload:
        containerPort: 58081
    version: 1.0.5

# environments/nonproduction/liferay.yaml: wins in nonproduction
  liferay-sample-etc-spring-boot:
    enabled: true
    values:
      clientExtension:
        domain: etc-spring-boot-able-nonproduction.runray.xyz
    version: 1.0.6.<12 hex>
```

Nonproduction runs the pinned `1.0.6` build, with `descriptor`, `network`, and `containerPort` from `base` and `enabled` and `domain` from nonproduction. Another environment that turns it on without its own `version` runs `1.0.5`.

Keep `enabled: false` in `base` and `enabled: true` in environments, not the other way around: an environment's `false` cannot turn off a `true` from `base`, because the merge does not let a `false` replace a value that is set. To see what each environment runs:

```bash
kubectl get clientextensions --all-namespaces \
	--output custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,VERSION:.spec.version,PHASE:.status.phase
```

Commit and push the values. Argo CD reads them after the bootstrap in Step 4, and within about three minutes of any later push, or at once with [Make Argo CD Pick Up A Change Now](#make-argo-cd-pick-up-a-change-now).

## Step 4: Bootstrap AWS From The Release Branch

1. Find the CI run for the release branch head. Every `push-rc-*` job must have succeeded; the run then waits at `approve-release`, which is expected.

	```bash
	gh run list \
		--branch "${RELEASE_BRANCH}" \
		--limit 3 \
		--repo cloudnative-team/liferay-portal \
		--workflow ci-publish-cloud-production-artifacts.yaml

	gh run view <run-id> \
		--jq '.jobs[] | select(.name | test("push-rc")) | "\(.conclusion) \(.name)"' \
		--json jobs \
		--repo cloudnative-team/liferay-portal
	```

	If a `test-cloud-terraform` job failed with `403 API rate limit exceeded` from TFLint, the push jobs were skipped; rerun with `gh run rerun <run-id> --failed --repo cloudnative-team/liferay-portal`. `${RC_SHA7}` is the first 7 characters of the run's head commit.

1. Download, verify, and extract the bootstrap:

	```bash
	P="gs://liferay-cloud-native-bootstrap-nonprd/bootstrap/${RELEASE_BRANCH//\//-}/${RC_SHA7}"

	gcloud storage cp \
		"${P}/liferay-aws-bootstrap-${BOOTSTRAP_VERSION}.tar.gz" \
		"${P}/liferay-aws-bootstrap-${BOOTSTRAP_VERSION}.tar.gz.sha256" \
		.

	sha256sum --check "liferay-aws-bootstrap-${BOOTSTRAP_VERSION}.tar.gz.sha256"

	mkdir --parents "liferay-aws-bootstrap-${BOOTSTRAP_VERSION}"

	tar --directory "liferay-aws-bootstrap-${BOOTSTRAP_VERSION}" --extract --file "liferay-aws-bootstrap-${BOOTSTRAP_VERSION}.tar.gz" --gzip
	```

	Every chart and the operator image in `liferay-aws-bootstrap-${BOOTSTRAP_VERSION}/cloud/scripts/config.json.defaults` should point at `.../release-lcd-53904/${RC_SHA7}`.

1. Write `config.json`:

	```json
	{
	  "account_id": "${ACCOUNT_ID}",
	  "deployment_name": "${DEPLOYMENT}",
	  "marketplace": false,
	  "options": {
	    "auto_approve": true,
	    "parallelism": 10
	  },
	  "platformComponents": {
	    "values": {
	      "argoWorkflows": {
	        "gateway": {
	          "enabled": true,
	          "hostname": "argo-${DEPLOYMENT}.${DOMAIN}",
	          "tls": {
	            "externalSecretKey": "liferay-certificates-${CERT_SLUG}"
	          }
	        }
	      },
	      "argocd": {
	        "gateway": {
	          "enabled": true,
	          "hostname": "argocd-${DEPLOYMENT}.${DOMAIN}",
	          "tls": {
	            "externalSecretKey": "liferay-certificates-${CERT_SLUG}"
	          }
	        }
	      },
	      "clientExtensions": {
	        "repoURL": "oci://us-central1-docker.pkg.dev/external-assets-nonprd/liferay-helm-chart/release-lcd-53904/${RC_SHA7}/liferay-client-extension",
	        "targetRevision": "0.1.1"
	      },
	      "gitops": {
	        "repository": {
	          "credentialsSecretName": "liferay-credentials-<name>",
	          "url": "${GITOPS_REPO}"
	        }
	      }
	    }
	  },
	  "region": "${REGION}",
	  "terraform": {
	    "platform": {
	      "keda_config": {
	        "enabled": false
	      },
	      "observability_config": {
	        "enabled": false
	      }
	    }
	  },
	  "tfstate": {
	    "bucket_name": "${DEPLOYMENT}-tfstate"
	  }
	}
	```

	- `clientExtensions` is required for a release candidate. The bootstrap points the charts it installs at the release candidate's registry, but the client extension chart is installed later by an ApplicationSet whose chart source defaults to the production registry. Use the version in `cloud/helm/client-extension/Chart.yaml` on `${RC_SHA7}`.
	- `setup_aws.sh` deep-merges this file over `config.json.defaults`. Only keys under `platformComponents.values` reach the platform chart.

1. Run the bootstrap. It is idempotent, so you can re-run it after any `config.json` change or a newer tarball:

	```bash
	aws sso login

	"./liferay-aws-bootstrap-${BOOTSTRAP_VERSION}/cloud/scripts/setup_aws.sh" config.json

	aws eks update-kubeconfig --name "${DEPLOYMENT}-eks" --region "${REGION}"
	```

	It creates the EKS cluster and the platform, and installs `liferay-platform` into `argocd-system`.

## Step 5: DNS And Wait For The Environment

1. List each hostname with its gateway's load balancer, and create a CNAME for each:

	```bash
	kubectl get gateway --all-namespaces --output json \
		| jq --raw-output '.items[] | "\(.metadata.namespace)/\(.metadata.name) \([.status.addresses[]?.value] | join(","))"'

	kubectl get httproute --all-namespaces --output json \
		| jq --raw-output '.items[] | "\(.metadata.namespace) \(.spec.parentRefs[0].name) \(.spec.hostnames // [] | join(","))"'
	```

	You need records for `argocd-${DEPLOYMENT}`, `argo-${DEPLOYMENT}`, `liferay-able-nonproduction`, `custom-element-2-able-nonproduction`, `etc-node-able-nonproduction`, `etc-spring-boot-able-nonproduction`, and `global-css-1-able-nonproduction`. The Argo CD records exist right after the bootstrap; the environment's gateway and routes appear only once its infrastructure is provisioned, so run this again later. The `*-able-nonproduction` names all point at the nonproduction gateway's load balancer.

1. Wait for the infrastructure (about 20 minutes; RDS and OpenSearch are the slowest), then DXP's first boot (up to about 40 minutes against an empty database):

	```bash
	kubectl get applications --namespace argocd-system

	kubectl get liferayinfrastructure --namespace "${NS}"

	kubectl get pods --namespace "${NS}" --watch
	```

	Done looks like:

	- every Application `Synced` and `Healthy`, including the six `able-nonproduction-liferay-sample-*` ones. The `uat` and `production` app Applications stay `Progressing`, because they run no DXP.
	- `liferay-default-0` `Running` with every container ready.
	- every ClientExtension `Ready`:

	```bash
	kubectl get clientextensions --namespace "${NS}" \
		--output custom-columns=NAME:.metadata.name,PHASE:.status.phase,VERSION:.spec.version,WORKLOAD:.status.workloadSummary,URL:.status.url
	```

	```
	NAME                              PHASE   VERSION   WORKLOAD    URL
	liferay-sample-batch              Ready   1.0.2     Completed   <none>
	liferay-sample-custom-element-2   Ready   1.0.2     1/1         https://custom-element-2-able-nonproduction.runray.xyz
	liferay-sample-etc-cron           Ready   1.0.2     Completed   <none>
	liferay-sample-etc-node           Ready   1.0.2     1/1         https://etc-node-able-nonproduction.runray.xyz
	liferay-sample-etc-spring-boot    Ready   1.0.5     1/1         https://etc-spring-boot-able-nonproduction.runray.xyz
	liferay-sample-global-css-1       Ready   1.0.0     1/1         https://global-css-1-able-nonproduction.runray.xyz
	```

	A ClientExtension stays `Pending` until DXP is up, and the batch Job retries until DXP answers. The cron job shows `Completed` after its first scheduled run.

1. Optionally, check the routes from outside:

	```bash
	curl --output /dev/null --silent --write-out '%{http_code}\n' "https://custom-element-2-able-nonproduction.${DOMAIN}/"
	curl --output /dev/null --silent --write-out '%{http_code}\n' "https://etc-node-able-nonproduction.${DOMAIN}/comic"
	curl --output /dev/null --silent --write-out '%{http_code}\n' "https://etc-spring-boot-able-nonproduction.${DOMAIN}/dad/joke"
	```

	Expect `200`, `401`, and `401`: the backends require a token. A `503` from a backend usually means a missing or wrong `workload.containerPort`.

## Step 6: Sign In To DXP

The admin user is `test@liferay.com`. Its first password is random, set when DXP first creates its database. Read it from the running pod:

```bash
kubectl exec liferay-default-0 --container liferay-default --namespace "${NS}" -- printenv LIFERAY_DEFAULT_PERIOD_ADMIN_PERIOD_PASSWORD
```

Do not read it from the `liferay-default` Secret: under Argo CD the Secret gets a new random value on every render, so it soon stops matching the real password. If the pod has restarted since its first boot, `printenv` has the same problem.

Sign in at `https://liferay-able-nonproduction.${DOMAIN}`, and change the password under **Account Settings** to keep using the account.

## Step 7: Add The Custom Element And Show Both Backends

1. Go to the **Home** page and click the pencil icon to edit it.
1. Open the **Fragments and Widgets** panel, choose the **Widgets** tab, and search for `Liferay Sample Custom Element 2`. It is under **Client Extensions**, which shows that the operator delivered its configuration into DXP.
1. Drag it onto the page and click **Publish**.
1. View the page. The widget shows:
   - "Hello" and your user's name;
   - an xkcd comic from the Node backend (`etc-node`, `/comic`);
   - a dad joke from the Spring Boot backend (`etc-spring-boot`, `/dad/joke`).

In the browser's developer tools, on the **Network** tab:

| Request | Status | Meaning |
| --- | --- | --- |
| `custom-element-2-able-nonproduction.${DOMAIN}/index.<hash>.js` | 200 | The element's code comes from its own Deployment and route, not from DXP |
| `liferay-able-nonproduction.${DOMAIN}/o/oauth2/application?externalReferenceCode=...` ×2 | 200 | The element looks up the OAuth2 applications the backends declared |
| `etc-node-able-nonproduction.${DOMAIN}/comic` | 200 | The Node backend accepted the user's token |
| `etc-spring-boot-able-nonproduction.${DOMAIN}/dad/joke` | 200 | The Spring Boot backend accepted the user's token |

The backends answer `401` without a token, so the `200`s show that DXP issued the browser an OAuth2 token for each backend and the backend validated it.

## Step 8: Edit A Sample Object And Watch The Object Actions

The batch client extension imported a `Sample` object definition with one text field, `field1`, and four `onAfterUpdate` object actions: two call the Node backend and two call the Spring Boot backend.

1. Follow both backends' logs, each in its own terminal:

	```bash
	kubectl logs --follow deployment/liferay-sample-etc-spring-boot --namespace "${NS}"
	```

	```bash
	kubectl logs --follow deployment/liferay-sample-etc-node --namespace "${NS}"
	```

1. In DXP, open the **Applications Menu**, then **Control Panel**, **Object**, **Sample**. Add an entry with any `field1` value and save it.
1. Edit the entry, change `field1`, and save again. The update fires the four object actions within a couple of seconds.

What to see:

- **Spring Boot action 1:** `ObjectAction1RestController` logs the entry's old and new `field1` values, and a validated JWT whose audience is `https://etc-spring-boot-able-nonproduction.${DOMAIN}`. DXP signed the call for that client extension's OAuth2 application, and the backend accepted it through the gateway.
- **Spring Boot action 2:** `ObjectAction2RestController` logs `success`.
- **Node:** it logs that your user is authorized, and the `objectEntryDTOSample` payload.

## Step 9: Show The Cron Job

The cron sample is a Spring Boot command-line app. Each run gets an OAuth2 token for its headless server application, calls DXP's headless delivery API, logs the result, and exits.

1. Show the CronJob and its recent runs:

	```bash
	kubectl get cronjob liferay-sample-etc-cron --namespace "${NS}" \
		--output custom-columns=SCHEDULE:.spec.schedule,POLICY:.spec.concurrencyPolicy,LAST:.status.lastScheduleTime,IMAGE:.spec.jobTemplate.spec.template.spec.containers[0].image

	kubectl get pods --namespace "${NS}" | grep etc-cron
	```

	The schedule comes from `liferay.yaml`, and the image is pinned by digest from the descriptor.

1. Run it now instead of waiting for the next quarter hour:

	```bash
	kubectl create job etc-cron-demo --from=cronjob/liferay-sample-etc-cron --namespace "${NS}"

	kubectl wait --for=condition=complete job/etc-cron-demo --namespace "${NS}" --timeout=300s

	kubectl logs job/etc-cron-demo --namespace "${NS}"
	```

	The log ends with:

	```
	Updated OAuth2 access token at <time>
	There are 0 message board threads in the Guest site at https://liferay-able-nonproduction.runray.xyz
	Dad joke, from the GitHub Actions build: <a joke>
	[INFO  tini (1)] Main child exited normally (with status '0')
	```

	Two `Unable to get client ID: 400 Bad Request` warnings near the start are normal for this sample; the run goes on to get its token.

1. Show what the operator reports, then delete the manual run:

	```bash
	kubectl get clientextension liferay-sample-etc-cron --namespace "${NS}" \
		--output custom-columns=PHASE:.status.phase,VERSION:.spec.version,WORKLOAD:.status.workloadSummary

	kubectl delete job etc-cron-demo --namespace "${NS}"
	```

	It shows `Ready`, its version, and `Completed` for the last run.

To change the schedule for one environment only, set it in that environment's `liferay.yaml`:

```yaml
clientExtensions:
  liferay-sample-etc-cron:
    values:
      workload:
        schedule: "0 * * * *"
```

## Step 10: Show The Global CSS

`liferay-sample-global-css-1` serves a stylesheet that puts blue text reading "Liferay Sample Global CSS 1" at the top of each page body. The file name is in its descriptor's configuration (`url=global.<hash>.css`), and its own Deployment serves it:

```bash
curl --silent "https://global-css-1-able-nonproduction.${DOMAIN}/global.<hash>.css"
```

Load a DXP page and look for the blue text. Its configuration has `scope=company`; if the text does not appear, attach the client extension in the pages' **Design** configuration, under the CSS client extensions.

## Step 11: Show What Is In A Descriptor

A descriptor image holds one file and nothing else: no operating system and nothing to run. Extract one:

```bash
R="${REGISTRY}/liferay-sample-etc-spring-boot-descriptor:1.0.5"

docker pull "${R}"

id=$(docker create "${R}" x)

docker cp "${id}:/client-extension-descriptor.json" - | tar --extract --to-stdout | jq . > client-extension-descriptor.json

docker rm "${id}"
```

It has four parts under `descriptor`:

- `descriptorVersion`: the format version, `1`. The chart refuses any other.
- `type`: `batch`, `configuration`, `frontend`, or `service`. It decides the workload: a Job for `batch`, none for `configuration`, and a Deployment otherwise, unless `values.workload.kind` says otherwise (such as `CronJob`).
- `configs`: the DXP configurations from the project's `client-extension.yaml`, keyed `<configuration class>~<client extension id>`, such as the OAuth2 application, object actions, and workflow actions. The DXP operator delivers these into DXP.
- `image`: the workload image:

	```json
	"image": {
	  "digest": "sha256:<digest>",
	  "repository": "${REGISTRY}/liferay-sample-etc-spring-boot",
	  "tag": "1.0.5"
	}
	```

The workload image is not inside the descriptor; the descriptor points at it:

1. The Application's values file is `cxdescriptor://<registry>/<name>-descriptor:<version>`. The `cxdescriptor` Helm plugin in Argo CD's repo server pulls that image and hands `client-extension-descriptor.json` to Helm as a values file, so it becomes `.Values.descriptor`.
1. The chart's `liferay-client-extension.image` helper builds the container image from `.Values.descriptor.image` as `<repository>:<tag>@<digest>`. `image.repository` and `image.tag` in `values` override it.
1. The kubelet pulls the workload image by that digest:

	```bash
	kubectl get deployment liferay-sample-etc-spring-boot --namespace "${NS}" \
		--output jsonpath='{.spec.template.spec.containers[0].image}'
	```

So a descriptor's digest tag pins both the configuration and, through the digest inside it, the exact workload image.

## Ship A New Version Or Add A Client Extension

- **New version:** change a sample, raise the `version` in its `gradle.properties`, and push. The workflow publishes only that client extension. Then set its `version` in `liferay.yaml`, in `base` or in one environment, and push.
- **New client extension:** copy its project into `liferay-workspace/client-extensions/`, give it a `gradle.properties` with `version=1.0.0`, and push. Make its two new packages public. Then add the summary's snippet to `base/liferay.yaml` with `enabled: false`, plus `values.network` if it serves HTTP; and in each environment that runs it, add `enabled: true`, its `domain`, and a DNS record for that domain.

## Make Argo CD Pick Up A Change Now

The ApplicationSets read the gitops repository every three minutes. To make them read it now:

```bash
kubectl annotate applicationset --all --namespace argocd-system \
	argocd.argoproj.io/application-set-refresh=true --overwrite
```

The controller reconciles within a couple of seconds and removes the annotation itself. New Applications are created and synced at once, and changed ones are updated.

To make every Application re-render its manifests too, which also clears Argo CD's manifest cache:

```bash
kubectl annotate applications --all --namespace argocd-system \
	argocd.argoproj.io/refresh=hard --overwrite
```

**Hard Refresh** on an Application in the Argo CD UI does the same for one Application. `argocd.argoproj.io/refresh` on an ApplicationSet does nothing; it is only for Applications.

See [`cne-cx-next-steps.md`](cne-cx-next-steps.md) for what comes after this release.

## Teardown

Disabling the environment does not delete its infrastructure: with auto-sync and no `allowEmpty`, Argo CD does not prune everything. Run the bootstrap's teardown script, which also deletes the cluster:

```bash
"./liferay-aws-bootstrap-${BOOTSTRAP_VERSION}/cloud/scripts/teardown_aws.sh" config.json
```

Then remove the DNS records and, if you want, the published packages.
