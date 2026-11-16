# Resume website deployment

This project contains a simple resume website for deployment to an EC2 instance using AWS CodePipeline.

## Architecture

```text
GitHub → CodePipeline → CodeBuild (package) → CodeDeploy → EC2 / Apache
                                  ↕
                       S3 pipeline artifact bucket

Separate private S3 bucket → resume file storage
```

## Setup

1. Push this repository to GitHub and authorize a GitHub connection in AWS Developer Tools. Confirm the connection is **Available**.
2. Launch an EC2 instance with a public IP, HTTP port 80 open, Apache installed, and the CodeDeploy agent running. Give the instance an IAM role with access to the pipeline artifact bucket.
3. Create a CodeBuild project with CodePipeline as its source and artifact provider. Enter this build specification in the console:

   ```yaml
   version: 0.2
   phases:
     build:
       commands:
         - echo Packaging website for CodeDeploy
   artifacts:
     files:
       - index.html
       - appspec.yml
       - scripts/**/*
   ```

4. Create a CodeDeploy EC2 application and deployment group targeting the instance tag. `appspec.yml` copies the page to `/var/www/html`; `scripts/start-server.sh` starts Apache.
5. Create a CodePipeline pipeline with GitHub as the source, your CodeBuild project as the build stage, and the CodeDeploy application and deployment group as the deploy stage. Wait for the pipeline to succeed, then open the EC2 public URL.
6. Optionally upload a PDF to the private resume bucket:

   ```sh
   aws s3 cp /path/to/resume.pdf s3://resume-website-resumebucket-pjewhrhvdquj/resume.pdf --profile new-profile-name
   ```

   Uploading the PDF stores it privately; it does not update the website.

## Maintenance and demonstration

Edit `index.html`, commit and push to the configured branch. In the CodePipeline console, show the new revision progressing through Source, Build and Deploy, then refresh the website to show the change. Record a walkthrough, explain completed and omitted features, and demonstrate this process. Upload the recording and submit its URL.

Planned features: GitHub change detection, packaging, EC2 deployment, private resume storage and documentation. Automated tests and custom monitoring/alarms are omitted. AWS services may generate standard operational logs.

## Troubleshooting

- **Source fails:** Verify the connection is Available, GitHub access is authorized, and the branch name matches.
- **Build fails:** Check the inline build specification and CodeBuild logs.
- **Deploy fails:** Check CodeDeploy deployment events, instance tags, IAM permissions, and the CodeDeploy agent logs at `/var/log/aws/codedeploy-agent/`.
- **Website unavailable:** Check the instance public IP, subnet routing, Apache, and port 80 security group.
- **Authentication errors:** Run `aws sts get-caller-identity --profile new-profile-name` to check the account and renew the login if needed.

## Cleanup

When finished, save anything needed from the S3 buckets, then remove the pipeline, CodeBuild project, CodeDeploy application, EC2 instance, buckets, unused IAM roles and connection. Empty buckets, including any versions, before deleting them.

An earlier setup created a CloudFormation stack named `resume-website`. Removing its template from this repository does not delete the stack or its resources. Deleting the stack removes its managed resources once its buckets are emptied, so keep it until you are ready to clean up or replace them.
