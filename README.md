# HashiCorp - Terraform AWS Cloud Deployment (pre-release)
## HashiCorp Terraform - Infrastructure Management

![GitHub Actions - Terraform](https://github.com/emvaldes/terraform-awscloud/workflows/GitHub%20Actions%20-%20Terraform/badge.svg)

It's imperative that these GitHub Secrets are set:

```bash
AWS_ACCESS_KEYPAIR:     Terraform AWS KeyPair (PEM file).
AWS_ACCESS_KEY_ID:      Terraform AWS Access Key-Id (e.g.: AKIA2...VT7DU).
AWS_DEFAULT_ACCOUNT:    The AWS Account number (e.g.: 123456789012).
AWS_DEFAULT_PROFILE:    The AWS Credentials Default User (e.g.: default).
AWS_DEFAULT_REGION:     The AWS Default Region (e.g.: us-east-1)
AWS_DEPLOY_TERRAFORM:   Enable|Disable (true|false) deploying terraform infrastructure
AWS_DESTROY_TERRAFORM:  Enable|Disable (true|false) destroying terraform infrastructure
AWS_SECRET_ACCESS_KEY:  Terraform AWS Secret Access Key (e.g.: zBqDUNyQ0G...IbVyamSCpe)
DEVOPS_ASSUME_ROLE:     Defines the STS TrustPolicy for the Terraform user.
DEVOPS_TRUST_POLICY:    Defines the STS Assume-Role for the Terraform user.
INSPECT_DEPLOYMENT:     Control-Process to enable auditing infrastructure state.
UPDATE_PYTHON_LATEST:   Enforce the upgrade from the default 2.7 to symlink to the 3.6
UPDATE_SYSTEM_LATEST:   Enforce the upgrade of the Operating System.
```

### The following features described here are not really scalable and will need to be reviewed.

The **AWS_ACCESS_KEYPAIR** is a GitHub Secret used to auto-populate the ***~/access-keypair*** file for post-deployment configurations.

In the event of needing to target a different account, change it in the GitHub Secrets **AWS_DEFAULT_ACCOUNT**. Keep in mind that both **AWS_SECRET_ACCESS_KEY** and **AWS_ACCESS_KEY_ID** are account specific.<br>

There is no need to midify the GitHub Secret **AWS_DEFAULT_PROFILE** as there is only one section defined in the ~/.aws/credentials file. If a specific AWS Region is required, then update the **AWS_DEFAULT_REGION** but keep in mind that any concurrent build will be pre-set.

The logical switch **AWS_DEPLOY_TERRAFORM** is set to enable or disable the deployment of the terraform plan is a safety messure to ensure that a control-mechanism is in place. The same concept applies to **AWS_DESTROY_TERRAFORM** which is set to enable or disable the destruction of the previously deployed terraform infrastructure.

**Note**: In addition to these basic/core requirements, it's important that a key-name **terraform** be created/active in AWS as it's hardcoded in this prototype. I will find a more efficient solution to this.

You must create the following JSON configuration files:

```console
$ cat /tmp/trust-policy.json ;
```
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:user/terraform"
      },
    "Action": "sts:AssumeRole"
  }]
}
```

```console
$ cat /tmp/assume-role.json ;
```
```json
{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:role/{{ DEVOPS_TRUST_POLICY }}"
    }
}
```

```console
$ aws --profile default \
      --region us-east-1 \
      iam create-role \
      --role-name {{ DEVOPS_TRUST_POLICY }} \
      --assume-role-policy-document file:///tmp/trust-policy.json ;
```

```json
{
    "Role": {
        "Path": "/",
        "RoleName": "{{ DEVOPS_TRUST_POLICY }}",
        "RoleId": "***",
        "Arn": "arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:role/{{ DEVOPS_TRUST_POLICY }}",
        "CreateDate": "2020-08-15T19:04:31+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:user/terraform"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```

```console
$ aws --profile default \
      --region us-east-1 \
      iam attach-role-policy 
      --role-name {{ DEVOPS_TRUST_POLICY }} \
      --policy-arn arn:aws:iam::aws:policy/AdministratorAccess ;
```

```console
$ aws --profile default \
      --region us-east-1 \
      iam create-policy \
      --policy-name {{ DEVOPS_ASSUME_ROLE }} \
      --policy-document file:///tmp/assume-role.json ;
```

```json
{
    "Policy": {
        "PolicyName": "{{ DEVOPS_ASSUME_ROLE }}",
        "PolicyId": "***",
        "Arn": "arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:policy/{{ DEVOPS_ASSUME_ROLE }}",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2020-08-15T19:09:56+00:00",
        "UpdateDate": "2020-08-15T19:09:56+00:00"
    }
}
```

```console
$ aws --profile default \
      --region us-east-1 \
      iam attach-user-policy \
      --user-name terraform \
      --policy-arn arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:policy/{{ DEVOPS_ASSUME_ROLE }} ;
```

```console
$ aws --profile default \
      --region us-east-1 \
      sts assume-role \
      --role-arn arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:role/{{ DEVOPS_TRUST_POLICY }} \
      --role-session-name "TerraformPipeline" > /tmp/assume-role-output.txt;
```

```console
$ cat /tmp/assume-role-output.txt ;
```

```json
{
    "Credentials": {
        "AccessKeyId": "***",
        "SecretAccessKey": "***",
        "SessionToken": "IQoJb3JpZ2 ... zMlKV1PVI=",
        "Expiration": "2020-08-16T04:54:47+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "***:TerraformPipeline",
        "Arn": "arn:aws:sts::{{ AWS_DEFAULT_ACCOUNT }}:assumed-role/{{ DEVOPS_TRUST_POLICY }}/TerraformPipeline"
    }
}
```

```shell
declare -a credentials=(
    aws_access_key_id~${AWS_ACCESS_KEY_ID}
    aws_secret_access_key~${AWS_SECRET_ACCESS_KEY}
  );
for credential in ${credentials[@]}; do
  sed -i -e "s|^\(${credential%\~*}\)\( =\)\(.*\)$|\1\2 ${credential#*\~}|g" ${AWS_SHARED_CREDENTIALS_FILE} ;
done;
declare -a session_token=($(
    aws --profile ${AWS_DEFAULT_PROFILE} \
        --region ${AWS_DEFAULT_REGION} \
        sts assume-role \
        --role-arn arn:aws:iam::${AWS_DEFAULT_ACCOUNT}:role/${{ env.AWS_TRUST_POLICY }} \
        --role-session-name "TerraformPipeline" \
        --duration-seconds 300 \
        --query 'Credentials.{aki:AccessKeyId,sak:SecretAccessKey,stk:SessionToken,sts:Expiration}' \
        --output text
     ));
declare -a session_items=(AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_TOKEN_EXPIRES);
counter=0; for xkey in "${session_token[@]}"; do
  eval "export ${session_items[$((counter++))]}=${xkey}";
done;
```

```shell
declare -a credentials=(
    aws_access_key_id~${AWS_ACCESS_KEY_ID}
    aws_secret_access_key~${AWS_SECRET_ACCESS_KEY}
    aws_session_token~${AWS_SESSION_TOKEN}
    x_principal_arn~arn:aws:iam::${AWS_DEFAULT_ACCOUNT}:user/terraform
    x_security_token_expires~${AWS_TOKEN_EXPIRES}
  );
for credential in ${credentials[@]}; do
  sed -i -e "s|^\(${credential%\~*}\)\( =\)\(.*\)$|\1\2 ${credential#*\~}|g" ${AWS_SHARED_CREDENTIALS_FILE} ;
done;
cat ${AWS_SHARED_CREDENTIALS_FILE} ;
```

```cosole
$ aws --profile default \
      --region us-east-1 \
      iam list-users;
```

```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "terraform",
            "UserId": "***",
            "Arn": "arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:user/terraform",
            "CreateDate": "2020-08-15T18:07:01+00:00"
        }
    ]
}
```

Please, make sure your **AWS IAM Policy** allows for something like this and enforce the appropriate ***User Permissions Boundary***:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::terraform-states-{{ AWS_DEFAULT_ACCOUNT }}",
                "arn:aws:s3:::terraform-states-{{ AWS_DEFAULT_ACCOUNT }}/*"
            ]
        }
    ]
}
```

I would also recommend that you append an ***AWS IAM Inline Policy*** to your **terraform** ***AWS IAM User account***:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "StmtCustomIAmPolicy",
            "Effect": "Deny",
            "Action": [
                "iam:AttachRolePolicy",
                "iam:UpdateAccessKey",
                "iam:DetachUserPolicy",
                "iam:CreateLoginProfile",
                "ec2:RequestSpotInstances",
                "organizations:InviteAccountToOrganization",
                "iam:AttachUserPolicy",
                "lightsail:Update*",
                "iam:ChangePassword",
                "iam:DeleteUserPolicy",
                "iam:PutUserPolicy",
                "lightsail:Create*",
                "lambda:CreateFunction",
                "lightsail:DownloadDefaultKeyPair",
                "iam:UpdateUser",
                "organizations:CreateAccount",
                "iam:UpdateAccountPasswordPolicy",
                "iam:CreateUser",
                "lightsail:Delete*",
                "iam:AttachGroupPolicy",
                "ec2:StartInstances",
                "iam:PutUserPermissionsBoundary",
                "iam:PutGroupPolicy",
                "lightsail:Start*",
                "lightsail:GetInstanceAccessDetails",
                "iam:CreateAccessKey",
                "organizations:CreateOrganization"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

I would also setup the target ***AWS S3 Bucket Policy*** based on these standard configurations:

```json
{
    "Version": "2012-10-17",
    "Id": "PolicyTerraformS3Bucket",
    "Statement": [
        {
            "Sid": "StmtTerraformS3Bucket",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::{{ AWS_DEFAULT_ACCOUNT }}:user/terraform"
            },
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::terraform-states-{{ AWS_DEFAULT_ACCOUNT }}/*"
        }
    ]
}
```

Block ALL Bucket public access (bucket settings: ON)
<ol>
<li>Block public access to buckets and objects granted through new access control lists (ACLs)</li>
<li>Block public access to buckets and objects granted through any access control lists (ACLs)</li>
<li>Block public access to buckets and objects granted through new public bucket or access point policies</li>
<li>Block public and cross-account access to buckets and objects through any public bucket or access point policies</li>
</ol>

**Reference**: This project is based on the original training materials from [PluralSight](https://www.pluralsight.com).<br />
[Terraform - Getting Started](https://app.pluralsight.com/library/courses/getting-started-terraform) by [Ned Bellavance](https://app.pluralsight.com/profile/author/edward-bellavance)
