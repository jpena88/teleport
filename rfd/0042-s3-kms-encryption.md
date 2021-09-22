# ---
authors: Joel Wejdenstål (jwejdenstal@goteleport.com)
state: draft
---

# RFD 42 - S3 KMS Encryption

## What

Allow users to configure a custom AWS KMS Customer Managed Key for encrypting
objects Teleport stores in S3.

## Why

Encrypting objects in S3 with a custom CMK allows for additional security as it
can be used to restrict access to sensitive data like session recordings
to those with read privileges to S3 buckets storing said sensitive data.

## Details

As per the current scheme for configuring storage backends like S3 an optional URL
parameter will be added to the S3 storage url with the name of `sse_kms_key`.
This URL parameter can be specified with the value of the KMS ID of the desired
custom key. When configured all objects uploaded will be encrypted with that key.

The configured KMS CMK needs to have a standard spec configuration and must be symmetric.

Below template KMS policies are provided for restricting access to
KMS keys and the needed permissions. The policies are to be filled in with
the IAM user the authentication nodes authenticate as.

### Encryption/Decryption

This policy allows an IAM user to encrypt and decryption objects.
This allows a cluster auth to write and play back session recordings.

Replace `[iam-key-admin-arn]` with the IAM ARN of the user(s) that should have
administrative key access and `[auth-node-iam-arn]` with the IAM ARN
of the user the Teleport auth nodes are using.

```json
{
  "Id": "Teleport Encryption and Decryption",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Teleport CMK Admin",
      "Effect": "Allow",
      "Principal": {
        "AWS": "[iam-key-admin-arn]"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Teleport CMK Auth",
      "Effect": "Allow",
      "Principal": {
        "AWS": "[auth-node-iam-arn]"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

### Encryption/Decryption with separate clusters

This policy allows specifying separate IAM users for encryption and decrtion.
This can be used to set up a multi cluster configuration where the main cluster
cannot play back session recordings but only write them.
A separate cluster authenticating as a different IAM user with decryption access
can be used for playing back the session recordings.

Replace `[iam-key-admin-arn]` with the IAM ARN of the user(s) that should have
administrative key access, `[iam-node-write-arn]` with the IAM ARN of the user the
main write-only cluster auth nodes are using and `[iam-node-read-arn]` with the
IAM ARN of the user used by the read-only cluster.

```json
{
  "Id": "Teleport Separate Encryption and Decryption",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Teleport CMK Admin",
      "Effect": "Allow",
      "Principal": {
        "AWS": "[iam-key-admin-arn]"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Teleport CMK Auth Encrypt",
      "Effect": "Allow",
      "Principal": {
        "AWS": "[auth-node-write-arn]"
      },
      "Action": [
        "kms:Encrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Teleport CMK Auth Decrypt",
      "Effect": "Allow",
      "Principal": {
        "AWS": "[auth-node-read-arn]"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```