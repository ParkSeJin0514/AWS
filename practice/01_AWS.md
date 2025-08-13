# 📖 08.13 AWS
### iamusertest1 VPC 생성, 삭제, 조회 권한 부여
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"ec2:CreateVpc",
				"ec2:DeleteVpc",
				"ec2:DescribeAccountAttributes",
				"ec2:DescribeVpcs",
				"ec2:DescribeAvailabilityZones",
				"ec2:CreateTags"
			],
			"Resource": "*",
			"Condition": {
				"StringEquals": {
					"aws:RequestedRegion": "ap-northeast-2"
				}
			}
		}
	]
}
```