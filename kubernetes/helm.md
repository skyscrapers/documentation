# Helm

## Setting up Helm v3

- [Install the latest helm binary](https://github.com/helm/helm#install)
  - Tip: if you want to have both helm v2 and helm v3: rename the binary of one or both versions.

## migration from Helm v2 to v3

In order to migrate from Helm v2 to v3 all deployments need to be migrated to their respective namespaces.
This can be done with the [helm-2to3](https://github.com/helm/helm-2to3) plugin:

This post from Helm explains how to set up helm v3 and how to migrate your config and the Helm v2 releases:

- [Official documentation on the v2-v3 migration](https://helm.sh/blog/migrate-from-helm-v2-to-helm-v3/)

### Breaking changes

Helm v2 charts are mostly compatible with version 3. There are however some things you need to be aware from:

- [Tiller is removed](https://v3.helm.sh/docs/faq/#removal-of-tiller)
Once the migration of all the deployments is done we will remove Tiller from our clusters.
- [the way CRDs are handled by Helm has changed](https://helm.sh/docs/topics/charts/#custom-resource-definitions-crds)
The way Helm 3 handles CRDs has been changed. They are now treated as special resources and are never upgraded by Helm once installed. The upgrade operation should be done by the cluster operators (admins) with extra care. Few things about this change:
>
> - CRDs should be put in the crds directory at the top level of chart directory.
> - The YAML files in this directory cannot be templatized like any other resources from templates directory.
> - The crd-install hook , which used to take care of installing CRD files from templates directory has been removed. If there are any files with crd-install annotation, then those are skipped by Helm.
> - Files from crds directory are applied first before rendering the chart. These are never applied again if the CRDs exist in the cluster.

- [Releases are now kept in their respective K8s Namespace](https://v3.helm.sh/docs/faq/#release-names-are-now-scoped-to-the-namespace)
With this change its not needed anymore to add an environment parameter to your deployment. The names only need to be unique per K8s Namespace.
Some of the helm commands have been updated to be more in line with other package managers.

[Full helm 3 changelog](https://v3.helm.sh/docs/faq/#changes-since-helm-2)
