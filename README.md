# Self-service developer platform

A small platform that runs on a laptop. A developer fills in one form and gets a new GitHub repository with a build pipeline already in place, plus a storage bucket in AWS. Nobody writes cloud configuration by hand.

This is the third project in a series. The first was a supply chain security pipeline (scan, build, sign, verify). The second was a self-healing Kubernetes system. This one puts the first project's pipeline inside a template, so every new service gets it without anyone having to remember to add it.

## How it works

1. In the Backstage portal, the developer picks "New service with storage" and types a name, a description and a team.
2. Backstage creates a public GitHub repository from a starter app. The repository already has a pipeline that scans the code, builds the container image, scans the image, produces a software bill of materials, pushes the image to GitHub's container registry and signs it with Cosign.
3. Backstage opens a pull request on a second repository, `platform-gitops`, asking for a storage bucket.
4. When that pull request is merged, Argo CD sees the new request file and applies it to a local Kubernetes cluster (kind).
5. Crossplane reads the request and creates a real S3 bucket in AWS, with public access blocked and tags set.

```
Backstage form
   |-- new GitHub repo (starter app + pipeline)
   |-- pull request on platform-gitops (storage request)
                          |
                    merge by a person
                          |
                      Argo CD  -->  Crossplane  -->  S3 bucket in AWS
```

The developer never types a region, a bucket naming rule, a tag or a security setting. Those live in the platform.

## What is in this repository

| Folder | What it holds |
|---|---|
| `cluster/` | Settings for the local kind cluster |
| `crossplane/` | The AWS connector, the connection settings, and the platform's definition of a storage request and how it becomes a bucket (`platform/xrd.yaml`, `platform/composition.yaml`) |
| `argocd/` | The Argo CD application that watches the `requests/` folder of `platform-gitops` |
| `templates/new-service/` | The Backstage template: the form, the starter app, the pipeline, and the storage request |
| `portal/` | The Backstage app, with GitHub sign-in |

The `platform-gitops` repository is separate. It holds the request files that Argo CD applies.

## What I tested on purpose

I broke the platform deliberately to see whether it held. Some of it did not, and those findings are the useful part.

- **A full run.** From clicking Create to a finished portal task took about 63 seconds in the run I timed. 47 of those seconds were GitHub rejecting a push four times on an unreliable connection, and Backstage retrying. After the pull request was merged, the bucket was ready in about 95 seconds. If someone merges straight away, the whole thing should take around three minutes. That total is an estimate from the measured parts. My real run took about 20 minutes because the pull request sat waiting for me.
- **Drift.** I deleted the bucket's "block public access" setting by hand in AWS. Crossplane put it back in about 3 to 4 minutes, with nobody touching Git or Kubernetes.
- **A bad request.** At first, a request with a made-up team name was accepted, because the platform only required "some text". The form hid this, but anyone writing a request file directly could get past it. The platform's definition now lists the three allowed teams, and a made-up one is refused. I checked this with a server-side dry run, not a full run through Git and Argo CD.
- **A vulnerable dependency.** New repositories came with a protected `main` branch by default, but the security scan did not run on pull requests and was not a required check, so nothing automatic would have stopped a vulnerable library from being merged. I fixed the template. Now a pull request with a known-vulnerable library (HIGH severity) is blocked by the failed scan alone, and a patched version is allowed through.
- **Who can sign in.** I removed guest sign-in and the setting that let any GitHub account into the portal. I did not try a second, unknown account, so the claim is that the setting was removed, not that an outsider was tried and refused.

## Limits

- It runs on a laptop with kind, not on a managed cluster such as EKS. It creates one kind of resource, an S3 bucket.
- Argo CD checks GitHub on a timer, so there is a wait of up to a few minutes after a merge. A webhook would shorten it. I did not set one up.
- The portal runs on one machine for one user. A shared portal would need real hosting, HTTPS and a proper way to manage which people are allowed in.
- The template only shapes new services. The services I made before fixing the pipeline and branch rules keep the old ones.
- Required approvals are set to zero so one person can merge their own passing pull requests. A real team should require a human reviewer as well as the scan.
- The list of teams exists in two places, the template and the platform definition. They can drift apart.
- The scan catches known vulnerabilities in dependencies and in the image. It does not review application code, and it ignores vulnerabilities that have no fix yet.
- The portal uses a personal access token that can write workflow files. A GitHub App with narrow permissions is the production answer.
- The platform definition is applied to the cluster by hand with `kubectl`, not through Argo CD.

## Running it yourself

There is no one-command setup yet. The rough order is:

1. Create the local cluster: `kind create cluster --config cluster/kind-config.yaml`.
2. Install Crossplane with Helm, then apply `crossplane/provider-aws-s3.yaml`.
3. Create a limited AWS user whose permissions are only S3 actions on buckets named `platform-*`. Store its key in the cluster as a secret called `aws-secret`, then apply `crossplane/provider-config.yaml`.
4. Apply `crossplane/platform/functions.yaml`, `xrd.yaml` and `composition.yaml`. The AWS region is written into `composition.yaml`, so change it to yours.
5. Install Argo CD with Helm and apply `argocd/platform-requests-app.yaml`. It points at the `requests/` folder of `platform-gitops` under my GitHub account, so change the repository to yours.
6. In `portal/`, create a GitHub OAuth app and a fine-grained token, put them in environment variables, and run `yarn install` then `yarn start`. The template file `templates/new-service/template.yaml` also has my GitHub username in it.

Secrets are never stored in this repository. The portal reads them from environment variables, and local configuration files are ignored by Git.
