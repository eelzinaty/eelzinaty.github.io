---
author : "Issam Alzinati"
title : "How to securly access AWS ECR from GitHub Actions using GitHub OIDC"
date : "2022-07-31"
description : "How to securly access AWS ECR from GitHub Actions using GitHub OIDC"
tags : [
    "aws",
    "aws iam",
    "control tower",
    "aws organization",
    "github",
    "github actions",
    "ECR",
    "OIDC"
]
---

Companies are shifting from building monolith applications into more modular services where each service might represent a business context. They can even get into a more fine-grained breakdown and use a microservices approach to build their applications. Either way, this increases the use of containers. These containers are based on docker images that hold the application logic and dependencies. These images are built with every codebase pushed to the code repository and pushed to a container registry, so it can be later deployed to the corresponding container orchestrator.

I did some work where I were using GitHub to host the code, GitHub Actions for CI/CD, and AWS ECR as the container registry. I take security seriously and make sure I cover it in all aspects. One of the things I hate is long-living credentials. Using short-living credentials with GitHub Actions wasn’t doable till they announced the support of [OpenID Connect in Oct 2021](https://github.blog/changelog/2021-10-27-github-actions-secure-cloud-deployments-with-openid-connect/).

As with any OIDC, to use it to access AWS resources you need to do the following:
1. Create an OIDC provider entity in AWS IAM as an external identity provider (IdP).
2. Create IAM Role with a `Principle` that identifies the created OIDC and authenticates your GitHub repo.
3. Create IAM Policy that gives the needed permissions for the IAM Role.


## Define GitHub OIDC and IAM Resources

I'm going to use CloudFormation to define the AWS resources.

### 1. Create an OIDC provider entity in AWS IAM as an external identity provider (IdP).

AWS IAM service has a resource called [OIDCProvider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html). Also, CloudFormation [supports this resource](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-oidcprovider.html), Great! Now we need to get the GitHub OIDC information, The URL of the OIDC identity provider(idP), and a thumbprint of the idP certificate. 
For the idP URL, you can find the information in [GitHub Actions documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#adding-the-identity-provider-to-aws). The URL is `https://token.actions.githubusercontent.com`.
For the idP certificate thumbprint, [which is a signature for the CA's certificate that was used to issue the certificate for the OIDC-compatible idP](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html), you can obtain it using the following commands([long live stack overflow](https://stackoverflow.com/questions/69247498/how-can-i-calculate-the-thumbprint-of-an-openid-connect-server)):
```shell
% HOST=$(curl https://vstoken.actions.githubusercontent.com/.well-known/openid-configuration \
| jq -r '.jwks_uri | split("/")[2]')
% echo | openssl s_client -servername $HOST -showcerts -connect $HOST:443 2> /dev/null \
| sed -n -e '/BEGIN/h' -e '/BEGIN/,/END/H' -e '$x' -e '$p' | tail +2 \
| openssl x509 -fingerprint -noout \
| sed -e "s/.*=//" -e "s/://g" \
| tr "ABCDEF" "abcdef"
6938fd4d98bab03faadb97b34396831e3780aea1
```

We got all the information we need, so here is the CloudFormation resource definition:
```yaml
GithubOidc:
    Type: 'AWS::IAM::OIDCProvider'
    Properties:
      Url: 'https://token.actions.githubusercontent.com' # GitHub OIDCProvider URL
      ClientIdList:
        - sts.amazonaws.com
      ThumbprintList:
        - 6938fd4d98bab03faadb97b34396831e3780aea1 # GitHub OIDCProvider thumbprint 
```

### 2. Create IAM Role with a `Principle` that identifies the created OIDC and authenticates your GitHub repo.

For the IAM Role, it will be straightforward. We need to allow a principle(GitHub OIDC) to assume the IAM role and get [a WebIdentityToken](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html). Also, we will need to limit the permission to a specific GitHub organization.

```yaml
Role:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName: GithubOIDCIAMRole
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Action: 'sts:AssumeRoleWithWebIdentity' # Action to get a WebIdentityToken
            Principal:
              Federated:
                - !Ref GithubOidc # AWS OIDCProvider resource created earlier
            Condition:
              StringLike:
                'token.actions.githubusercontent.com:sub': !Sub 'repo:GitHubOrgName/*' # Allow any repo in under your GitHub Organization to assume this IAM Role
```

### 3. Create IAM Policy that gives the needed permissions for the IAM Role.

Last part is to create an IAM policy that allows your GitHub Actions CI/CD to interact with AWS ECR.

```yaml
Policy:
    Type: 'AWS::IAM::Policy'
    Properties:
      PolicyName: AllowPushImagesToECR
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow # Retrieves an authorization token to login into ECR
            Action: 'ecr:GetAuthorizationToken'
            Resource: '*'
          - Effect: Allow # Get container image information from ECR
            Action:
              - 'ecr:GetDownloadUrlForLayer'
              - 'ecr:BatchGetImage'
              - 'ecr:BatchCheckLayerAvailability'
            Resource: !Sub 'arn:aws:ecr:${AWS::Region}:${AWS::AccountId}:repository/*'
          - Effect: Allow # upload container image to ECR
            Action:
              - 'ecr:CompleteLayerUpload'
              - 'ecr:UploadLayerPart'
              - 'ecr:InitiateLayerUpload'
              - 'ecr:PutImage'
            Resource: !Sub 'arn:aws:ecr:${AWS::Region}:${AWS::AccountId}:repository/*'
      Roles:
        - !Ref Role # Reference to the AWS IAM role created in the previous step
```

This is the basic setup, [check the complete file](https://gist.github.com/eelzinaty/ab5e1b8b4aa670f223ba8e6e4ed5927f). Next I will show how to deploy and to use these resources with GitHub Actions.

## Deploy AWS Resources then Using Them with GitHub Actions

### Deploy CloudFormation Stack to create AWS Resources
This step can be done in different ways. The easiest one is to open the AWS Console and go to [CloudFormation Create Stack](https://eu-central-1.console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/create/template) and use the `Upload a template file` option. I'm going to use the AWS CLI because it is better for automation if you want to! Here is the command:
```shell
$ aws cloudformation deploy --template-file define_github_oidc_with_iam_role.yaml --stack-name github-oidc --parameter-overrides GitHubOrg=name-of-your-github-org
```
Here is [a link for the complete CloudFormation stack](https://gist.github.com/eelzinaty/ab5e1b8b4aa670f223ba8e6e4ed5927f).

You can verify the resources by visiting the IAM Console page and navigate to [Identity Providers](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/identity_providers). Then Check the [IAM Roles](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/roles) and the [IAM Policies](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/policies) for the new IAM Role and Policy we created via CloudFormation.

### Set up the GitHub Actions Workflows to Use GitHub OIDC to Access AWS Resources.

I prefer to use a parametrized GitHub workflows whenever it is possible. This helps me to abstract the workflow and use it with different repos without any changes. For that I use GitHub Actions secrets and configure the secrets across all the repos from the GitHub Organization settings. From settings navigate to `Security -> Secrets -> Actions`.

![GitHub Secrets for Actions](/images/post/github-oidc-with-aws-orgnization/github-actions-secrets.png)

Here are the secrets I usually add to all my workflows:
1. `AWS_DEFAULT_REGION`, this is the region where your workload is located. Example, `us-east-1`.
2. `AWS_ACCOUNT_ID`, This is your AWS Account ID, `1234567890`
3. `AWS_ROLE_TO_PUSH_IMG`, This is the IAM Role name, `GithubOIDCIAMRole`.

After setting the above secrets, we can build the GitHub Actions Workflow. As with any workflow, it consists of one job or more. In the job you are going to use to build and push your container image to ECR, you will need to define the following:
1. For the job permissions, you need to add `id-token: write`. This permission is needed to interact with GitHub's OIDC Token endpoint.
2. You need a step to generate the AWS credentials.
3. You need a step to log in to the ECR repo
4. Toy need a step to build and push the container image to ECR.

#### 1. Define the workflow and add the job permission.
Let's assume that this workflow will be triggered when the code is `merged` to the `main` branch. And it will have only one job, called `build-push-image`. Here is the skeleton of this workflow:

```yaml
name: Deploy 🚀
  
on:
  push:
    branches:
      - main # Set a branch to deploy

jobs:
  build-push-image:
    runs-on: ubuntu-latest
    # These permissions are needed to interact with GitHub's OIDC Token endpoint.
    permissions:
      id-token: write
      contents: read
      actions: read

    steps:
    # Here is where all the steps will be added
```

#### 2. Generate the AWS credentials
This is where we are going to use AWS STS service to authenticate with GitHub OIDC to generate short living credentials for the IAM Role we created earlier. I will use [the official AWS GitHub action](https://github.com/aws-actions/configure-aws-credentials), `configure-aws-credentials`, to generate the credentials.

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    role-to-assume: 'arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ secrets.AWS_ROLE_TO_PUSH_IMG }}'
    aws-region: ${{ secrets.AWS_DEFAULT_REGION }}
```

#### 3. Log In to the ECR repo
As with any remote and private container repo, you need to log in before you push an image. To do that, I'm going to use [the AWS official GitHub Action](https://github.com/aws-actions/amazon-ecr-login),`amazon-ecr-login`, to log in to ECR.

```yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v1
```

#### 4. Build and Push the Container Image
Finally, I will build and push the image using the regular docker commands, `build` and `push`.

```yaml
- name: Build, tag, and push image to Amazon ECR
  env:
    ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    ECR_REPOSITORY: test-service # the ECR repo name
    IMAGE_TAG: test # The image tag, it could be the commit SHA: ${{ github.sha }}
  run: |
    docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
    docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

And that was it! Now whenever a new code is pushed to the `main` branch a workflow will be triggered, short living token will be generated, and a container image will be built and deployed to ECR. [Here is a full version of the workflow file](https://gist.github.com/eelzinaty/4606aa792ada38f73209a220d68c0769).
