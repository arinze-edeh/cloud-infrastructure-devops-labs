# AWS Cloud Labs



~ on ☁️  (us-east-1) ➜  aws sts get-caller-identity
{
    "UserId": "AIDA2QVTQSGV2I63UL6AY",
    "Account": "723004723627",
    "Arn": "arn:aws:iam::723004723627:user/kk_labs_user_926814"
}

~ on ☁️  (us-east-1) ➜  aws iam list-policies --scope Local --query "Policies[?PolicyName=='iampolicy_rose'].Arn" --output text
arn:aws:iam::723004723627:policy/iampolicy_rose

~ on ☁️  (us-east-1) ➜  aws iam attach-user-policy --user-name iamuser_rose --policy-arn arn:aws:iam::723004723627:policy/iampolicy_rose

~ on ☁️  (us-east-1) ➜  aws iam list-attached-user-policies --user-name iamuser_rose
{
    "AttachedPolicies": [
        {
            "PolicyName": "iampolicy_rose",
            "PolicyArn": "arn:aws:iam::723004723627:policy/iampolicy_rose"
        }
    ]
}

~ on ☁️  (us-east-1) ➜  






<img width="1027" height="563" alt="image" src="https://github.com/user-attachments/assets/921dd7be-746b-4d23-b68e-7dc7d773defd" />
<img width="1031" height="724" alt="image" src="https://github.com/user-attachments/assets/1fc7127b-330d-4e00-948c-dcdb7c30479f" />
<img width="1035" height="711" alt="image" src="https://github.com/user-attachments/assets/26e23862-19f6-4336-a00d-e2be0ff7e2d0" />
<img width="1035" height="633" alt="image" src="https://github.com/user-attachments/assets/8ec2db43-628e-46eb-97f5-2616421e9ee2" />



