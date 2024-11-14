# spin-sbom-example

This repo is a guide to workflows using SBOMs with Spin applications.

## The WHAT and the WHY

SBOM stands for Software Bill of Materials. It's a detailed inventory of the components that make up a software application including licenses, dependencies, versions and patch statuses. SBOMs are used to track vulnerabilities, audit software, identify risks and more. Please note that the completeness of an SBOM depends on the tool used to generate it. SBOMs can come in different formats. Cyclonedx and SPDX are two popular formats for SBOMs.

Generating an SBOM for a Spin application in CI, attaching that to the packaged Spin application (OCI artifact) and periodically having a scanner scan for vulnerabilities can be a good security practice to adopt in an organization or for project(s). This guide outlines two different ways to use SBOMs with Spin OCI artifacts.

## The Tutorial

### Pre-requisites

1. Download [trivy](https://aquasecurity.github.io/trivy/v0.57/getting-started/installation/) - for generating and scanning SBOMs
2. Download [oras](https://oras.land/docs/installation/) - CLI for working with OCI registries
3. Download [cosign](https://docs.sigstore.dev/cosign/system_config/installation/)
4. Download [spin](https://developer.fermyon.com/spin/v3/install) and have an example (spin) app. If you don't have one, feel free to use this [triage app](https://github.com/michelleN/triage)!

### Generate an SBOM

You'll need a tool to generate an SBOM. There are a variety of tools available ([Syft](https://github.com/anchore/syft), [trivy](https://aquasecurity.github.io/trivy/v0.32/docs/sbom/), etc.). We'll use trivy for this example because we can both generate and scan an sbom with this tool.

```console
cd <your-app-dir>

# This command generates an SBOM in the current directory called sbom.json
trivy fs --format spdx-json -o sbom.json .
```

```console
# Scan SBOM for vulnerabilities using trivy
trivy sbom ./sbom.json
```


### Attach SBOM as an accessory to the Spin OCI artifact

This step assumes requires you to have pushed your spin app to an OCI registry.

```console
spin registry push ghcr.io/<your-github-username>/triage-app:0.1.0
```

Attach SBOM to an OCI artifact using the ORAS CLI. The next step attaches the SBOM as an accessory to the Spin OCI artifact. This feature is available with the referrers API as of OCI v1.1. "Accessories" are associated with a main artificact using the concept of "references". This feature allows you to attach additional metadata, like signatures, SBOMs and other related artifacts to an OCI image or artifact without modifying the original artifact itself. Find more info [here](https://opencontainers.org/posts/blog/2024-03-13-image-and-distribution-1-1/#artifacts).

```console
# Attach SBOM as accessory to OCI artifact
$ oras attach ghcr.io/<your-github-username>/triage-app:0.1.0 sbom.json --artifact-type "application/vnd.sbom.spdx+json"

# Discover SBOM
$ oras discover ghcr.io/<your-github-username>/triage-app:0.1.0 --artifact-type application/vnd.sbom.spdx+json
ghcr.io/michellen/triage-app@sha256:6f9b98f1b0161f50c7ffddfd469052899873f6e35318e622bb566e7d797cdff
└── application/vnd.sbom.spdx+json
    ├── sha256:5aaeb7b009aacf62c430717e2ccb5e6a1eeb14392cdc03f895f5714faeebf2a9
```

Pull and scan SBOM from OCI registry

```console
$ oras pull ghcr.io/michellen/triage-app@sha256:5aaeb7b009aacf62c430717e2ccb5e6a1eeb14392cdc03f895f5714faeebf2a9 -o output

$ trivy sbom output/*
```

### Automating Scanning of SBOMs

Once you have an SBOM in place, you'll want to use a scanner to scan it regularly to ensure there are no new vulnerabilities. One way to do this is to set up a GitHub Action that scans the SBOM on a cron schedule.

### Securing SBOMs

The example above is a great way to understand the basics of SBOMs but lacks in one area. If an SBOM is used to track vulnerabilities but the SBOM is tampered with for any reason, it cannot be relied on. Attestations can help with this problem. Attestations provide metadata or claims about an artifact which can be used to verify its authenticity, integrity, or compliance with certain standards.

"Cosign is a command line tool that allows users to sign and verify software artifacts using Sigstore. Sigstore is a Linux Foundation project that aims to improve the security of open source supply chains by providing public software signing and transparency." More info [here](https://docs.sigstore.dev/quickstart/quickstart-cosign/#quickstart-signing-and-verifying-with-cosign )."

`cosign attest` creates an attestation that includes the SBOM as the payload. This ensures that the SBOm is treated as a claim about the Spin OCI artifact which can be independently verified.

```
# Generate SBOM in cyclonedx format
$ trivy fs --format cyclonedx -o sbom.cdx .

# Set up cosign keypair
$ cd ../ && cosign generate-key-pair

$ cd <your-app-dir>

# Creates and pushes attestation to registry
$ cosign attest --key ../cosign.key --type cyclonedx --predicate sbom.cdx ghcr.io/<your-github-username/triage-app:0.1.0

# Verify attestation and save sbom
$ cosign verify-attestation --key ../cosign.pub --type cyclonedx  ghcr.io/<your-github-username>/triage-app:0.1.0 > sbom.cdx.intoto.jsonl

# Scan sbom
$ trivy sbom ./sbom.cdx.intoto.jsonl
```

If you do not want to upload information about your app to the transparency log (public Rekor instance), you can select `N` via the prompt or pass use the `cosign attest --no upload` flag. Discover more about deploying your own Rekor instance [here](https://medium.com/@sabre1041/setting-up-your-own-rekor-transparency-log-server-using-helm-fc7bbeafb59c).