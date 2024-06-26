# generate-build-provenance

GitHub Action to create, sign and upload a build provenance attestation for
artifacts built as part of a workflow.

## Usage

Within the GitHub Actions workflow which builds some artifact you would like to
attest,

1. Ensure that the following permissions are set:

   ```yaml
   permissions:
     id-token: write
     attestations: write
     contents: read # optional, usually required
   ```

   The `id-token` permission gives the action the ability to mint the OIDC token
   necessary to request a Sigstore signing certificate. The `attestations`
   permission is necessary to persist the attestation.

   > **NOTE**: The set of required permissions will be refined in a future
   > release.

1. After your artifact build step, add the following:

   ```yaml
   - uses: github-early-access/generate-build-provenance@main
     with:
       subject-path: '${{ github.workspace }}/PATH_TO_FILE'
   ```

   The `subject-path` parameter should identity the artifact for which you want
   to generate an attestation.

### What is being attested?

The generated attestation is a [SLSA provenance][2] document which captures
non-falsifiable information about the GitHub Actions run in which the subject
artifact was created:

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    {
      "name": "some-app",
      "digest": {
        "sha256": "7d070f6b64d9bcc530fe99cc21eaaa4b3c364e0b2d367d7735671fa202a03b32"
      }
    }
  ],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://slsa-framework.github.io/github-actions-buildtypes/workflow/v1",
      "externalParameters": {
        "workflow": {
          "ref": "refs/heads/main",
          "repository": "https://github.com/user/app",
          "path": ".github/workflows/build.yml"
        }
      },
      "internalParameters": {
        "github": {
          "event_name": "push",
          "repository_id": "706279790",
          "repository_owner_id": "19792534"
        }
      },
      "resolvedDependencies": [
        {
          "uri": "git+https://github.com/user/app@refs/heads/main",
          "digest": {
            "gitCommit": "c607ef44660b66df4d10b0dd6b01f56ec98f293f"
          }
        }
      ]
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/actions/runner/github-hosted"
      },
      "metadata": {
        "invocationId": "https://github.com/user/app/actions/runs/1880241037/attempts/1"
      }
    }
  }
}
```

The provenance statement is signed with a short-lived, [Sigstore][1]-issued
certificate.

If the repository initiating the GitHub Actions workflow is public, the public
instance of Sigstore will be used to generate the attestation signature. If the
repository is private, it will use the GitHub private Sigstore instance.

### Where does the attestation go?

On the actions summary page for a repository you'll see an "Attestations" link
which will take you to a list of all the attestations generated by workflows in
that repository.

![Actions summary view](./.github/attestations.png)

### How are attestations verified?

Attestations can be verified using the [gh-attestation][3] extension for the
[GitHub CLI][4].

## Customization

See [action.yml](action.yml)

### Inputs

- `subject-path` - Path to the artifact for which the provenance will be
  generated.

  Must specify exactly one of `subject-path` or `subject-digest`. Wildcards can
  be used to identify more than one artifact.

- `subject-digest` - Digest of the subject for which the provenance will be
  generated.

  Only SHA-256 digests are accepted and the supplied value must be in the form
  "sha256:\<hex-encoded-digest\>". Must specify exactly one of `subject-path` or
  `subject-digest`.

- `subject-name` - Subject name as it should appear in the provenance statement.

  Required when the subject is identified by the `subject-digest` parameter.
  When attesting container images, the name should be the fully qualified image
  name.

- `push-to-registry` - If true, the signed attestation is pushed to the
  container registry identified by the `subject-name`. Default: `false`.

- `github-token` - Token used to make authenticated requests to the GitHub API.
  Default: `${{ github.token }}`.

  The supplied token must have the permissions necessary to write attestations
  to the repository.

### Outputs

- `bundle` - The JSON-serialized [Sigstore bundle][5] containing the attestation
  and related verification material.

## Sample Workflows

### Identify Artifact by Path

For the basic use case, simply add the `generate-build-provenance` action to
your workflow and supply the path to the artifact for which you want to generate
build provenance.

```yaml
name: build-with-provenance

on:
  workflow_dispatch:

jobs:
  build:
    permissions:
      id-token: write
      attestations: write
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Build artifact
        run: make some-app
      - name: Attest artifact
        uses: github-early-access/generate-build-provenance@main
        with:
          subject-path: '${{ github.workspace }}/some-app'
```

### Identify Artifacts by Wildcard

If you are generating multiple artifacts, you can generate build provenance for
each artifact by using a wildcard in the `subject-path` input.

```yaml
name: build-wildcard-with-provenance

on:
  workflow_dispatch:

jobs:
  build:
    permissions:
      id-token: write
      attestations: write
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v4
      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v5
        with:
          distribution: goreleaser
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Attest artifact
        uses: github-early-access/generate-build-provenance@main
        with:
          subject-path: 'dist/**/my-bin-*'
```

For supported wildcards along with behavior and documentation, see
[@actions/glob][6] which is used internally to search for files.

### Container Image

When working with container images you may not have a `subject-path` value you
can supply. In this case you can invoke the action with the `subject-name` and
`subject-digest` inputs.

If you want to publish the attestation to the container registry with the
`push-to-registry` option, it is important that the `subject-name` specify the
fully-qualified image name (e.g. "ghcr.io/user/app" or
"acme.azurecr.io/user/app"). Do NOT include a tag as part of the image name --
the specific image being attested is identified by the supplied digest.

> **NOTE**: When pushing to Docker Hub, please use "index.docker.io" as the
> registry portion of the image name.

```yaml
name: build-image-with-provenance

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      packages: write
      attestations: write
      contents: read
    env:
      REGISTRY: ghcr.io
      IMAGE_NAME: ${{ github.repository }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push image
        id: push
        uses: docker/build-push-action@v5.0.0
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
      - name: Attest image
        uses: github-early-access/generate-build-provenance@main
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true
```

[1]: https://www.sigstore.dev/
[2]: https://slsa.dev/spec/v1.0/provenance
[3]: https://github.com/github-early-access/gh-attestation
[4]: https://cli.github.com/
[5]:
  https://github.com/sigstore/protobuf-specs/blob/main/protos/sigstore_bundle.proto
[6]: https://github.com/actions/toolkit/tree/main/packages/glob#patterns
