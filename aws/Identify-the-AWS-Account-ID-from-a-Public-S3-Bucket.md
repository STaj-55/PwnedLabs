# PwnedLabs - Identify the AWS Account ID from a Public S3 Bucket

In this lab, we will be learning how we can find the AWS Account ID from Public S3 Buckets. Finding something as crucial as the Account ID can allow us to enumerate the user and find more services we can exploit. Attackers could utilize this info to verify if an IAM user or role exists, which can allow them to compile list of targets within that account. Another weak point are public EBS and RDS snapshots from that Account that we can potentially find with their account ID.

Step 1: Assuming the initial credentials, and setting up VPN.
- When starting this lab, make sure to run the vpn associated to it. This can be done with the command:
```
sudo openvpn --config "labvpn.ovpn"
```

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

- 
