# AWS Cloud Labs

~ on ☁️  (us-east-1) ➜  showcreds
╒══════════════════════╤═════════════════════════════════════════════════════════════════════╕
│ Name                 │ Value                                                               │
╞══════════════════════╪═════════════════════════════════════════════════════════════════════╡
│ AWS Console URL      │ https://854993966332.signin.aws.amazon.com/console?region=us-east-1 │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ AWS User Name        │ kk_labs_user_561418                                                 │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ AWS Password         │ N8wuDVc3%y!!                                                        │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ AWS Session End Time │ 2026-02-19T01:28:59Z                                                │
╘══════════════════════╧═════════════════════════════════════════════════════════════════════╛

~ on ☁️  (us-east-1) ➜  cat <<EOF > policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeImages",
                "ec2:DescribeSnapshots"
            ],
            "Resource": "*"
        }
    ]
}
EOF

~ on ☁️  (us-east-1) ➜  aws iam create-policy \
    --policy-name iampolicy_kirsty \
    --policy-document file://policy.json
{
    "Policy": {
        "PolicyName": "iampolicy_kirsty",
        "PolicyId": "ANPA4OEM4ST6L6BHWMJRO",
        "Arn": "arn:aws:iam::854993966332:policy/iampolicy_kirsty",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-02-19T00:36:37Z",
        "UpdateDate": "2026-02-19T00:36:37Z"
    }
}

~ on ☁️  (us-east-1) ➜  aws iam list-policies --scope Local --query 'Policies[?PolicyName==`iampolicy_kirsty`].Arn' --output text
arn:aws:iam::854993966332:policy/iampolicy_kirsty

~ on ☁️  (us-east-1) ➜  

<img width="1029" height="685" alt="image" src="https://github.com/user-attachments/assets/7a75c81b-dd0c-438b-97b0-7b143515fd8f" />
<img width="1034" height="650" alt="image" src="https://github.com/user-attachments/assets/5120b923-cb2b-4a57-85ee-4945fb56c245" />
<img width="1032" height="770" alt="image" src="https://github.com/user-attachments/assets/8cb66d29-4ae3-46ab-81fb-b02f271fe667" />
<img width="1033" height="858" alt="image" src="https://github.com/user-attachments/assets/092cc3e3-20f5-49e3-be83-965eaea2cf0e" />

