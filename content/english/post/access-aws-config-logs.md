+++
author = "Issam Alzinati"
title = "Access AWS Config Logs with AWS SSO User"
date = "2022-07-25"
description = "How to access encrypted AWS Config Logs with AWS SSO User from Shared Archive Account"
tags = [
    "aws",
    "aws config",
    "control tower",
    "aws sso",
    "aws iam",
]
+++

I recently noticed a sudden increase in the AWS Config cost in the production environment. Because we are using AWS Control Tower to govern and manage the company's AWS accounts, we get out-of-the-box a dedicated AWS account where all logs are shipped to it, and part of these logs is the AWS Config logs. The name of this account is `Log Archive`.

The AWS Config logs are saved as files in S3 bucket, and these files are partitioned in a way that allow Athena to query them. After setting up the Athena table and write the first query, I got an error complaining about the lack of permissions to read the S3 bucket data!

![Athena Denied Query](/images/post/access-aws-config-logs/athena-denied-query.png)

I was logged in the account using the `AWSAdministratorAccess` SSO Permission Set, which have administrator access to the account. I went to check the S3 bucket that have the files to see of there an explicit permission that denies access to the bucket data, but found nothing. Then I went to check one of the files saved in the bucket and tried to download it but get another access denied error!

![Log download access denied](/images/post/access-aws-config-logs/logs-download-access-denied.png)

I thought that could be a file permission that prevent me from accessing the file, so I checked the permission tab, yet nothing is set to deny the access to the file.

Checking the file information and scrolling down in the page I found that `Server-Side Encryption` is enabled and KMS key lives in the AWS Master account.

![S3 KMS Key](/images/post/access-aws-config-logs/aws-config-s3-kms-key.png)


It turns out that AWS Config is configured to send the logs to S3 and encrypt then using a KMS key that is managed in the AWS Master account.
```yaml
# Part of the cloudformation stack the is deployed in the all accounts managed by AWs Control Twoer.
Resources:
    ConfigDeliveryChannel:
        Type: AWS::Config::DeliveryChannel
        Properties:
        Name: !Sub ${ManagedResourcePrefix}-BaselineConfigDeliveryChannel
        ConfigSnapshotDeliveryProperties:
            DeliveryFrequency: !FindInMap
            - Settings
            - FrequencyMap
            - !Ref Frequency
        S3BucketName: !Ref AuditBucketName
        S3KeyPrefix: !Ref AWSLogsS3KeyPrefix
        SnsTopicARN: !Sub arn:aws:sns:${AWS::Region}:${SecurityAccountId}:${AllConfigTopicName}
        S3KmsKeyArn: !If
            - IsUsingKmsKey     # is True
            - !Ref KMSKeyArn    # equal to Master KMS key, arn:aws:kms:eu-west-1:zzzzzzzzz:key/528ee8b8-4cc4-410d-9285-97a66ba1c4cf
            - !Ref AWS::NoValue
```

## Creating AWS SSO user with the required permissions to access the encrypted logs
When AWS Athene read the data from S3 bucket, it uses the current user IAM permissions. This means that I need to grant the user I will use permissions to decrypt the Master KMS key. Also, To do that, I'm going create new AWS SSO Permission Set, and call it `AdminSecurityAudit`. Then grant this group a permission to full access the account and decrypt the Master KMS key. Here are the steps:
1. Login to your AWS Master account as an Administrator.
2. Go AWS SSO service and start the [create new permission set wizard](https://us-east-1.console.aws.amazon.com/iamv2/home?region=eu-west-1#/organization/permission-sets/create).
    1. In the `Select permission set type` step choose `Custom permission set`.
    2. In the `Specify policies and permissions boundary` step:
        1. For `AWS managed policies` choose `AdministratorAccess`
        2. For `Inline policy`, you need to define a policy that allows `ecnrypt` action for the Master KMS key
        ```json
        {
            "Statement": [
                {
                    "Sid": "Statement1",
                    "Effect": "Allow",
                    "Action": [
                        "kms:Decrypt",
                        "kms:DescribeKey"
                    ],
                    "Resource": [
                        "arn:aws:kms:eu-west-1:zzzzzzzzz:key/528ee8b8-4cc4-410d-9285-97a66ba1c4cf"
                    ]
                }
            ]
        }
        ```
        3. In the `Specify permission set details` step, fill a permission set name: `AdminSecurityAudit`.
        4. Finally, review the permission set details and click the create button.
3. Now you need to [create AWS SSO Group](https://eu-west-1.console.aws.amazon.com/singlesignon/identity/home?region=eu-west-1#!/groups/create) and add yourself to it.
 
 ![AWS SSO Group](/images/post/access-aws-config-logs/aws-admin-audit-group.png)

4. Then you go [AWS Account](https://us-east-1.console.aws.amazon.com/iamv2/home?region=eu-west-1#/organization/accounts) and choose the `Log Archive` account
5. In the `Log Archive` account page click on `Assign users or groups`

![AWS SSO Group](/images/post/access-aws-config-logs/aws-log-archive-account.png)

6. In the `Assign users or groups` follow the steps:
    1. In the `Groups` tab select `AdminSecurityAudit`
    2. For the permission set, search for `AdminSecurityAudit` and select it.
    3. Review and Submit.
7. The new Group with the corresponding permission set will be deployed to your account.

This is not enough though to allow the user to use the Master KMS key. We need to edit the Master KMS key resource policy.
1. Login to your AWS Master account as an Administrator.
2. Go the [KMS service page](https://eu-west-1.console.aws.amazon.com/kms/home?region=eu-west-1#/kms/keys)
3. Click on the `management` key.
4. Edit the key policy, append the following part. The AWS Principle that we need to use is the name of IAM role for the `AdminSecurityAudit` permission set we just created.
```json
{
    "Sid": "Allow SSO Admin to decrypt logs",
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::xxxxxxxxxx:role/aws-reserved/sso.amazonaws.com/eu-west-1/AWSReservedSSO_AdminSecurityAudit_e73ce8da357r209c"
    },
    "Action": [
        "kms:DescribeKey",
        "kms:Decrypt"
    ],
    "Resource": "*"
}
```
5. Save the changes.


Now, when we log in to the `Log Archive` account using the permission set we just created, we will be able to run the AWS Athena queries against the encrypted log files.

![AWS Successful Athena Query](/images/post/access-aws-config-logs/aws-athena-query.png)
