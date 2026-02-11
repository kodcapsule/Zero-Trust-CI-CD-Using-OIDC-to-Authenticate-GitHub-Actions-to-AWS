
# Stop Storing AWS Access Keys in GitHub Secrets — Use OIDC Instead to Authenticate GitHub Actions to AWS
![Stop Storing AWS Access Keys in GitHub Secrets — Use OIDC Instead](/images/cover.png)


## Introduction
If you’re storing AWS access keys in GitHub Secrets, you’re already behind modern cloud security practices.Let’s all be honest to ourselves, we have in one way or the other dump AWS access keys into GitHub Secrets or we might have read somewhere especially on Reddit about someone  dumping AWS access keys into GitHub Secrets. Well, this works, but you’re carrying unnecessary risk.  Access keys are long-lived credentials and don’t rotate automatically, and once leaked, they’re hard to contain. 
 
OpenID Connect (OIDC) offers  a better option for authentication without storing a long-lived credentials in GitHub secrets. In this post, I will walk you through  how  GitHub OIDC actually works, why it is more secure compared to long-lived credentials like access keys and how to integrate AWS with GitHub Actions using OIDC.
If you care about safer pipelines and cleaner CloudOps practices, this is one upgrade you don’t want to skip. 


| Feature           | Access Keys | OIDC        |
| ----------------- | ----------- | ----------- |
| Credential Type   | Long-lived  | Short-lived |
| Rotation Required | Manual      | Automatic   |
| Stored in GitHub  | Yes         | No          |
| Blast Radius      | High        | Low         |



## How GitHub Actions OIDC integrates with AWS to access resources
![How GitHub Actions OIDC integrates with AWS to access resources](/images/archi.gif)

  - **Step 1 Establish trust relationship:** A trust relationship needs to be established between GitHub (the Identity Provider – IdP) and AWS (the Service Provider / Relying Party - RP).This relationship will register GitHub as a trusted OIDC provider and will tell AWS where to   fetch GitHub’s public signing keys and which token claims AWS should trust.
  - **Step 2 Authentication Request:** when a workflow runs, github generates a  signed  JSON Web Token (JWT)  token with identity claims that include, repository name, branch or tag , workflow name, commit SHA, gitHub org etc which proves where the job is running at the moment.
  - **Step 3 Token Validation:** The Relying Party (RP) , AWS,  validates the token using the token signature, issuer,audience and if token claims match IAM trust policy conditions. The checks either fails or succeeds. 
  - **Step 4 Issues Short-Lived Credentials:** If the validation is successful, it allows the workflow to assume an IAM role and temporary credentials are issued via STS.The credentials have an automatic expiration period. 
  - **Step 5 Workflow Uses AWS:** The workflow then executes/access AWS resources  based on actions that are defined on in the IAM role. After the job is completed, the credentials expire  requiring no cleanup. 


## Configuring GitHub Actions OIDC with AWS IAM
### **Step 1:** Create an OIDC provider in your AWS  IAM for GitHub

In this step,  you will set up the OpenID Provider which is GitHub.  We will be using the AWS console but you can also use AWS CLI too if that is your preference.   

1. Login to AWS and open the IAM console and select Identity providers in the left navigation menu
2.  Click  Add provider button and in the provider details enter these details:
      - Provider type: OpenID Connect
      - Provider URL: https://token.actions.githubusercontent.com
      - Audience: sts.amazonaws.com
      - Add tags :optional

![Create an OIDC provider in your AWS  IAM for GitHub](/images/1.png)


### **Step 2:** Create an IAM role 
1. In the Identity providers screen select the provider you just created and  choose the `Assign role` button. Select Create new role and then choose Next.
- **Select Identity provider**
![Select Identity provider](/images/2.png)
- **Select Assign role**
![Select Assign role](/images/4.png)
- **Seclect Create new role**
![CSeclect Create new role](/images/3.png)

2. Select Web identity and provide these details
      - Identity provider: token.actions.githubusercontent.com
      - Audience: sts.amazonaws.com
      - GitHub organization: provide your github organization. 
if you don't have Github organization you can use [Creating a new organization from scratch](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch)  to create one.

- **Seclect Web identity**
![Seclect Web identity](/images/5.png)
- **Web identity detailes**
![Web identity details](/images/6.png)

3. On the Permissions page, search for `AmazonS3FullAccess` and select it  for this demo and select Next to continue

- **S3 Permissions**
![S3 Permissions](/images/7.png)

4.  In the next step enter  a role name, for this demo, `GitHubAction-AssumeRoleWithAction`. You can optionally add a description. Review and click Create role. 
- **Review and create role** 
  ![Review and create role](/images/8.png)


### **Step 3:** Create a trust policy conditions to restrict access by repository, branch, or environment
1. In the IAM console,  select the newly created role, `GitHubAction-AssumeRoleWithAction`, and choose `Trust relationship` tab and select Edit trust policy. 
2. Copy and paste this policy in editor and save the changes. Make sure to replace `<AWS_ACCOUNT_ID>` with your AWS account ID, `<GITHUB_USER_NAME>` with your Github username and `<REPO_NAME>` with your repository name.
```bash
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
				},
				"StringLike": {
					"token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO_NAME>/ref:refs/heads/<BRANCH>"
				}
			}
		}
	]
}
```

### Understanding OIDC Token Claims (Advanced but Important)

When GitHub issues the OIDC token, it includes several claims that AWS evaluates in the trust policy. The most important claim is the `sub (subject)`. The `sub` claim typically follows this format:

```bash
repo:OWNER/REPO:ref:refs/heads/BRANCH
```
example
```bash
repo:my-org/my-repo:ref:refs/heads/main
```
This means that the  workflow is running in `my-org/my-repo` and it was triggered from the main branch. Restricting by  repository name means that any  branch in that repo can assume the role, but if you restrict by branch  only workflows from that branch can assume the role. This is best security parctice because it prevents:

- Malicious pull requests
- Feature branches gaining production access and 
- Unauthorized role assumption

This is where OIDC becomes truly powerful — it allows fine-grained, identity-based authorization instead of relying on static secrets.

### **Step 4:Optional:** 
You can attach more permissions AWS policies, either AWS  or custom policies, (use the principle of least privilege) to the IAM role defining what AWS actions are allowed when using the role.  

### **Step 5:**  Add the IAM Role  ARN  to GitHub repository as  secrets
- **1.** In your Github account create a repo or select an existing repo that you want to use. The repo i am using is  "Zero-Trust CI/CD: Using OIDC to Authenticate GitHub Actions to AWS".


- **2.** Select the repo settings  and  , in the left menu  select Secrets and variables. Choose Actions in the sub menu and click on  `New repositry secret` button. Add these details
    - Name: AWS_OIDC_ROLE_ARN
    - Secret: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<ROLE_NAME>. 
  *NOTE* copy the role ARN and add it as a secret. You are now ready to use it in your workflows.
![Add role ARN as github secret step 1](/images/10.png)
![Add role ARN as github secret step 2](/images/11.png)

### **Step 6:**  Create a  workflow to use the role ARN and specify the AWS region
In this next step, we will   validate the integration of OIDC with AWS. To do this we will create a workflow. Create this workflow 

`.github/workflows/create-s3-bucket.yml`

```bash
name: Create S3 Bucket with OIDC

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
  pull-requests: write
  security-events: write

jobs:
  create-bucket:
    runs-on: ubuntu-latest

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v5.1.0
        with:
          role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN}}
          role-session-name: GitHubAction-AssumeRoleWithAction
          aws-region: us-east-1


      - name: Create S3 bucket
        run: |
          aws s3api create-bucket \
            --bucket just-another-new-bucket-124 \
            --region us-east-1
```
Test the  OIDC Connection  by running the workflow.

**Workflow ran successfully**.
![Workflow success 1 ](/images/workflow.png)

**S3 bucket created** 
![Workflow success 2 ](/images/bucket%20created.png )

## Conclusion
Building a secure CI/CD pipeline forms a crucial part of modern software development. Storing long-lived credentials in CI/CD pipelines is a bad security practice that is still being used by some teams. OIDC offers a better alternative of enhancing your workflows using  identity-based and  short-lived credentials. This approach reduces your attack surface and simplifies credentials management. your pipelines still rely on stored AWS Access keys, the question is no longer “Why switch to OIDC?”
It’s “Why haven’t you switched yet?”. Modern CloudOps is identity-driven. If you're still using static credentials in 2026, it's time to upgrade your pipeline security posture.I'd like to find out from you on how you manage AWS credentials using GitHub Actions. Have you tried the OIDC approach? Please let me know in the comments.