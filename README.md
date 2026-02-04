
# Stop Storing AWS Access Keys in GitHub Secrets — Use OIDC Instead to Authenticate GitHub Actions to AWS
![Stop Storing AWS Access Keys in GitHub Secrets — Use OIDC Instead](/images/cover.png)


## Introduction
Let’s all be honest to ourselfs , we have in one way or the other dump AWS access keys into GitHub Secrets or we might have read somewhere espercially Redit about someone using dump AWS access keys into GitHub Secrets. Well this works but you are you’re carrying unnecessary risk.  Access keys are long-lived and don’t rotate automatically  and once leaked, they’re hard to contain. 
 
OpenID Connect (OIDC) offers  bettter option of authentication without storing a long-live credentials in GitHub secretes. In this  post i will walk you through how  GitHub OIDC actually works, why is is more secure compared to long-lived crdentials like acess keys and how to integrate AWS with GitHub actions  OIDC with.  
If you care about safer pipelines and cleaner CloudOps practices, this is one upgrade you don’t want to skip. 


## How GitHub Actions OIDC integrates with AWS to access resources
![How GitHub Actions OIDC integrates with AWS to access resources](/images/cover.png)

- **Step 1 Extablish trust relatioship:** A trust relationship needs to be established between Github,the  Identity Provider (IdP) and  AWS the OpenID Provider (OP).This relationship will register GitHub as a trusted OIDC provider and will tell AWS where to   fetch GitHub’s public signing keys and which token claims AWS should trust.
- **Step 2 Authentication Request:** when a workflow runs, github generates a  signed  JSON Web Token (JWT)  token with identity claims that include, repository name, branch or tag , workflow name, commit SHA, gitHub org etc which proves where the job is running at the moment.
- **Step 3 Token Validation:** The Relying Party (RP) , AWS validates the token using the token signature, issuer,audiance and if token claims match IAM trust policy conditions. The checks either fails or succeeds. 
- **Step 4 Issues Short-Lived Credentials:** If the validatons is successfull allows the workflow to asume an IAM role and temporary credentials are issued via STS.The credentials have an authomatic experations period. 
- **Step 5 Workflow Uses AWS:** The workflow then executes/access AWS resources  based on actions that are defined on in the IAM role. After the job is completed the credentials die with it requiring no cleanup. 


## Configuring GitHub Actions OIDC with AWS IAM
### **Step 1:** Create an OIDC provider in your AWS  IAM for GitHub

In this step,  you will setup the OpenID Provider which is GitHub to AWS.  We will be using the AWS console but you can also use AWS CLI too if that is your preference.   

1. Login to AWS and open the IAM console and select   Identity providers in the left navigation menu
2.  Click  Add provider buttom and in the provider details enter these details
   Provider type: OpenID Connect
   Provider URL: https://token.actions.githubusercontent.com
   Audience: sts.amazonaws.com
   Add tags :optional

### **Step 2:** Create an IAM role with a trust policy that allows GitHub Actions to assume the role
1. In the Identity providers screen select the provider you just create and , choose the Assign role button. Seclect Create new role and then choose Next.
2. Select Web identity and provide these details
       Identity provider: 
       Audience: sts.amazonaws.com
       GitHub organization: provide your github organiazation. 
if you don't have Github orgization you can (Creating a new organization from scratch)follow https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch to create one.

3. On the Permissions page, search for AmazonS3FullAccess and select it  for this demo and select next to continue
4.  In the next step enter  a role name, for this demo, enter GitHubAction-AssumeRoleWithAction. You can optionally add a description. Review and click create role. 
  
### **Step 3:** Create a trust policy conditions to restrict access by repository, branch, or environment
1. In the IAM console,  select the newly created role and choose Trust relationship tab and select edit trust policy.
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
					"token.actions.githubusercontent.com:sub": "repo:kodcapsule/*"
				}
			}
		}
	]
}
```
save the changes. 

### **Step 4:Optional:** You can attach more permissions policies to the IAM role defining what AWS actions are allowed using the  least privilege principle.  

### **Step 5:**  Add the IAM Role  ARN  to GitHub repository as  secrets
1. In your Github Account reate a repo or select and existing repo that you want to us . The repo i am using is  "Zero-Trust CI/CD: Using OIDC to Authenticate GitHub Actions to AWS".

2. Select the repo settings  and  in the left menu and  select Secretes and variables. Choose Actions in the sub menu and click on the create new repositoy secret buttom. Add these details
Name: AWS_OIDC_ROLE_ARN
Secret: arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubAction-AssumeRoleWithAction. copy the role ARN and add it as a secrete. Your are now ready to use in in your workflows

### **Step 6:**  Create a  workflow to use the role ARN and specify the AWS region
In this next step, we will   validate the integration of OIDC with AWS. To do this we will create a workflow. Create this workflow 

.github/workflows/create-s3-bucket.yml

```bash
name: Create S3 Bucket with OIDC

on:
  workflow_dispatch:

permissions:
  id-token: write   # REQUIRED for OIDC
  contents: read

jobs:
  create-bucket:
    runs-on: ubuntu-latest

    steps:
      - name: Configure AWS credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN }}
          aws-region: us-east-1

      - name: Create S3 bucket
        run: |
          aws s3api create-bucket \
            --just-another-new-bucket-124 \
            --region us-east-1
```
Test the  OIDC Connection  by running the workflow
### **Step 7:**   Audit the role usage: Query CloudTrail logs

## Conclusion
Building a secure CI/CD pipeline forms a crucial part  of modern software development. Storing Long-lived credentials in CI/CD pipelines are bad security practices that is still been used by some teams. OIDC offers a better alternative of enhancing your workflows using  identity-based and  short-lived credentails. This approach reduce your attack surface and simplifies credentials management. If your pipelines still relies on stored AWS Accesss keys, the question is no longer “Why switch to OIDC?”
It’s “Why haven’t you switch yet?”. 

I'd like to find out from you on how you manage AWS credentials using Github Actions. Have you tried the OIDC approach? Please let me know in the comments.