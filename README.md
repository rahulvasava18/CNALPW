# AWS CI/CD Practical: Starter Kit and Help File

Read this first (5 minutes). It gives hints, not full solutions.
AI tools are allowed, but you must explain and defend what you submit.

## What is in this kit
| Path | Purpose |
|---|---|
| app/app.py, test_app.py, requirements.txt | Flask service on port 8080 with `/` and `/health` + unit tests |
| app/Dockerfile | Container image |
| buildspec.yml | Starter build file (has mistakes) |
| taskdef.json | Reference values for your ECS task definition: container name `app`, port `8080` |
| iam/codebuild-ecr-policy.json | Sample ECR permissions to compare with your role |

## First 5 minutes
1. Run the tests locally if you can: `cd app && pip install -r requirements.txt && pytest`
2. Push this kit to the provided repo on branch `main`.
3. Note down your AWS region and repository URL. You create the ECR repository, ECS cluster, task definition and service yourself (see Task 1 hints). Suggested names: `cicd-exam-app` (ECR and task definition), `cicd-exam-cluster`, `cicd-exam-service`, `cicd-exam-alb`; container name `app`.
4. Keep a text file open for your prompt log (Task 5).

## Task 1 hints: Create the platform, then the Build stage
Create these in the console, in the exam region, using the default VPC:
1. ECR: private repository `cicd-exam-app`, scan on push on.
2. ECS: cluster `cicd-exam-cluster` (AWS Fargate only).
3. Task definition: family `cicd-exam-app`, Fargate, 0.25 vCPU, 0.5 GB, container name `app`, port 8080, image `<ECR URI>:placeholder`. `taskdef.json` shows the expected values.
4. ECS service `cicd-exam-service`: public subnets, public IP on, Application Load Balancer `cicd-exam-alb` (listener 80, target group health check `/health`, container port 8080), **desired tasks 0** (no image exists yet).
5. Security groups: ALB allows TCP 80 from anywhere; the service allows TCP 8080 from the ALB's security group.
6. After the first successful Deploy stage, set the service's desired tasks to 1.

Then build the pipeline:
- CodePipeline needs a Source stage (trigger: push to `main`) and a Build stage using CodeBuild.
- Docker builds inside CodeBuild need **privileged mode** enabled on the build project.
- The CodeBuild service role needs ECR permissions (compare with `iam/codebuild-ecr-policy.json`).
- Read the starter `buildspec.yml` carefully. Find four problems:
  - How does Docker get permission to push to ECR?
  - Is the variable used for the image tag ever defined?
  - Does the container name in the artifact match the task definition?
  - Is the artifact file name exactly what the ECS deploy action expects?
- Useful commands (fill in the blanks yourself):
  ```
  aws ecr get-login-password --region <REGION> | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com
  IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
  ```

## Task 2 hints: Deploy stage
- Use the **Amazon ECS** deploy action (not CodeDeploy) with your cluster and service.
- Input artifact is the build output containing `imagedefinitions.json` (note the spelling and the `s`).
- The file format is: `[{"name":"<container name>","imageUri":"<image uri>"}]`
- To prove a deployment: change the version text in `app/app.py` (or set `APP_VERSION`), push, then call the ALB URL.

## Task 3 hints: Troubleshooting order
Work from the outside in and write down what you see at each step.
1. Which pipeline stage is red? Click "Details".
2. Build failure: open the CodeBuild log (CloudWatch Logs). Read the first error, not the last.
3. Deploy failure or tasks restarting: ECS service, Events tab, then Tasks, then stopped reason.
4. Unhealthy targets: EC2, Target Groups, Health checks. Compare port and path with the app (`8080`, `/health`).
5. Change one thing at a time and redeploy.

| Symptom | Likely area |
|---|---|
| AccessDenied on ecr or s3 | IAM role of CodeBuild or CodePipeline |
| "no space left" or docker daemon error | CodeBuild not in privileged mode |
| Deploy: container not found in task definition | Name in imagedefinitions.json vs task definition |
| Tasks start then stop; targets unhealthy | Port or health-check path mismatch |
| Pipeline did not start after push | Source trigger or branch name |

## Task 4 hints: Pick three
- Secrets: store in Secrets Manager or SSM Parameter Store, reference with `env.secrets-manager` or `env.parameter-store` in buildspec.
- Least privilege: replace wildcard actions and resources; restrict ECR actions to the one repository.
- Manual approval: add an Approval action before Deploy.
- Circuit breaker: enable deployment circuit breaker with rollback on the ECS service.
- Scan: enable scan on push on the ECR repository and show findings.

## Task 5: Prompt log template
```
Prompt 1: <what you asked>
What was right: <...>
What was wrong or missing: <...>
What I changed: <...>
```
Do not paste access keys, secrets or your account ID into AI tools.

## Task 6 (challenge) starting points
- A, blue/green: you need two target groups, an `appspec.yaml` for ECS, a CodeDeploy application and a CloudWatch alarm on 5XX or unhealthy hosts.
- B, security gate: ECR scan findings can be read with `aws ecr describe-image-scan-findings`; fail the build if the HIGH or CRITICAL count is above 0.
- C, pipeline as code: start with one stack for the build project and roles, then add the pipeline.
- D, promote same image: use the image digest (`sha256:...`) from the build, not a rebuilt tag.

## Evidence checklist
- [ ] Final buildspec.yml
- [ ] Pipeline screenshot, all stages green
- [ ] ECR screenshot showing the commit-SHA tag
- [ ] ALB URL showing new version
- [ ] Root cause note for Task 3 (error message, where found, fix)
- [ ] Task 4 items (3) with one-line justification each
- [ ] Prompt log (3 prompts)
- [ ] Task 6 evidence (if attempted), including one failure case
