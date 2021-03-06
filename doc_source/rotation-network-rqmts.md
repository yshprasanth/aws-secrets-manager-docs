# Configuring Your Network to Support Rotating Secrets<a name="rotation-network-rqmts"></a>

To successfully rotate your secrets, the Lambda rotation function must be able to communicate with both the protected database or service, and the AWS Secrets Manager service\. The rotation function makes calls to your database or other service to request that the user's password be updated with a new value\. The function also calls Secrets Manager API operations to retrieve and update the secrets that are involved in the rotation process\. If your Amazon RDS instance or other secret\-protected service is running in a virtual private cloud \(VPC\) provided by Amazon VPC, you must take the following high\-level steps to enable the required connectivity\.
+ **Configure your Lambda rotation function to enable communications between the function and the database instance\.** If you use one of the [database services that are fully supported by Secrets Manager](intro.md#full-rotation-support), then the AWS CloudFormation template that creates your function determines whether your database instance is publicly accessible\.
  + If your protected service is running in a VPC and ***isn't*** publicly accessible, then the AWS CloudFormation template configures the Lambda rotation function to run in the same VPC\. In this scenario, the rotation function can communicate with the protected service directly within the VPC\.
  + If your protected service ***is*** publicly accessible, whether or not it is in a VPC, then AWS CloudFormation template configures the Lambda rotation function ***not*** to run in a VPC\. In this scenario, the Lambda rotation function communicates with the protected service through the publicly accessible connection point\. 

  If you configure your rotation function manually and want to put it in a VPC, then on the function's **Details** page scroll down to the **Networking** section and choose the appropriate **VPC** from the list\.
+ **Configure your VPC to enable communications between the Lambda rotation function running in a VPC and the Secrets Manager service endpoint\.** By default, the Secrets Manager endpoints are on the public Internet\. If your Lambda rotation function and protected database or service are both running in a VPC, then you must perform one of the following steps:
  + You can enable your Lambda function to access the public Secrets Manager endpoint by adding a [NAT gateway](http://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat.html) or an [internet gateway](http://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) to your VPC\. This enables traffic that originates in your VPC to reach the public Secrets Manager endpoint\. *This does expose your VPC to a level of risk* because there's an IP address \(for the gateway\) that can be attacked from the public internet\.
  + You can configure Secrets Manager service endpoints directly within your VPC\. This configures your VPC to intercept any request that's addressed to the public regional endpoint, and redirect it to the private service endpoint that's running within your VPC\. For more details, see [Connecting to Secrets Manager Through a VPC Endpoint](#vpc-endpoint)\.

**Topics**
+ [Connecting to Secrets Manager Through a VPC Endpoint](#vpc-endpoint)
+ [Create a Secrets Manager VPC Private Endpoint](#vpc-endpoint-create)
+ [Connecting to a Secrets Manager VPC Private Endpoint](#vpc-endpoint-connect)
+ [Using a VPC Private Endpoint in a Policy Statement](#vpc-endpoint-policies)
+ [Audit the Use of Your Secrets Manager VPC Endpoint](#vpc-endpoint-audit)

## Connecting to Secrets Manager Through a VPC Endpoint<a name="vpc-endpoint"></a>

Instead of connecting your VPC to the internet, you can connect directly to Secrets Manager through a private endpoint that you configure within your VPC\. When you use a VPC service endpoint, communication between your VPC and Secrets Manager occurs entirely within the AWS network, and requires no public internet access\.

Secrets Manager supports Amazon VPC [interface endpoints](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpce-interface.html) that are provided by [AWS PrivateLink](http://docs.aws.amazon.com/vpc/latest/userguide/VPC_Introduction.html#what-is-privatelink)\. Each VPC endpoint is represented by one or more [elastic network interfaces](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) with private IP addresses in your VPC subnets\.

The VPC interface endpoint connects your VPC directly to Secrets Manager without an internet gateway, NAT device, VPN connection, or AWS Direct Connect connection\. The instances in your VPC don't need public IP addresses to communicate with Secrets Manager\.

For your Lambda rotation function to find the private endpoint, perform one of the following steps:
+ You can manually specify the VPC endpoint in [Secrets Manager API operations](http://docs.aws.amazon.com/secretsmanager/latest/apireference/) and [AWS CLI commands](http://docs.aws.amazon.com/cli/latest/reference/secretsmanager/index.html)\. For example, the following command uses the **endpoint\-url** parameter to specify a VPC endpoint in an AWS CLI command to Secrets Manager\.

  ```
  $ aws secretsmanager list-secrets --endpoint-url https://vpce-1234a5678b9012c-12345678.secretsmanager.us-west-2.vpce.amazonaws.com
  ```
+ If you enable [private DNS hostnames](http://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html#vpce-private-dns) for your VPC private endpoint, you don't even need to specify the endpoint URL\. The standard Secrets Manager DNS hostname that the Secrets Manager CLI and SDKs use by default \(`https://secretsmanager.<region>.amazonaws.com`\) automatically resolves to your VPC endpoint\.

If you enable [private DNS hostnames](http://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html#vpce-private-dns) for your VPC endpoint, you don't even need to specify the endpoint URL\. The standard Secrets Manager DNS hostname that the Secrets Manager CLI and SDKs use by default \(`https://secretsmanager.<region>.amazonaws.com`\) automatically resolves to your VPC endpoint\.

You can also use AWS CloudTrail logs to audit your use of secrets through the VPC endpoint\. And you can use the conditions in IAM and secret \(resource\-based\) policies to deny access to any request that doesn't come from a specified VPC or VPC endpoint\.

**Note**  
Use caution when creating IAM and key policies that are based on your VPC endpoint\. If a policy statement requires that requests come from a particular VPC or VPC endpoint, then requests from other AWS services that interact with the secret on your behalf might fail\. For help, see [Using VPC Endpoint Conditions in Policies with Secrets Manager Permissions](reference_iam-permissions.md#iam-contextkeys-vpcendpoint)\.

**Regions**

Secrets Manager supports VPC endpoints in all AWS Regions where both [Amazon VPC](http://docs.aws.amazon.com/general/latest/gr/rande.html#vpc_region) and [Secrets Manager](http://docs.aws.amazon.com/general/latest/gr/rande.html#asm_region) are available\.

## Create a Secrets Manager VPC Private Endpoint<a name="vpc-endpoint-create"></a><a name="proc-vpc-endpoint-create"></a>

**To create a Secrets Manager VPC private endpoint**  
Follow the steps under one of the following tabs:

------
#### [ Using the AWS Management Console ]<a name="proc-vpc-endpoint-create-console"></a>

1. Open the Amazon VPC console at [https://console\.aws\.amazon\.com/vpc/](https://console.aws.amazon.com/vpc/)\.

1. On the navigation bar, use the region selector to choose the region\.

1. In the navigation pane, choose **Endpoints**\. In the main pane, choose **Create Endpoint**\.

1. For **Service category**, choose **AWS services**\.

1. In the **Service Name** list, choose the entry for the Secrets Manager interface endpoint in the region\. For example, in the US East \(N\.Virginia\) Region, the entry name is `com.amazonaws.us-east-1.secretsmanager`\.

1. For **VPC**, choose your VPC\.

1. For **Subnets**, choose a subnet from each Availability Zone that you want to include\.

   The VPC endpoint can span multiple Availability Zones\. An elastic network interface for the VPC endpoint is created in each subnet that you choose\. Each network interface has a DNS hostname and a private IP address\.

1. If you enable the **Enable Private DNS Name** option \(it's enabled by default\), the standard Secrets Manager DNS hostname \(`https://secretsmanager.<region>.amazonaws.com`\) automatically resolves to your VPC endpoint\. This option makes it easier to use the VPC endpoint\. The Secrets Manager CLI and SDKs use the standard DNS hostname by default, so you don't need to specify the VPC endpoint URL in applications and commands\.

   This feature works only when the `enableDnsHostnames` and `enableDnsSupport` attributes of your VPC are set to `true` \(those are default values\)\. To set these attributes, [update DNS support for your VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-dns.html#vpc-dns-updating)\.

1. For **Security group**, select or create a security group\.

   You can use [security groups](http://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) to control access to your endpoint, much like you would use a firewall\.

1. Choose **Create endpoint**\.

The results show the VPC endpoint, including the VPC endpoint ID and the DNS names that you use to [connect to your VPC endpoint](http://docs.aws.amazon.com/kms/latest/developerguide/kms-vpc-endpoint.html#connecting-vpc-endpoint)\.

You can also use the Amazon VPC tools to view and manage your endpoint\. This includes creating a notification for an endpoint, changing properties of the endpoint, and deleting the endpoint\. For instructions, see [Interface VPC Endpoints](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpce-interface.html)\.

------
#### [ Using the AWS CLI or SDK Operations ]

You can use the [create\-vpc\-endpoint](http://docs.aws.amazon.com/cli/latest/reference/ec2/create-vpc-endpoint.html) command in the AWS CLI to create a VPC endpoint that connects to Secrets Manager\.

Be sure to use `interface` as the VPC endpoint type\. Also, use a service name value that includes `secretsmanager` and the region where your VPC is located\.

The command doesn't include the `PrivateDnsNames` parameter because it defaults to the value `true`\. To disable the option, you can include the parameter with a value of `false`\. Private DNS names are available only when the `enableDnsHostnames` and `enableDnsSupport` attributes of your VPC are set to `true`\. To set these attributes, use the [ModifyVpcAttribute](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_ModifyVpcAttribute.html) API\.

The following diagram shows the general syntax of the command\.

```
aws ec2 create-vpc-endpoint  --vpc-id <vpc id> \
                             --vpc-endpoint-type Interface \
                             --service-name com.amazonaws.<region>.kms \
                             --subnet-ids <subnet id> \
                             --security-group-id <security group id>
```

For example, the following command creates a VPC endpoint in the VPC with VPC ID `vpc-1a2b3c4d`, which is in the `us-west-2` Region\. It specifies just one subnet ID to represent the Availability Zones, but you can specify many\. The security group ID is also required\.

The output includes the VPC endpoint ID and DNS names that you can use to connect to your new VPC endpoint\.

```
$ aws ec2 create-vpc-endpoint  --vpc-id vpc-1a2b3c4d \
                               --vpc-endpoint-type Interface \
                               --service-name com.amazonaws.us-west-2.secretsmanager \
                               --subnet-ids subnet-e5f6a7b8c9 \
                               --security-group-id sg-1a2b3c4d
{
  "VpcEndpoint": {
      "PolicyDocument": "{\n  \"Statement\": [\n    {\n      \"Action\": \"*\", \n      \"Effect\": \"Allow\", \n      \"Principal\": \"*\", \n      \"Resource\": \"*\"\n    }\n  ]\n}",
      "VpcId": "vpc-1a2b3c4d",
      "NetworkInterfaceIds": [
          "eni-abcdef12"
      ],
      "SubnetIds": [
          "subnet-e5f6a7b8c9"
      ],
      "PrivateDnsEnabled": true,
      "State": "pending",
      "ServiceName": "com.amazonaws.us-west-2.kms",
      "RouteTableIds": [],
      "Groups": [
          {
              "GroupName": "default",
              "GroupId": "sg-1a2b3c4d"
          }
      ],
      "VpcEndpointId": "vpce-1234a5678b9012c",
      "VpcEndpointType": "Interface",
      "CreationTimestamp": "2018-06-12T20:14:41.240Z",
      "DnsEntries": [
          {
              "HostedZoneId": "Z7HUB22UULQXV",
              "DnsName": "vpce-1234a5678b9012c-12345678.secretsmanager.us-west-2.vpce.amazonaws.com"
          },
          {
              "HostedZoneId": "Z7HUB22UULQXV",
              "DnsName": "vpce-1234a5678b9012c-12345678-us-west-2a.secretsmanager.us-west-2.vpce.amazonaws.com"
          },
          {
              "HostedZoneId": "Z1K56Z6FNPJRR",
              "DnsName": "secretsmanager.us-west-2.amazonaws.com"
          }
      ]
  }
}
```

------

## Connecting to a Secrets Manager VPC Private Endpoint<a name="vpc-endpoint-connect"></a>

Because, by default, VPC automatically enables private DNS names when you create a VPC private endpoint, you typically don't need to do anything other than use the standard endpoint DNS name for your region\. It automatically resolves to the correct endpoint within your VPC:

```
https://secretsmanager.<region>.amazonaws.com
```

The AWS CLI and SDKs use this hostname by default, so you can begin using the VPC endpoint without changing anything in your scripts and application\.

If you don't enable private DNS names, you can still connect to the endpoint by using its full DNS name\.

For example, this [list\-secrets](http://docs.aws.amazon.com/cli/latest/reference/secretsmanager/list-secrets.html) command uses the `endpoint-url` parameter to specify the VPC private endpoint\. To use a command like this, replace the example VPC private endpoint ID with one in your account\.

```
aws secretsmanager list-secrets --endpoint-url https://vpce-1234a5678b9012c-12345678.secretsmanager.us-west-2.vpce.amazonaws.com
```

## Using a VPC Private Endpoint in a Policy Statement<a name="vpc-endpoint-policies"></a>

You can use IAM policies and Secrets Manager secret policies to control access to your secrets\. You can also use [global condition keys](http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#AvailableKeys) to restrict these policies based on the VPC endpoint or VPC in the request\.
+ Use the `aws:sourceVpce` condition key to grant or restrict access to a secret based on the VPC endpoint\.
+ Use the `aws:sourceVpc` condition key to grant or restrict access to a secret based on the VPC that hosts the private endpoint\.

**Note**  
Use caution when you create IAM and secret policies based on your VPC endpoint\. If a policy statement requires that requests come from a particular VPC or VPC endpoint, then requests from other AWS services that access the secret on your behalf might fail\. For more information, see [Using VPC Endpoint Conditions in Policies with Secrets Manager Permissions](reference_iam-permissions.md#iam-contextkeys-vpcendpoint)\.  
Also, the `aws:sourceIP` condition key isn't effective when the request comes from an Amazon VPC endpoint\. To restrict requests to a VPC endpoint, use the `aws:sourceVpce` or `aws:sourceVpc` condition keys\. For more information, see [VPC Endpoints \- Controlling the Use of Endpoints](http://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html#vpc-endpoints-iam-access) in the *Amazon VPC User Guide*\.

For example, the following sample secret policy allows a user to perform Secrets Manager operations only when the request comes through the specified VPC endpoint\.

When a user makes a request to Secrets Manager, the VPC endpoint ID in the request is compared to the `aws:sourceVpce` condition key value in the policy\. If they don't match, then the request is denied\.

To use a policy like this one, replace the placeholder AWS account ID and VPC endpoint IDs with valid values for your account\.

```
{
    "Id": "example-policy-1",
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Enable Secrets Manager permissions to principals in account 123456789012",
            "Effect": "Allow",
            "Principal": {"AWS":["123456789012"]},
            "Action": ["secretsmanager:*"],
            "Resource": "*"
        },
        {
            "Sid": "Restrict GetSecretValue operation to only my VPC endpoint",
            "Effect": "Deny",
            "Principal": "*",
            "Action": ["secretsmanager:GetSecretValue"],
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpce": "vpce-1234a5678b9012c"
                }
            }
        }

    ]
}
```

You can also use the `aws:sourceVpc` condition key to restrict access to your secrets based on the VPC in which the VPC endpoint resides\.

The following sample secret policy allows commands that create and manage secrets only when they come from `vpc-12345678`\. In addition, the policy allows operations that use access the secret's encrypted value only when the requests come from `vpc-2b2b2b2b`\. You might use a policy like this one if an application is running in one VPC, but you use a second, isolated VPC for management functions\.

To use a policy like this one, replace the placeholder AWS account ID and VPC endpoint IDs with valid values for your account\.

```
{
    "Id": "example-policy-2",
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Allow administrative actions from ONLY vpc-12345678",
            "Effect": "Allow",
            "Principal": {"AWS": "123456789012"},
            "Action": [
                "secretsmangaer:Create*",
                "secretsmanager:Put*",
                "secretsmanager:Update*",
                "secretsmanager:Delete*","secretsmanager:Restore*",
                "secretsmanager:RotateSecret","secretsmanager:CancelRotate*",
                "secretsmanager:TagResource","secretsmanager:UntagResource"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:sourceVpc": "vpc-12345678"
                }
            }
        },
        {
            "Sid": "Allow secret value access from ONLY vpc-2b2b2b2b",
            "Effect": "Allow",
            "Principal": {"AWS": "111122223333"},
            "Action": ["secretsmanager:GetSecretValue"],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:sourceVpc": "vpc-2b2b2b2b"
                }
            }
        },
        {
            "Sid": "Allow read actions from everywhere",
            "Effect": "Allow",
            "Principal": {"AWS": "111122223333"},
            "Action": [
                "secretsmanager:Describe*","secretsmanager:List*","kms:GetRandomPassword"
            ],
            "Resource": "*",
        }
    ]
}
```

## Audit the Use of Your Secrets Manager VPC Endpoint<a name="vpc-endpoint-audit"></a>

When a request to Secrets Manager uses a VPC endpoint, the VPC endpoint ID appears in the [AWS CloudTrail log](monitoring.md) entry that records the request\. You can use the endpoint ID to audit the use of your Secrets Manager VPC endpoint\.

For example, this sample log entry records a `GenerateDataKey` request that used the VPC endpoint\. In this example, the `vpcEndpointId` field appears at the end of the log entry\. For brevity, many irrelevant parts of the example have been omitted\.

```
{
	"eventVersion":"1.05",
	"userIdentity": {
        "type": "IAMUser",
        "arn": "arn:aws:iam::123456789012:user/Anika",
        "accountId": "123456789012",
        "userName": "Anika"
    },
	"eventTime":"2018-01-16T05:46:57Z",
	"eventSource":"secretsmanager.amazonaws.com",
	"eventName":"GetSecretValue",
	"awsRegion":"us-west-2",
	"sourceIPAddress":"172.01.01.001",
	"userAgent":"aws-cli/1.14.23 Python/2.7.12 Linux/4.9.75-25.55.amzn1.x86_64 botocore/1.8.27",
	"requestID":"EXAMPLE1-90ab-cdef-fedc-ba987EXAMPLE",
	"eventID":"EXAMPLE2-90ab-cdef-fedc-ba987EXAMPLE",
	"readOnly":true,
	"eventType":"AwsApiCall",
	"recipientAccountId":"123456789012",
	"vpcEndpointId": "vpce-1234a5678b9012c"
}
```