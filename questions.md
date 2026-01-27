# Questions to discuss

- Should the name of the
  `liferay-infrastructure-provider' application in argocd be called`liferay-aws-infrastructure-provider`
  to match the helm chart name?

- sometimes component is 'storage' and sometimes 'storage-versioning' but not
  split out for others, ok?

- BucketPublicAccessBlock options are not as verbose as equivalent terraform
  block. is this ok?

- composition-resource-name is not specific? but generic ? genus and not
  species?

- I tried to disable liferay app environments in yaml "enabled: false" but it
  was still created?

- Is there inconsistency in instructure_provider_appset naming?

- lets say i want to disable/delete liferayinfra in gitops (enabled: false)
  works but what about for liferay (renaming liferay.yaml to \_liferay.yaml) has
  problems, do we need an 'enabled: false' as well?L
