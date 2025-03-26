# OpenTofu v1.10 alpha testing guide for OCI Registry Provider Mirrors

The first alpha release for OpenTofu v1.10.0 includes new functionality for using repositories in OCI Registries as a new kind of "mirror" for installing providers.

A "provider mirror" is a secondary location hosting a copy of a provider to be used instead of accessing the provider's primary OpenTofu Provider Registry. Creating a local mirror of some or all of the providers you use can reduce data transfer costs and can help with running OpenTofu in "air-gapped" environments that cannot access any services over the public internet.

This temporary GitHub Organization is here to help support prerelease testing efforts for the v1.10.0 release. Everything in this organization will be deleted once OpenTofu v1.10.0 final is released, so **do not depend on anything in this GitHub organization outside of OpenTofu v1.10 prerelease testing activities**.

## Current Known Quirks and Limitations

Although the fundamentals of this new mirror source are already implemented, it still has some quirks that we intend to address in further development before the final release:

- Similar to OpenTofu's other "mirror" installation methods, the system currently relies on each mirrored provider having a primary installation source in an upstream OpenTofu provider registry, which acts as the authority for which checksums are correct for the provider, so that the [Dependency Lock File](https://opentofu.org/docs/language/files/dependency-lock/) can be properly updated.

    Before final release we intend to improve this so that the OCI mirror source alone is able to record the checksum of the package it installed as part of the dependency lock file, which will reduce the dependency on an upstream registry when using the final v1.10.0 release.
    
    We are also intending to support using OCI Registries as a _primary source_ for providers in a later release -- that is, as a replacement for OpenTofu's Provider Registry Protocol rather than as a new kind of mirror -- but that requires considerable further design work for artifact signature verification and so that will not be included in the first round of features included in OpenTofu v1.10.

- We do not currently have any OpenTofu-specific tools for constructing an OCI artifact for a provider release.

    For the initial release we intend to document how to use [ORAS](https://oras.land/) CLI to assemble valid artifacts, although the ORAS project has not yet released support for constructing multi-platform index manifests and so for alpha we have documented how to construct such a manifest manually. We hope that by the time of the final v1.10.0 release ORAS v1.3.0 will be ready and we can then document how to use it to generate the index manifest.
    
    In a later release we hope to introduce new functionality similar to the current `tofu providers mirror` which is able to automatically assemble manifests for a set of providers and push them directly to a remote registry in a single action, which will then make the construction of an OCI mirror involve a similar level of effort as constructing a "HTTP mirror" for the OpenTofu-specific network mirror protocol today.
    
- We currently have only an early draft of the relevant documentation, in [OCI Registry Integrations](http://opentofu.org/docs/main/cli/oci_registries/). We will continue to improve and elaborate this documentation before the final release.

Due to these current quirks and limitations, it may be challenging to participate in initial alpha testing if you are not already quite familiar with concepts related to the OCI Distribution protocol.

## Pre-built Provider OCI Artifacts

Because the process for building provider artifacts is not yet finalized and currently involves a number of manual steps, we have constructed a small set of pre-built artifacts for use during alpha testing.

> [!WARNING]
>
> These artifacts are **not intended for production use** and will be deleted once OpenTofu v1.10.0 prerelease testing is completed.

If you want to _directly_ use these temporary artifacts during your testing, you can add the following to your [CLI Configuration file](https://opentofu.org/docs/main/cli/config/config-file/):

```hcl
provider_installation {
  oci_mirror {
    repository_template = "ghcr.io/opentofu-v1-10-alpha-testing/provider-${namespace}-${type}"
    include             = ["registry.opentofu.org/*/*"]
  }
}
```

With this configuration, OpenTofu is required to use _only_ the OCI repositories matching the given `repository_template` when installing providers, and so only the subset of providers included in our temporary set will be available for use. All others will fail to install. You can find which providers are currently available in [the package index](https://github.com/orgs/opentofu-v1-10-alpha-testing/packages).

Once you have finished alpha testing, you can restore your original CLI Configuration file (if any) to return to your previous provider installation settings.

### Artifacts in your own Registry

If you would prefer to test the new functionality using artifacts in an OCI Registry you directly control, you can directly copy the artifacts from our temporary test repositories into your own repositories.

You _must_ use a systematic naming scheme when creating your own repositories so that you can redefine the `repository_template` argument to represent your own registry's naming scheme. OpenTofu can support any naming scheme where the namespace and provider type from the provider source address can be interpolated directly into the repository address.

If you are feeling particularly adventurous, you can also construct your own artifacts by following the instructions in [Assembling and Pushing Provider Manifests](http://opentofu.org/docs/main/cli/oci_registries/provider-mirror/#assembling-and-pushing-provider-manifests). This process is currently _very_ manual, but we intend to refine it before the final release.

## Installing from OCI Mirrors

With the `oci_mirror` installation method selected in your CLI configuration, `tofu init` will attempt to install all needed providers from OCI repositories by transforming the provider source address into an OCI repository address using the specified `repository_template` setting.

"Mirror" installation sources rely on checksums provided from the origin registry to verify whether the mirror contains correct content, so if you don't already have the relevant providers tracked in your dependency lock file you will also need to run `tofu providers lock` to fetch the provider's official checksums from the registry.

```shellsession
$ tofu providers lock
- Fetching hashicorp/azurerm 4.24.0 for linux_amd64...
- Retrieved hashicorp/azurerm 4.24.0 for linux_amd64 (signed, key ID 0C0AF313E5FD9F80)
- Fetching hashicorp/google 6.27.0 for linux_amd64...
- Retrieved hashicorp/google 6.27.0 for linux_amd64 (signed, key ID 0C0AF313E5FD9F80)
- Fetching hashicorp/null 3.2.3 for linux_amd64...
- Retrieved hashicorp/null 3.2.3 for linux_amd64 (signed, key ID 0C0AF313E5FD9F80)
- Fetching hashicorp/aws 5.92.0 for linux_amd64...
- Retrieved hashicorp/aws 5.92.0 for linux_amd64 (signed, key ID 0C0AF313E5FD9F80)
- Obtained hashicorp/azurerm checksums for linux_amd64; This was a new provider and the checksums for this platform are now tracked in the lock file
- Obtained hashicorp/google checksums for linux_amd64; This was a new provider and the checksums for this platform are now tracked in the lock file
- Obtained hashicorp/null checksums for linux_amd64; This was a new provider and the checksums for this platform are now tracked in the lock file
- Obtained hashicorp/aws checksums for linux_amd64; This was a new provider and the checksums for this platform are now tracked in the lock file

Success! OpenTofu has updated the lock file.

Review the changes in .terraform.lock.hcl and then commit to your
version control system to retain the new checksums.

$ tofu init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/azurerm from the dependency lock file
- Reusing previous version of hashicorp/google from the dependency lock file
- Reusing previous version of hashicorp/null from the dependency lock file
- Reusing previous version of hashicorp/aws from the dependency lock file
- Installing hashicorp/aws v5.92.0...
- Installed hashicorp/aws v5.92.0 (verified checksum)
- Installing hashicorp/azurerm v4.24.0...
- Installed hashicorp/azurerm v4.24.0 (verified checksum)
- Installing hashicorp/google v6.27.0...
- Installed hashicorp/google v6.27.0 (verified checksum)
- Installing hashicorp/null v3.2.3...
- Installed hashicorp/null v3.2.3 (verified checksum)

OpenTofu has been successfully initialized!

You may now begin working with OpenTofu. Try running "tofu plan" to see
any changes that are required for your infrastructure. All OpenTofu commands
should now work.

If you ever set or change modules or backend configuration for OpenTofu,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

The `tofu providers lock` command first prepopulates the dependency lock file with all of the official checksums for each provider, retrieved from the origin registry. You only need to perform this step once unless you change which versions of providers are required by your configuration.

The `tofu init` command then installs the providers from the OCI mirror source, verifying the mirrored artifacts against the official checksums that were previously recorded.

In practical use, the `tofu providers lock` command would be run in a development environment that is able to access the origin registry, while the `tofu init` command can be run in an "air-gapped" environment that only has access to the OCI mirror.

We intend to reduce the need for the separate `tofu providers lock` step before the final release. We intend to support OCI registries as the _primary_ source for certain providers -- as an alternative to the OpenTofu-specific Provider Registry protocol -- in a later OpenTofu release.
