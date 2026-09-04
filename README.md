# ecs-bluegreen-lab-app

The application half of the **ECS Fargate Blue/Green CI/CD lab**: a
minimal Spring Boot app whose only job is to render the landing page
required by the lab (your full name + the lab name), plus enough
production-shaped scaffolding (health endpoint, container metadata,
version stamp) to make a blue/green deployment visibly verifiable.
Infrastructure (the CloudFormation that provisions the VPC, ECS service,
ECR repo, and CI/CD pipeline this app deploys into) lives in the separate
[`ecs-bluegreen-lab-infra`](https://github.com/1MuhireDavid/ecs-bluegreen-lab-infra)
repo.

## Layout

```
src/main/java/com/labs/ecsdemo/     # Spring Boot app
src/main/resources/
  application.yaml                  # reads APP_OWNER_NAME/LAB_NAME/APP_VERSION
  templates/index.html              # the landing page
Dockerfile                          # multi-stage build, non-root runtime user
ecs/
  taskdef.json                      # CodeDeploy task definition template
  appspec.yaml                      # CodeDeploy appspec template
```
(the GitHub Actions workflow that builds/pushes this app lives at
`.github/workflows/build-and-push.yml` -- this repo holds only this app,
so unlike a shared monorepo lab there's no path filter needed)

## Run locally

```bash
cd ecs-bluegreen-lab-app
mvn spring-boot:run
# or
docker build -t ecsdemo --build-arg APP_VERSION=local .
docker run -p 8080:8080 -e APP_OWNER_NAME="Jane Doe" ecsdemo
curl localhost:8080
```

## Image tagging strategy: consistent and immutable

Every successful build on `main` pushes **exactly one** tag:
`sha-<12-char-git-sha>`. It's never reused -- a brand-new tag every build.

This only works as a deploy trigger because the ECR repository itself is
created with `ImageTagMutability: IMMUTABLE`
(`ecs-bluegreen-lab-infra/cfn/modules/03-ecr-endpoints.yaml`), which also
means there is **no** floating `:latest` pointer for anything to watch.
Instead, the infra repo's `EcrPushRule` (in `05-cicd-pipeline.yaml`)
watches for *any* successful push of a `sha-*` tag and passes the exact
image **digest** that was just pushed into CodePipeline as a
[source-revision override](https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-trigger-source-overrides.html) --
so the pipeline always deploys the specific image this workflow just
built, regardless of tag.

## Why a bootstrap placeholder image

The ECS service, ALB target groups, and CodeDeploy all get created by
CloudFormation the very first time the infra stack runs -- before this
app's GitHub Action has ever pushed a real image. To avoid a chicken-
and-egg failure (`CREATE_COMPLETE` blocked on an image that doesn't
exist), the task definition's `InitialImageTag` parameter defaults to
the sentinel value `bootstrap`, which the infra template swaps for a
public `nginx` image (remapped to listen on the same container port) --
see `../ecs-bluegreen-lab-infra/cfn/modules/04-alb-ecs.yaml`. The ALB
health check path is `/` for exactly this reason: it's the one path both
nginx and this app answer with `200`. Once you push a real image and
CodeDeploy runs its first blue/green release, CodeDeploy -- not this
parameter -- owns the running task definition from then on.

## One-time setup for `ecs/taskdef.json`

`taskdef.json` is a template, but CodePipeline's `CodeDeployToECS` action
only substitutes the `<IMAGE1_NAME>` placeholder automatically. The IAM
role ARNs, account ID, and region are stable values, so fill them in once
by editing the file directly -- no CLI needed:

1. Open `ecs/taskdef.json` in GitHub's web editor (pencil icon) or any
   local text editor.
2. Replace the three placeholders:
   - `<AWS_ACCOUNT_ID>` -- find it in the **AWS Console**: click your
     account name in the top-right corner; your 12-digit Account ID is
     shown in that menu (also on the **Account** page under your name).
   - `<AWS_REGION>` -- the region shown in the Console's top-right region
     selector (whichever region you deployed the infra stack into).
   - `<YOUR_FULL_NAME>` -- your name, exactly as you want it displayed.

   > This file may already have real-looking values in it from a
   > previous deployment attempt -- double-check they're still correct
   > for your current account/region rather than assuming they are.
3. Commit directly to `main` (GitHub web UI's **Commit changes** button,
   or a normal `git commit` + `git push`).

`ecs/appspec.yaml`'s `<TASK_DEFINITION>` placeholder is different: it's a
**literal string** CodeDeploy itself substitutes at deploy time with the
ARN of the task definition revision it just registered -- leave it as-is.

## Required GitHub repo secrets

Added via **Settings -> Secrets and variables -> Actions** on this repo,
using values read from the bootstrap stack's Outputs tab (see the infra
repo's README):

| Secret / variable | Example |
|---|---|
| `AWS_ECR_PUSH_ROLE_ARN` (secret) | `arn:aws:iam::123456789012:role/ecs-bluegreen-lab-gha-ecr-push-role` |
| `ECR_REPOSITORY` (secret) | `ecs-bluegreen-lab-app` |
| `AWS_REGION` (variable) | `us-east-1` |

## What happens on push to `main`

1. `build-and-push.yml` assumes `AWS_ECR_PUSH_ROLE_ARN` via **OIDC** -- a
   role that (via the `job_workflow_ref` trust condition) only this exact
   workflow file, in this exact repo, can assume.
2. Builds the image, tags it `sha-<sha>`, pushes it.
3. That push fires an `ECR Image Action` event -> the `EcrPushRule`
   EventBridge rule in the infra stack, which extracts the exact image
   **digest** from the event and starts `ecs-bluegreen-lab-pipeline` with
   that digest as a source-revision override.
4. CodePipeline resolves that exact image (ECR source action, overridden
   to this digest) and this repo's `ecs/appspec.yaml` + `ecs/taskdef.json`
   (GitHub source action, `DetectChanges: false` so it never self-triggers
   on an unrelated commit like this README), hands both to CodeDeploy.
5. CodeDeploy registers a new task definition revision, spins up "green"
   tasks, waits for them to pass the ALB health check, shifts the
   listener's traffic from "blue" to "green", then terminates the old
   "blue" tasks.
