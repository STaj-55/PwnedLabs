# PwnedLabs - Identify the AWS Account ID from a Public S3 Bucket

In this lab, we will be learning how we can find the AWS Account ID from Public S3 Buckets. Finding something as crucial as the Account ID can allow us to enumerate the user and find more services we can exploit. Attackers could utilize this info to verify if an IAM user or role exists, which can allow them to compile list of targets within that account. Another weak point are public EBS and RDS snapshots from that Account that we can potentially find with their account ID.

Step 1: Assuming the initial credentials, and setting up VPN.
- When starting this lab, make sure to run the vpn associated to it. This can be done with the command:
```
sudo openvpn --config "labvpn.ovpn"
```
_From my experience with this lab, use a fresh vpn each time, it'll make things easier_

- After starting the lab we can then set our credentials using the AWS CLI.
```
aws configure
```

- We can make sure that our account is set by verifying with get-caller-identity.
```
aws sts get-caller-identity

{
  "UserId": "AIDAWHEOTHRF62U7I6AWZ",
  "Account": "427648302155",
  "Arn": "arn:aws:iam::427648302155:user/s3user"
}
```

Step 2: Enumeration
- Now that we have assumed this new user, let's go ahead and scan our target IP. We can first try an **nmap** scan to see what ports are open, to better understand our target.
```
nmap -Pn -p 80 54.204.171.32

Starting Nmap 7.93 ( https://nmap.org ) at 2025-12-25 22:44 GMT
Nmap scan report for ec2-54-204-171-32.compute-1.amazonaws.com (54.204.171.32)
Host is up.

PORT   STATE    SERVICE
80/tcp filtered http

Nmap done: 1 IP address (1 host up) scanned in 2.04 seconds
```
_Now what I noticed from the writeup is that it only had the option -Pn but it didn't show port 80. I added -p to get the right response_

- Here we see a webpage which is our primary target for this lab. There's only really pictures on the site so let's check out the source code to see where its getting it from.

<img width="1901" height="896" alt="image" src="https://github.com/user-attachments/assets/3c9e89b5-3d53-4077-93eb-2e63bf548770" />

<img width="766" height="468" alt="image" src="https://github.com/user-attachments/assets/3bf1cae1-9a64-44f2-83a3-328e0eecd604" />

- Looking at the sourc code, we see that this site is hosted via S3 bucket. this is a fairly common practice especially for static sites. Let's see if we can find out more.

<img width="1005" height="879" alt="image" src="https://github.com/user-attachments/assets/85cfb3c1-5cbf-4de6-b87f-d576a9a483b3" />

- After trying to view the url in our browser, we see a simple directory for this but nothing crazy. However knowing the exact bucket name itself, we can attempt to find the ID of the AWS Account it's hosted in.

_Now there are two routes you can do for this but at the moment I am following the route of the user the premade account, this was part of the aws configure in the beginning of the lab. If you want you can create your own S3 user to assume roles in instances like these._

- We'll use the ARN of the role we have with the command s3-account-search to find the account ID of the bucket.
```
s3-account-search arn:aws:iam::427648302155:role/LeakyBucket mega-big-tech
Starting search (this can take a while)
found: 1
found: 10
found: 107
found: 1075
found: 10751
found: 107513
found: 1075135
found: 10751350
found: 107513503
found: 1075135037
found: 10751350379
found: 107513503799
```

- Success! We have obtained the AccountID of _107513503799_. With this we have completed this lab! We have the flag needed to complete this lab. However if you wan an extra challenge, feel free to follow me in our investigation to find extra challenge.

## Extra Challenge

- We can use our newly found AccountID to see if they have any public snapshots we can replicate to gather more info. However we're restricted at this point because snapshots are region-specific. We need to figure out what region that account user typically works in.
  
- Let's try to find out with cURL:
```
curl -I https://mega-big-tech.s3.amazonaws.com
HTTP/1.1 200 OK
x-amz-id-2: +OOIBxDj6c4K78/cFqHMmwu4f7MYzwGYGEQMYYDS3pgoVuQdcCzxb7uhnwnkksnq5y1zWlRTgeqwtpyd4t4/UXzxV3HIKcYI9wQfgQ2FcEk=
x-amz-request-id: 8ETCQT1WTAD02WT6
Date: Fri, 26 Dec 2025 19:51:38 GMT
x-amz-bucket-region: us-east-1
x-amz-access-point-alias: false
x-amz-bucket-arn: arn:aws:s3:::mega-big-tech
Content-Type: application/xml
Transfer-Encoding: chunked
Server: AmazonS3
```

- As we can see from the output of the cURL request, we find that they work in us-east-1. Off to North Virginia we go in order to find any snapshots of the user. Once you have logged into your own AWS account, switch your region to us-east-1 and head over to snapshots in EC2. We can switch from Owned by Me to Public Snapshots and use the filter **_Owner = 107513503799_**. We should see a snapshot available after doing so.

<img width="1903" height="747" alt="image" src="https://github.com/user-attachments/assets/d4ea304c-2c13-4671-a5cc-4522915d0d2d" />

- 
