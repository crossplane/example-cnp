# Platform API as Configuration

This repository contains configuration for the Cloud Native Platform (CNP). CNP
consists of a series of Crossplane packages that are automatically built and
pushed to the Upbound Cloud Registry by CI/CD when commits are merged to this
repository. The following packages exist:

* `eks.clusters.cnp.example.org` - Defines a composite resource representing an
  EKS cluster (including all network and IAM dependencies).
* `fluentbits.charts.cnp.example.org` - Defines a composite resource
  representing a Helm release of the Fluent Bit logging daemonset.
* `clusters.cnp.example.org` - Defines a composite resource representing a 'CNP
  Kubernetes Cluster'. This cluster is composed of the EKS cluster and Fluent
  Bit composite resources above, and thus this package depends on the above two.
  **The files in this package contain explanatory annotations; it's the most
  interesting one to look at.**

The root of each package directory contains a file named `crossplane.yaml`. This
file is required. A Crossplane package is an OCI image that is expected to emit
this file, followed by any packaged resources in the order they should be
applied, when its entrypoint is invoked.

To create a package, the package author (or their CI/CD system) runs the
following commands:

```bash
# Any OCI image that omits a YAML resource of kind: Configuration or kind:
# Provider (in the pkg.crossplane.io API Group) followed by zero or more
# supplemental resources when its ENTRYPOINT is invoked is considered a valid
# Crossplane package. This tools builds a valid package by reading a file named
# crossplane.yaml at the root of the working directory (which must contain a
# kind: Configuration or a kind: Provider), then discovering all Crossplane
# resources in or under the working directory. These resources are concatenated
# into a YAML stream in a randomly generated filename (to avoid implementations
# relying on the path of this file), and the ENTRYPOINT of the OCI image is set
# to `cat randomlynamedstream.yaml`.
$ crossplane package build --tag exampleorg/cnp-cluster:v0.1.0

# The `crossplane package` command provides a similar interface to the `docker`
# tool frequently used to build, tag, and push images. `crossplane package`
# publishes packages to the Upbound Cloud Registry rather than the Docker Hub
# by default. These tools are recommended but optional; you could package a
# Python script that emitted a valid stream of YAML if you wanted to.
$ crossplane package push exampleorg/cnp-cluster:v0.1.0
```

Note that package metadata does not include a version. Crossplane packages use
the package (image) tag as their version, and thus require that tags be valid
semantic versions. Images built by CI/CD as part of a gitops flow are encouraged
to derive their semantic version from a git tag. In such a case all packages
defined in this repository would be versioned in lock-step.

## Proposed Changes

This sketch assumes the acceptance of the in-flight [Package Manager refactor].
It also:

* Renames `InfrastructureDefinition` to `CompositeResourceDefinition` (XRD).
* Replaces `InfrastructurePublication` with a configuration field on an XRD.
* Automatically derives UI metadata from the OpenAPI schema of an XRD.
* Renames the `spec.to` and `spec.from` of a `Composition` to
  `spec.compositeResource` and `spec.composedResources`, in order to allow for
  'bidirectional patching' - i.e. patching from composed resource status to
  composite resource status in addition to patching from composite resource spec
  to composed resource spec (not yet demonstrated in this repo).
* Uses `metadata.annotations` to represent most package metadata (maintainer,
  company, etc) in order to loosen the contract around purely informational
  package metadata.
* Couples the creation and aggregation of suggested RBAC ClusterRoles to the
  existence of XRDs, rather than Packages.

When automatic RBAC ClusterRole management is enabled by an XRD (management is
enabled by default) Crossplane will create `admin`, `edit`, and `view` RBAC
ClusterRoles granting access to the defined kind and its requirement (if any).
These ClusterRoles will labelled such that they aggregate (as appropriate) to
the ClusterRoles intended to be granted to:

* The Crossplane (Deployment) service account.
* The Crossplane superuser (`crossplane-admin`).
* Subjects allowed to administrate, edit, or view all Crossplane resources.

Furthermore, Crossplane will create `admin`, `edit`, and `view` ClusterRoles
corresponding to any namespace that is labelled with the name of one or more
XRD, for example `rbac.crossplane.io/clusters.cnp.example.org`. If the named XRD
exists and has `rbacPolicy: ManageClusterRoles` the ClusterRole corresponding to
the labelled namespace will have a `matchLabels` stanza added to its
`clusterRoleSelectors` such that the XRD's managed cluster roles aggregate up
to it. This allows consumers of the Crossplane API that wish to simulate a
kind of resource existing or not existing at the namespace level (i.e. Upbound
Cloud) to read and write namespace annotations to determine which resources are
allowed (by RBAC) in a particular namespace.

[Package Manager refactor]: https://github.com/crossplane/crossplane/pull/1616
