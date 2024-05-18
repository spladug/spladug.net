+++
title = "Restricting certbot IAM permissions for AWS Route53"
description = "The default IAM policy is more lax than it needs to be. We can restrict what certbot can do considerably."
date = 2023-04-??
+++

https://certbot-dns-route53.readthedocs.io/en/stable/

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:GetChange"
            ],
            "Resource": "*"
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": "route53:ChangeResourceRecordSets",
            "Resource": "arn:aws:route53:::hostedzone/YOURHOSTEDZONEID",
            "Condition": {
                "ForAllValues:StringEquals": {
                    "route53:ChangeResourceRecordSetsRecordTypes": "TXT"
                },
                "ForAllValues:StringLike": {
                    "route53:ChangeResourceRecordSetsNormalizedRecordNames": "_acme-challenge.*"
                }
            }
        }
    ]
}
```
