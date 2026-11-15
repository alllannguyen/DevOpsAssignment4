# Resume website deployment

This project deploys a simple resume website to an Amazon EC2 instance. A push to the configured GitHub branch starts a pipeline that packages and deploys the website.

## Architecture

```text
GitHub repository / branch
        │ push
        ▼
AWS CodePipeline: Source (GitHub connection)
        │
        ▼
CodeBuild: package website ──► S3 pipeline artifact bucket
        │
        ▼
CodeDeploy ──► EC2 (Amazon Linux 2023, Apache HTTP server)

Separate private S3 bucket: resume file storage
```

The CloudFormation stack creates EC2, the web security group, IAM roles, two S3 buckets, CodeBuild, CodeDeploy and CodePipeline. HTTP port 80 is public; no SSH port is opened. The stack uses the default VPC, which must have a subnet with public IP assignment enabled and internet access. The resume bucket is private and is separate from pipeline artifacts. Storing a resume file in that bucket does not automatically update the website.

## Setup

1. Authenticate to the intended AWS account with `aws login --profile new-profile-name` and configure a region for the profile, or include `--region REGION` in each command.
2. Commit and push this repository to GitHub. Confirm the branch name, and use it in the stack parameter below.
3. In AWS Developer Tools, create a GitHub connection under **Settings → Connections**. Complete the GitHub authorization in the console and make sure its status is **Available**. Copy its ARN. Creating the connection alone does not complete authorization.
4. Deploy, replacing the connection ARN and branch as needed:

   ```sh
   aws cloudformation deploy --profile new-profile-name \
     --stack-name resume-website \
     --template-file infrastructure.yml \
     --capabilities CAPABILITY_IAM \
     --parameter-overrides GitHubConnectionArn=CONNECTION_ARN GitHubBranch=master
   ```

5. Get the website URL and bucket names:

   ```sh
   aws cloudformation describe-stacks --profile new-profile-name --stack-name resume-website --query 'Stacks[0].Outputs' --output table
   ```

6. Open CodePipeline and wait for all three stages to succeed. The EC2 instance must finish installing Apache and the CodeDeploy agent before deployment can succeed; retry the pipeline if the first deployment starts too soon. Open the website URL after deployment.
7. Optionally upload a resume file to the private bucket, substituting the output name:

   ```sh
   aws s3 cp /path/to/resume.pdf s3://RESUME_BUCKET/resume.pdf --profile new-profile-name
   ```

## Maintenance and demonstration

Edit `index.html`, commit, and push to the configured branch. In the CodePipeline console, show the new source revision moving through Source, Build and Deploy; then refresh the public website to show the changed content. The package contents are specified in `buildspec.yml`; `appspec.yml` and `scripts/start-server.sh` control the CodeDeploy installation. Record a walkthrough, list completed and omitted features, and demonstrate this change for the required presentation. Upload the recording and submit its URL.

Included: GitHub source change detection, packaging, EC2 deployment, private resume storage, infrastructure documentation. Omitted: automated tests, custom monitoring and alarms. AWS services may produce their standard operational logs.

## Troubleshooting

- **Source fails:** verify that the connection is Available, the GitHub app has access to the repository, and the branch name matches exactly.
- **Deploy fails:** wait for EC2 user data to install the CodeDeploy agent, then retry. Check deployment events in CodeDeploy and use Systems Manager Session Manager, if available, to inspect `/var/log/aws/codedeploy-agent/` and `/var/log/cloud-init-output.log`.
- **Website unavailable:** check the stack's public IP output, instance status, public subnet routing and port 80 security group. The instance must have internet connectivity to install packages and the agent.
- **Authentication errors:** renew the profile login, verify the account with `aws sts get-caller-identity --profile new-profile-name`, and check the selected region.

## Cleanup

After the demonstration, download anything you want to keep from the resume bucket. Empty **both** buckets (including any versions, if versioning was enabled), then delete the stack:

```sh
aws s3 rm s3://ARTIFACT_BUCKET --recursive --profile new-profile-name
aws s3 rm s3://RESUME_BUCKET --recursive --profile new-profile-name
aws cloudformation delete-stack --stack-name resume-website --profile new-profile-name
aws cloudformation wait stack-delete-complete --stack-name resume-website --profile new-profile-name
```

Delete the separately created GitHub connection if it is no longer needed. Check for any remaining resources or logs that could incur charges.
