# aws-dyndns
A minimalistic approach to Dynamic DNS using AWS Route 53.

# Motivation

This whole solution was done in just a few hours, because it offloads most of the work to other SaaS/PaaS:
- you need a hosted DNS zone under AWS Route 53
- interaction with the AWS API is done using their official command-line CLI
- authentication is done by AWS using a restricted IAM user
- your remote IP address is discovered using the free ["ipify" service](https://www.ipify.org/)

Communication is done entirely via HTTPS, and does not require "root" on the Linux machine.

# Prerequisites

This script depends on the [AWS CLI](https://aws.amazon.com/cli/), [dig](https://en.wikipedia.org/wiki/Dig_(command)), and Curl. On a Debian/Ubuntu machine, use the following command to install them:

```bash
sudo apt-get install awscli dnsutils curl
```

# Installation

Besides already having a hosted DNS zone under AWS Route 53, you need to set up the following in the AWS ```IAM console```:
- Create a user.
- Write down the "Access Key ID" and "Secret Access Key" credentials.
- Click on the newly created user to edit its properties. Click ```Inline Policies``` to create one. Use the Policy Generator:
  - **Effect**: Allow
  - **AWS Service**: Amazon Route 53
  - **Actions**: select only "ChangeResourceRecordSets"
  - **Amazon Resource Name (ARN)**: ```arn:aws:route53:::hostedzone/%ID%```
    - You can get the [ID](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/UsingWithIAM.html) of your hosted zone from the list of the "Hosted zones" in your AWS Route 53 service.
    - Example: ```arn:aws:route53:::hostedzone/Z148QEXAMPLE8V```
- Click Next. The final Policy Document would look something like:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1456599587000",
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/Z148QEXAMPLE8V"
            ]
        }
    ]
}
```
- Click "Apply Policy".

On the Linux machine, you need to do the following:
- Install the Prerequisites packages (described above).
- Configure the AWS login credentials by executing:
```bash
aws configure
```
- Run the script manually; here is an example:
```bash
./aws-dyndns test.example.com 5 Z148QEXAMPLE8V 30
```

You should ensure that this script runs all the time. The easiest way would be to add the following to your ```/etc/rc.local``` file, which will start the script in background on system boot:
```bash
/usr/bin/sudo -u $YOUR_USER \
        /home/$YOUR_USER/$PATH_TO_SCRIPT/aws-dyndns \
        test.example.com 5 Z148QEXAMPLE8V 30 2>&1 | logger -t awsdyndns --id=$$ &
```

# Installing aws-dyndns as a systemd service
Running aws-dyndns as a systemd service will start the service boot, manage logging,
execute with the least possible permissions, and relaunch on crash.

```bash
# These commands are relative to the project root. Change directories to get there.
cd aws-dyndns

# This will launch the default editor, so you can set a configuration.
# Because this will run as the unprivileged nobody/nogroup user/group we need to pass the AWS credentials.
editor systemd/aws-dyndns.conf

# The configuration file needs to be readable by root, but we can give it very restrictive permissions to protect our AWS key.
sudo chown root:root aws-dyndns systemd/aws-dyndns.service systemd/aws-dyndns.conf 
sudo chmod 600 systemd/aws-dyndns.conf

# Install the executable, configuration, and service
sudo mv aws-dyndns /usr/local/sbin
sudo mv systemd/aws-dyndns.conf /etc/
sudo mv systemd/aws-dyndns.service /etc/systemd/system

# Reload systemd and then start the service.
sudo systemctl daemon-reload
sudo systemctl start aws-dyndns

# Check the log to make sure everything started as expected.
tail -n 20 /var/log/syslog
```

# Sample log output

```
[Sun Feb 28 11:06:44 EET 2016]: No IP change detected: 190.204.20.169
[Sun Feb 28 11:07:16 EET 2016]: Updating IP to: 181.251.13.59 (test.example.com); OLD=190.204.20.169
{
    "ChangeInfo": {
        "Comment": "DynDNS update",
        "Status": "PENDING",
        "Id": "/change/C459FZQZ7R9H4",
        "SubmittedAt": "2016-02-28T09:07:18.616Z"
    }
}
[Sun Feb 28 11:07:18 EET 2016]: Done. Updated IP to: 181.251.13.59 (test.example.com)
[Sun Feb 28 11:07:50 EET 2016]: No IP change detected: 181.251.13.59
...
curl: (28) Resolving timed out after 5517 milliseconds
[Sun Feb 28 11:47:09 EET 2016]: Invalid NEW_IP: 
[Sun Feb 28 11:47:40 EET 2016]: No IP change detected: 181.251.13.59
...
```
