## IAM & AWS CLI

**IAM**
- `IAM` stands for `Identity And Access Management` , Global Service
- `Root account`: created by default, shouldn't be used or shared
- `Users`: people within your organization, and can be grouped
  - `Groups` only contain `Users`, not other group 
  - A user can belong to multiple groups

- `Users or Groups` can be assigned `JSON` documents called policies

```json
{
  "Version": "2026-04-10",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:Describe",
      "Resource": "*"
    }
  ]
}
```
- In AWS apply the `least privilege principle (최소 권한 원칙)`
  - don't give more permissions than a user needs 