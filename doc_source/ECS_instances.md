# Amazon ECS container instances<a name="ECS_instances"></a>

An Amazon ECS container instance is an Amazon EC2 instance that is running the Amazon ECS container agent and has been registered into an Amazon ECS cluster\. When you run tasks with Amazon ECS using the EC2 launch type or an Auto Scaling group capacity provider, your tasks are placed on your active container instances\.

**Note**  
Tasks using the Fargate launch type are deployed onto infrastructure managed by AWS, so this topic does not apply\.

Amazon ECS supports the following container instance types\.
+ Linux
+ Windows
+ External, such as an on\-premises VM

**Topics**
+ [Container instance concepts](#container_instance_concepts)
+ [Container instance lifecycle](#container_instance_life_cycle)
+ [Check the instance IAM role for your account](#check-instance-role)
+ [Linux instances](ecs-linux.md)
+ [Windows instances](ecs-windows.md)
+ [External instances \(Amazon ECS Anywhere\)](ecs-anywhere.md)
+ [Monitoring your container instances](using_cloudwatch_logs.md)
+ [Container instance draining](container-instance-draining.md)

## Container instance concepts<a name="container_instance_concepts"></a>
+ Your container instance must be running the Amazon ECS container agent\. The container agent is able to register the instance into one of your clusters\. If you are using an Amazon ECS\-optimized AMI, the agent is already installed\. To use a different operating system, install the agent\. For more information, see [Amazon ECS container agent](ECS_agent.md)\.
+ Because the Amazon ECS container agent makes calls to Amazon ECS on your behalf, you must launch container instances with an IAM role that authenticates to your account and provides the required resource permissions\. For more information, see [Amazon ECS container instance IAM role](instance_IAM_role.md)\.
+ Beginning with Linux Amazon ECS\-optimized AMI version `20200430` and later, the Amazon EC2 Instance Metadata Service Version 2 \(IMDSv2\) is supported on your container instances\. For Amazon ECS\-optimized AMIs prior to version `20200430`, Amazon EC2 Instance Metadata Service Version 1 \(IMDSv1\) is supported\. For more information, see [Configuring the instance metadata service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) in the *Amazon EC2 User Guide for Linux Instances*\. 
+ If any of the containers associated with your tasks require external connectivity, you can map their network ports to ports on the host Amazon ECS container instance so they are reachable from the internet\. Your container instance security group must allow inbound access to the ports you want to expose\. For more information, see [Create a Security Group](https://docs.aws.amazon.com/AmazonVPC/latest/GettingStartedGuide/getting-started-create-security-group.html) in the [Amazon VPC Getting Started Guide](https://docs.aws.amazon.com/AmazonVPC/latest/GettingStartedGuide/)\.
+ We strongly recommend launching your container instances inside a VPC, because Amazon VPC delivers more control over your network and offers more extensive configuration capabilities\. For more information, see [Amazon EC2 and Amazon Virtual Private Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-vpc.html) in the *Amazon EC2 User Guide for Linux Instances*\.
+ Container instances need access to communicate with the Amazon ECS service endpoint\. This can be through an interface VPC endpoint or through your container instances having public IP addresses\.

  For more information about interface VPC endpoints, see [Amazon ECS interface VPC endpoints \(AWS PrivateLink\)](vpc-endpoints.md)\.

  If you do not have an interface VPC endpoint configured and your container instances do not have public IP addresses, then they must use network address translation \(NAT\) to provide this access\. For more information, see [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) in the *Amazon VPC User Guide* and [HTTP proxy configuration](http_proxy_config.md) in this guide\. For more information, see [Create a virtual private cloud](get-set-up-for-amazon-ecs.md#create-a-vpc)\.\.
+ The type of Amazon EC2 instance that you choose for your container instances determines the resources available in your cluster\. Amazon EC2 provides different instance types, each with different CPU, memory, storage, and networking capacity that you can use to run your tasks\. For more information, see [Amazon EC2 Instances](https://aws.amazon.com/ec2/instance-types/)\.
+ Because each container instance has unique state information that is stored locally on the container instance and within Amazon ECS:
  + You should not deregister an instance from one cluster and re\-register it into another\. To relocate container instance resources, we recommend that you terminate container instances from one cluster and launch new container instances with the latest Amazon ECS\-optimized Amazon Linux 2 AMI in the new cluster\. For more information, see [Terminate Your Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html) in the *Amazon EC2 User Guide for Linux Instances* and [Launching an Amazon ECS Linux container instance](launch_container_instance.md)\.
  + You cannot stop a container instance and change its instance type\. Instead, we recommend that you terminate the container instance and launch a new container instance with the desired instance size and the latest Amazon ECS\-optimized Amazon Linux 2 AMI in your desired cluster\. For more information, see [Terminate Your Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html) in the *Amazon EC2 User Guide for Linux Instances* and [Launching an Amazon ECS Linux container instance](launch_container_instance.md) in this guide\.

## Container instance lifecycle<a name="container_instance_life_cycle"></a>

When the Amazon ECS container agent registers an Amazon EC2 instance into your cluster, the Amazon EC2 instance reports its status as `ACTIVE` and its agent connection status as `TRUE`\. This container instance can accept `RunTask` requests\.

If you stop \(not terminate\) an Amazon ECS container instance, the status remains `ACTIVE`, but the agent connection status transitions to `FALSE` within a few minutes\. Any tasks that were running on the container instance stop\. If you start the container instance again, the container agent reconnects with the Amazon ECS service, and you are able to run tasks on the instance again\.

**Important**  
If you stop and start a container instance, or reboot that instance, some older versions of the Amazon ECS container agent register the instance again without deregistering the original container instance ID\. In this case, Amazon ECS lists more container instances in your cluster than you actually have\. \(If you have duplicate container instance IDs for the same Amazon EC2 instance ID, you can safely deregister the duplicates that are listed as `ACTIVE` with an agent connection status of `FALSE`\.\) This issue is fixed in the current version of the Amazon ECS container agent\. For more information about updating to the current version, see [Updating the Amazon ECS container agent](ecs-agent-update.md)\.

If you change the status of a container instance to `DRAINING`, new tasks are not placed on the container instance\. Any service tasks running on the container instance are removed, if possible, so that you can perform system updates\. For more information, see [Container instance draining](container-instance-draining.md)\.

If you deregister or terminate a container instance, the container instance status changes to `INACTIVE` immediately, and the container instance is no longer reported when you list your container instances\. However, you can still describe the container instance for one hour following termination\. After one hour, the instance description is no longer available\.

**Important**  
 You can drain the instances manually, or build an Auto Scaling group lifecycle hook to set the instance status to `DRAINING`\. See [Amazon EC2 Auto Scaling lifecycle hooks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html) for more information about Auto Scaling lifecycle hooks\.

## Check the instance IAM role for your account<a name="check-instance-role"></a>

The Amazon ECS container agent makes calls to the Amazon ECS APIs on your behalf\. Container instances that run the agent require an IAM policy and role for the service to know that the agent belongs to you\.

In most cases, the Amazon ECS instance role is automatically created for you in the console first\-run experience\. You can use the following procedure to check and see if your account already has an Amazon ECS service role\.

**To check for the `ecsInstanceRole` in the IAM console**

1. Sign in to the AWS Management Console and open the IAM console at [https://console\.aws\.amazon\.com/iam/](https://console.aws.amazon.com/iam/)\.

1. In the navigation pane, choose **Roles**\. 

1. Search the list of roles for `ecsInstanceRole`\. If the role exists, you do not need to create it\. If the role does not exist, follow the procedures in [Amazon ECS container instance IAM role](instance_IAM_role.md) to create the role\. 