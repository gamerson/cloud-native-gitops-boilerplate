# CNE Client Extension Feature Demo

Step-by-step instructions to stand up a Liferay Cloud Native (CNE) environment on AWS that runs one nonproduction DXP pod with five client extensions, and to demonstrate them working end to end:

| Client extension | Descriptor type | Workload | What it shows |
| --- | --- | --- | --- |
| `liferay-sample-batch` | `batch` | Job | Imports the `Sample` object definition, with four object actions, into DXP |
| `liferay-sample-custom-element-2` | `frontend` | Deployment | A widget served from its own domain that calls both backends |
| `liferay-sample-etc-node` | `service` | Deployment | Node backend: a REST endpoint and two object actions |
| `liferay-sample-etc-spring-boot` | `service` | Deployment | Spring Boot backend: a REST endpoint and two object actions |
| `liferay-sample-etc-cron` | `service` | CronJob | A scheduled job that calls DXP's headless API every 15 minutes |

The flow is: Workspace publishes each client extension as a workload image plus a small descriptor image. The gitops repository's `liferay.yaml` lists the descriptors and versions. Argo CD turns each entry into an Application, the `liferay-client-extension` chart deploys it, and the DXP operator delivers its configuration into DXP.

This was written from a working run of `release/LCD-53904` on `cloudnative-team/liferay-portal` (commit `ee802bc`, AWS bootstrap `4.0.1`) in November 2026. That branch combines cloudnative-team#599, cloudnative-team#600, and liferay-devtools#1091.

## Variables

| Variable | Meaning | Reference value |
| --- | --- | --- |
| `${RELEASE_BRANCH}` | Release branch on `cloudnative-team/liferay-portal` | `release/LCD-53904` |
| `${RC_SHA7}` | First 7 characters of the release branch commit CI built | `ee802bc` |
| `${BOOTSTRAP_VERSION}` | `liferay-aws-bootstrap` in `cloud/scripts/versions.json` on that commit | `4.0.1` |
| `${ACCOUNT_ID}` | AWS account | (yours) |
| `${REGION}` | AWS region | `ca-central-1` |
| `${DEPLOYMENT}` | `deployment_name` in `config.json` | `gma-lcd53904` |
| `${DOMAIN}` | Domain you control, with a wildcard TLS certificate | `runray.xyz` |
| `${CERT_SLUG}` | The domain with the dots removed, used in secret names | `runrayxyz` |
| `${GITOPS_REPO}` | Your copy of this gitops repository | `https://github.com/gamerson/cloud-native-gitops-boilerplate` |
| `${REGISTRY}` | Registry path you push the client extensions to | `ghcr.io/gamerson/lcd-53904` |
| `${VERSION}` | Version you publish the client extensions at | `1.0.0` |
| `${NS}` | Environment namespace | `liferay-able-nonproduction` |

The project is `able` and the environment is `nonproduction` throughout, as in this repository.

## Prerequisites

- **Tools:** `aws`, `gcloud`, `gh`, `git`, `helm`, `jq`, `kubectl`, `terraform`, `docker`, and JDK 21.
- **AWS:** an account where you have administrator access through SSO.
- **GCP:** read access to `gs://liferay-cloud-native-bootstrap-nonprd`, where CI uploads the release candidate bootstrap tarballs.
- **A domain and DNS provider** where you can create CNAME records, and a wildcard TLS certificate for the domain.
- **A container registry** you can push to and that the cluster can pull from anonymously. The reference run used GitHub Container Registry (`ghcr.io`) with public packages.
- **A liferay-portal checkout**, to build the Workspace plugin snapshot.
- **AWS Secrets Manager secrets** in `${REGION}`, which External Secrets reads:
  - `liferay-certificates-${CERT_SLUG}`: JSON with the keys `tls.crt` and `tls.key`. The name must start with `liferay-certificates-` and contain no dots.
  - `liferay-credentials-<name>`: JSON with the keys `git_machine_user_id` and `git_access_token`, the HTTPS login Argo CD uses for `${GITOPS_REPO}`.

## Step 1: Copy The Gitops Repository

Argo CD watches this repository, and you will push changes to it, so you need your own copy.

```bash
gh repo create <your-user>/cne-cx-demo-gitops --public --clone

cd cne-cx-demo-gitops

git remote add upstream https://github.com/gamerson/cloud-native-gitops-boilerplate
git pull upstream master
git push --set-upstream origin master
```

It contains:

- `liferay/`: the gitops values that Argo CD reads. Step 4 explains which settings matter.
- `liferay-workspace/`: a Liferay Workspace with the five sample client extensions. It builds with a snapshot of the Workspace plugin from the release branch, which is checked in under `liferay-workspace/maven-repository/`. Argo CD reads only paths under `liferay/`, so this folder never becomes an Application.
- `.github/workflows/publish-client-extensions.yaml`: a GitHub Actions workflow that publishes the client extensions to GitHub Container Registry (Step 3).
- `cne-cx-feature-demo.md`: this document.

## Step 2: The Workspace Plugin Snapshot

Publishing a client extension as an image plus a descriptor is new in the Workspace plugin (LCD-53642, liferay-devtools#1091) and is not released yet. `liferay-workspace/settings.gradle` loads `com.liferay.gradle.plugins.workspace` `17.2.0-SNAPSHOT` from `liferay-workspace/maven-repository/`, where a build from `${RELEASE_BRANCH}` is checked in. The Liferay repository supplies the plugin's dependencies. Nothing needs to be built for the demo, and the GitHub Actions workflow in Step 3 needs no liferay-portal checkout.

To pick up newer plugin code from the branch, rebuild the snapshot and copy it over the checked-in one:

```bash
cd <liferay-portal checkout>

git fetch https://github.com/cloudnative-team/liferay-portal "${RELEASE_BRANCH}"
git worktree add ../wt-cne-cx-demo FETCH_HEAD

cd ../wt-cne-cx-demo/modules/sdk/gradle-plugins-workspace

../../../gradlew publishToMavenLocal -Psnapshot -x test

cp ~/.m2/repository/com/liferay/com.liferay.gradle.plugins.workspace/17.2.0-SNAPSHOT/com.liferay.gradle.plugins.workspace-17.2.0-SNAPSHOT.{jar,module,pom} \
	<gitops repo>/liferay-workspace/maven-repository/com/liferay/com.liferay.gradle.plugins.workspace/17.2.0-SNAPSHOT/
```

`-Psnapshot` matters. Without it, the build publishes a release version, `17.2.0`. If `Bundle-Version` in `modules/sdk/gradle-plugins-workspace/bnd.bnd` on the branch is no longer `17.2.0`, copy the new `<version>-SNAPSHOT` directory instead and change the version in `settings.gradle`.

## Step 3: Publish The Client Extensions

Publish from your machine (below) or from GitHub Actions ([Publish From GitHub Actions](#publish-from-github-actions)). Either way, set the registry first.

### Publish From Your Machine

1. Set your registry in `liferay-workspace/gradle.properties`:

	```properties
	liferay.workspace.client.extension.registry=${REGISTRY}
	liferay.workspace.client.extension.version=${VERSION}
	```

1. Log in to the registry. For `ghcr.io`, use a personal access token with `write:packages`:

	```bash
	echo "${GITHUB_TOKEN}" | docker login ghcr.io --username <your-user> --password-stdin
	```

1. Publish all five:

	```bash
	cd liferay-workspace

	./gradlew publishClientExtension
	```

	Each client extension pushes two images, each under two tags:

	- `${REGISTRY}/<name>:${VERSION}` and `${REGISTRY}/<name>:${VERSION}.<12-hex digest>`: the workload image.
	- `${REGISTRY}/<name>-descriptor:${VERSION}` and `${REGISTRY}/<name>-descriptor:${VERSION}.<12-hex digest>`: the descriptor, a one-file image holding `client-extension-descriptor.json`.

	The build log prints every `Pushed <reference>@sha256:...` line. The digest-suffixed tag lets an environment pin one exact build.

1. Optionally, check what a descriptor says before deploying it:

	```bash
	./gradlew createClientExtensionDescriptor

	jq '.descriptor | {type, image, oauthApplicationHeadlessServerERC}' \
		client-extensions/*/build/client-extension-descriptor/client-extension-descriptor.json
	```

	The expected types are `batch`, `frontend`, `service`, `service`, and `service` for batch, custom-element-2, etc-cron, etc-node, and etc-spring-boot.

### Publish From GitHub Actions

`.github/workflows/publish-client-extensions.yaml` runs the same `publishClientExtension` on a GitHub runner, logged in to `ghcr.io` as the workflow with its `GITHUB_TOKEN`, so no personal token is needed.

1. Set the registry in `liferay-workspace/gradle.properties` to `ghcr.io/<owner of your gitops repository>/<path>` and push. The workflow can push only to packages under the account that owns the repository.
1. In the repository's **Actions** tab, choose **Publish Client Extensions**, then **Run workflow**, and enter the version.
1. The run's summary lists each descriptor under its digest tag, such as `ghcr.io/<owner>/<path>/liferay-sample-batch-descriptor:1.0.0.92ca470377d5`. That tag, `<version>.<12 hex>`, is all a client extension needs: used as its `version` in an environment's `liferay.yaml`, it pins that exact build, and the descriptor pins its workload image by digest. The same versions are attached to the run as a `liferay.yaml` fragment in the `client-extension-versions-<version>` artifact; fetch it with `gh run download <run-id> --name client-extension-versions-<version>`. The summary also links each package's settings page.

If a package already exists because you published it from your machine first, the workflow cannot push to it until you open that package's settings, find **Manage Actions access**, click **Add Repository**, and give the gitops repository the **Write** role.

### Make The Packages Public

New packages under a personal account are private, and the cluster pulls anonymously. GitHub has no API for a package's visibility, so neither the Gradle build nor the workflow can change it. Do it once per package in the browser; later versions keep the package's visibility:

1. Open `https://github.com/users/<your-user>/packages/container/<path>/settings`, with the `/` in `<path>` written as `%2F`, for example `cne-cx-demo%2Fliferay-sample-batch-descriptor`. The workflow's summary links each one. For an organization, use `orgs/<org>` instead of `users/<your-user>`.
1. Under **Danger Zone**, click **Change visibility**, choose **Public**, and type the package name to confirm.

There are ten packages: the five workload images and the five `-descriptor` images. Check one anonymously:

```bash
docker logout ghcr.io

docker pull "${REGISTRY}/liferay-sample-batch-descriptor:${VERSION}"
```

### Publish A New Build

To publish a new build later, raise `${VERSION}` (or pass `-Pliferay.workspace.client.extension.version=<version>`), publish, and change the versions in `liferay.yaml`.

## Step 4: Edit The Gitops Values

These are the settings that make one nonproduction DXP pod run with the five client extensions. Replace `runray.xyz`, `runrayxyz`, `ghcr.io/gamerson/lcd-53904`, and the versions with your own values throughout.

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
- `forceDestroy: true` lets Crossplane delete the database and buckets on teardown. Keep it for a demo.
- DXP 2026.q3 needs OpenSearch 2.19 or later. Keep the quotes; the value must be a string.

The `backup` and `overlayAccess` blocks in the file are not needed for this demo.

### `liferay/projects/able/environments/nonproduction/infrastructure.yaml`

```yaml
enabled: true
```

Leave `uat` and `production` at `enabled: false`.

### `liferay/projects/able/base/liferay.yaml`

The client extensions, keyed by name. Every environment inherits these, and an environment's own `liferay.yaml` is merged over them key by key:

```yaml
clientExtensions:
  liferay-sample-batch:
    descriptor: oci://${REGISTRY}/liferay-sample-batch-descriptor
    version: ${VERSION}
  liferay-sample-custom-element-2:
    descriptor: oci://${REGISTRY}/liferay-sample-custom-element-2-descriptor
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
    version: ${VERSION}
  liferay-sample-etc-cron:
    descriptor: oci://${REGISTRY}/liferay-sample-etc-cron-descriptor
    values:
      workload:
        kind: CronJob
        schedule: "*/15 * * * *"
    version: ${VERSION}
  liferay-sample-etc-node:
    descriptor: oci://${REGISTRY}/liferay-sample-etc-node-descriptor
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
      workload:
        containerPort: 3001
    version: ${VERSION}
  liferay-sample-etc-spring-boot:
    descriptor: oci://${REGISTRY}/liferay-sample-etc-spring-boot-descriptor
    values:
      network:
        enabled: true
        endpointRef: runray.xyz
      workload:
        containerPort: 58081
    version: ${VERSION}

liferay-default:
  clientExtensions:
    delivery:
      enabled: true
  image:
    repository: liferay/dxp
    tag: 2026.q3.5
```

- `descriptor` and `version` pick the build. `values` is passed straight to the `liferay-client-extension` chart and wins over what the descriptor says.
- `network` gives the three client extensions that serve HTTP a route on the environment's gateway.
- **Workload settings from `LCP.json` are not read.** The descriptor carries the client extension's type, image, and DXP configuration, but not its `LCP.json`. So the settings that `LCP.json` holds on Liferay PaaS go here:
  - `containerPort`: the port the backend listens on. Without it the chart uses port 80 and the route answers `503`.
  - `kind: CronJob` and `schedule` for the cron job. Without them it runs as a Deployment. The CronJob's `concurrencyPolicy` is always `Forbid`.
- `liferay-default.clientExtensions.delivery.enabled: true` lets the DXP operator deliver each client extension's configuration into this environment's DXP. It is `false` by default, and without it no client extension appears in DXP.
- To leave a client extension out of one environment, set `enabled: false` on its entry in that environment's `liferay.yaml`, as `uat` and `production` do here.

The rest of `liferay-default` in this file (licensing, marketplace, a feature flag, an observability network policy, and commented-out experiments) belongs to the reference environment and is not needed for client extensions. Keep or drop it as your environment needs. The reference environment activated DXP through `licensing` against One UAT; use whatever license setup you normally use.

### `liferay/projects/able/environments/nonproduction/liferay.yaml`

Each HTTP client extension's public hostname, one DXP pod, and the settings that DXP needs behind the gateway:

```yaml
clientExtensions:
  liferay-sample-custom-element-2:
    values:
      clientExtension:
        domain: custom-element-2-able-nonproduction.runray.xyz
  liferay-sample-etc-node:
    values:
      clientExtension:
        domain: etc-node-able-nonproduction.runray.xyz
  liferay-sample-etc-spring-boot:
    values:
      clientExtension:
        domain: etc-spring-boot-able-nonproduction.runray.xyz

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

- `clientExtension.domain` is the hostname each client extension is served on, and the address DXP gives the browser and the object actions for it. The domains live per environment so that environments do not collide.
- `LIFERAY_WEB_PERIOD_SERVER_PERIOD_PROTOCOL=https`: the gateway redirects HTTP to HTTPS, so DXP must advertise `https` in the metadata it hands client extensions, or their calls back to DXP get a `301`.
- `resources`: EKS Auto Mode sizes the node from the request, and the chart's 4Gi default gets a node that cannot fit the 12Gi limit.
- `startupProbe.failureThreshold: 240`: the first boot against an empty database runs past the chart's default 15 minute startup budget.
- This repository's copy also pins custom-element-2 to a digest tag (`version: 0.6.0.8862529eb926`) to show pinning. Delete that `version:` line so the base version applies, or replace it with one of your own `${VERSION}.<digest>` tags. The `overlay` block is not needed for this demo.

Commit and push. Argo CD picks the changes up after the bootstrap in Step 5, and again within about three minutes of any later push.

## Step 5: Bootstrap AWS From The Release Branch

1. Find the CI run for the release branch head. It must have published the release candidate (RC). The run then waits at `approve-release`, which is expected.

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

	Every `push-rc-*` job must be `success`. If a `test-cloud-terraform` job failed with `403 API rate limit exceeded` from TFLint, the push jobs were skipped; rerun with `gh run rerun <run-id> --failed --repo cloudnative-team/liferay-portal`. `${RC_SHA7}` is the first 7 characters of the run's head commit.

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

	- `clientExtensions` is required for a release candidate. The RC bootstrap points the charts it installs at the RC registry, but the client-extension chart is installed later by an ApplicationSet, whose chart source defaults to the production registry, where `0.1.1` does not exist yet. Use the `liferay-client-extension` version from `cloud/helm/client-extension/Chart.yaml` on `${RC_SHA7}`.
	- `setup_aws.sh` deep-merges this file over `config.json.defaults`. Only keys under `platformComponents.values` reach the platform chart.
	- The reference run added `operatorApplications.dxpOperator.values` (`provisioning.baseURL: https://api.one-uat.liferay.com` and short intervals) for its licensing setup. Client extensions do not need it.

1. Run the bootstrap. It is idempotent, so you can re-run it after any `config.json` change or a newer tarball:

	```bash
	aws sso login

	"./liferay-aws-bootstrap-${BOOTSTRAP_VERSION}/cloud/scripts/setup_aws.sh" config.json
	```

	It creates the EKS cluster and the platform, then installs `liferay-platform` into `argocd-system`. It ends with `Release "liferay-platform" has been upgraded` (or installed).

1. Point `kubectl` at the cluster if the script did not:

	```bash
	aws eks update-kubeconfig --name "${DEPLOYMENT}-eks" --region "${REGION}"
	```

## Step 6: DNS And Wait For The Environment

1. List each hostname with its gateway's load balancer, and create a CNAME for each one:

	```bash
	kubectl get gateway --all-namespaces --output json \
		| jq --raw-output '.items[] | "\(.metadata.namespace)/\(.metadata.name) \([.status.addresses[]?.value] | join(","))"'

	kubectl get httproute --all-namespaces --output json \
		| jq --raw-output '.items[] | "\(.metadata.namespace) \(.spec.parentRefs[0].name) \(.spec.hostnames // [] | join(","))"'
	```

	You need records for `argocd-${DEPLOYMENT}`, `argo-${DEPLOYMENT}`, `liferay-able-nonproduction`, `custom-element-2-able-nonproduction`, `etc-node-able-nonproduction`, and `etc-spring-boot-able-nonproduction`. The Argo CD records exist right after the bootstrap. The environment's gateway and routes appear only once its infrastructure is provisioned, so run this again after the next step.

1. Wait for the infrastructure (about 20 minutes: RDS and OpenSearch are the slowest), then DXP's first boot (up to about 40 minutes against an empty database):

	```bash
	kubectl get applications --namespace argocd-system

	kubectl get liferayinfrastructure --namespace "${NS}"

	kubectl get pods --namespace "${NS}" --watch
	```

	Done looks like:

	- every Application `Synced` and `Healthy`, including the five `able-nonproduction-liferay-sample-*` ones (the `uat` and `production` app Applications stay `Progressing`, because they run no DXP);
	- `liferay-default-0` `Running` with every container ready;
	- the ClientExtensions all `Ready`:

	```bash
	kubectl get clientextensions --namespace "${NS}" \
		--output custom-columns=NAME:.metadata.name,PHASE:.status.phase,VERSION:.spec.version,WORKLOAD:.status.workloadSummary,URL:.status.url
	```

	```
	NAME                              PHASE   VERSION   WORKLOAD    URL
	liferay-sample-batch              Ready   1.0.0     Completed   <none>
	liferay-sample-custom-element-2   Ready   1.0.0     1/1         https://custom-element-2-able-nonproduction.runray.xyz
	liferay-sample-etc-cron           Ready   1.0.0     Completed   <none>
	liferay-sample-etc-node           Ready   1.0.0     1/1         https://etc-node-able-nonproduction.runray.xyz
	liferay-sample-etc-spring-boot    Ready   1.0.0     1/1         https://etc-spring-boot-able-nonproduction.runray.xyz
	```

	A ClientExtension stays `Pending` until DXP is up, and the batch Job retries until DXP answers, so expect them to settle after `liferay-default-0` is ready. The cron job shows `Completed` only after its first scheduled run.

1. Optionally, check the backends from outside. Without a token they answer `401`, which shows the route reaches the app:

	```bash
	curl --output /dev/null --silent --write-out '%{http_code}\n' "https://custom-element-2-able-nonproduction.${DOMAIN}/"
	curl --output /dev/null --silent --write-out '%{http_code}\n' "https://etc-node-able-nonproduction.${DOMAIN}/comic"
	curl --output /dev/null --silent --write-out '%{http_code}\n' "https://etc-spring-boot-able-nonproduction.${DOMAIN}/dad/joke"
	```

	Expect `200`, `401`, and `401`. A `503` from a backend usually means a missing or wrong `workload.containerPort`.

## Step 7: Sign In To DXP

The admin user is `test@liferay.com`. Its first password is random, set from the `liferay-default` Secret when DXP first creates its database. Read it from the running pod, which keeps the value it started with:

```bash
kubectl exec liferay-default-0 --container liferay-default --namespace "${NS}" -- printenv LIFERAY_DEFAULT_PERIOD_ADMIN_PERIOD_PASSWORD
```

Do not read it from the Secret. Under Argo CD the chart's `lookup` sees nothing, so the Secret gets a new random value on every render and soon stops matching the real password. If the pod has restarted since its first boot, `printenv` has the same problem; reset the password from the database or redeploy instead.

Sign in at `https://liferay-able-nonproduction.${DOMAIN}`. To keep using the account, change its password under **Account Settings**.

## Step 8: Add The Custom Element And Show Both Backends

1. Go to the **Home** page and click the pencil icon to edit it.
1. Open the **Fragments and Widgets** panel on the left, choose the **Widgets** tab, and search for `Liferay Sample Custom Element 2`. It is under **Client Extensions**, which shows that the operator delivered its configuration into DXP.
1. Drag it onto the page and click **Publish**.
1. View the page. The widget shows:
   - "Hello" and your user's name;
   - an xkcd comic, fetched from the Node backend (`etc-node`, `/comic`);
   - a dad joke, fetched from the Spring Boot backend (`etc-spring-boot`, `/dad/joke`).

What happens behind it, which you can show in the browser's developer tools on the **Network** tab:

| Request | Status | Meaning |
| --- | --- | --- |
| `custom-element-2-able-nonproduction.${DOMAIN}/index.<hash>.js` | 200 | The element's code comes from its own Deployment and route, not from DXP |
| `liferay-able-nonproduction.${DOMAIN}/o/oauth2/application?externalReferenceCode=...` ×2 | 200 | The element looks up the OAuth2 user-agent applications the backends declared |
| `etc-node-able-nonproduction.${DOMAIN}/comic` | 200 | The Node backend accepted the user's token |
| `etc-spring-boot-able-nonproduction.${DOMAIN}/dad/joke` | 200 | The Spring Boot backend accepted the user's token |

Both backends answer `401` without a token (Step 6), so the `200`s show that DXP issued the browser an OAuth2 token for each backend and the backend validated it.

## Step 9: Edit A Sample Object And Watch The Object Actions

The batch client extension imported a `Sample` object definition with one text field, `field1`, and four `onAfterUpdate` object actions: two call the Node backend and two call the Spring Boot backend.

1. In one terminal, follow both backends' logs:

	```bash
	kubectl logs --follow deployment/liferay-sample-etc-spring-boot --namespace "${NS}"
	```

	```bash
	kubectl logs --follow deployment/liferay-sample-etc-node --namespace "${NS}"
	```

1. In DXP, open the **Applications Menu**, then **Control Panel**, **Object**, **Sample**. Add an entry with any `field1` value and save it.
1. Edit the entry, change `field1`, and save again. The update fires the four object actions within a couple of seconds.

What to see:

- **Spring Boot:** `ObjectAction1RestController` logs the entry's old and new `field1` values, and a validated JWT whose audience is `https://etc-spring-boot-able-nonproduction.${DOMAIN}`. DXP signed the call for that client extension's OAuth2 application, and the backend accepted it through the gateway.
- **Node:** it logs that your user is authorized, and the `objectEntryDTOSample` payload.
- **Expected error:** Spring Boot's `ObjectAction2RestController` throws a `JSONException` about `JSONObject["modelDTOAccount"]`. That handler is written for Account updates, and the sample also wires it to `Sample`. The request arrived with a valid token, so this is a bug in the sample, not in the deployment.

The same can be done without the UI by calling `/o/c/samples` with a signed-in session. Basic authentication is refused with `403`.

## Step 10: Show The Cron Job

The cron sample is a Spring Boot command-line app. Each run gets an OAuth2 token for its headless server application, calls DXP's headless delivery API, logs the result, and exits.

1. Show the CronJob and its recent runs:

	```bash
	kubectl get cronjob liferay-sample-etc-cron --namespace "${NS}" \
		--output custom-columns=SCHEDULE:.spec.schedule,POLICY:.spec.concurrencyPolicy,LAST:.status.lastScheduleTime,IMAGE:.spec.jobTemplate.spec.template.spec.containers[0].image

	kubectl get pods --namespace "${NS}" | grep etc-cron
	```

	The schedule `*/15 * * * *` comes from `liferay.yaml`, and the image is pinned by digest from the descriptor.

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
	Dad joke: <a joke>
	[INFO  tini (1)] Main child exited normally (with status '0')
	```

	Two `Unable to get client ID: 400 Bad Request` warnings near the start are normal for this sample. They appear on every run, and the run goes on to get its token.

1. Show what the operator reports:

	```bash
	kubectl get clientextension liferay-sample-etc-cron --namespace "${NS}" \
		--output custom-columns=PHASE:.status.phase,VERSION:.spec.version,WORKLOAD:.status.workloadSummary
	```

	`Ready`, your version, and `Completed` for the last run.

1. Delete the manual run:

	```bash
	kubectl delete job etc-cron-demo --namespace "${NS}"
	```

To change the schedule for one environment only, set it in that environment's `liferay.yaml`; the merge keeps the rest of the entry from `base`:

```yaml
clientExtensions:
  liferay-sample-etc-cron:
    values:
      workload:
        schedule: "0 * * * *"
```

## Optional: Ship A New Version

To show an upgrade, change something in a sample, publish with a new version (`./gradlew publishClientExtension -Pliferay.workspace.client.extension.version=1.0.1`), and change that entry's `version` in `liferay.yaml`. Argo CD rolls out the new build, and the ClientExtension's `VERSION` column follows. To pin one environment to an exact build, use the digest-suffixed tag the publish printed, such as `1.0.1.8862529eb926`.

## Teardown

Disabling the environment does not delete its infrastructure on its own: with auto-sync and no `allowEmpty`, Argo CD will not prune everything. Run the bootstrap's teardown script, which also deletes the cluster:

```bash
"./liferay-aws-bootstrap-${BOOTSTRAP_VERSION}/cloud/scripts/teardown_aws.sh" config.json
```

Then remove the DNS records and, if you want, the published packages.
