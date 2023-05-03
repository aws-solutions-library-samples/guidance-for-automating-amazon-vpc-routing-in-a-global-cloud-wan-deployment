## AWS Cloud WAN and VPC IPAM integration for end-to-end automated routing

AWS Cloud WAN is a service that you can use to build, manage, and monitor a unified global network that connects resources running across your cloud and on-premises environments. Cloud WAN ensures dynamic route propagation across multiple regions. So for a sample deployment that spans across 2 regions: you'll create VPC attachments into Cloud WAN segments. Cloud WAN will ensure that each VPC's CIDR is dynamically learned across regions. To ensure that end to end connectivity works between your VPCs, you'll need to update the routing tables of the VPCs to provide network connectivity for workloads that reside within the VPCs. This is illustrated in Figure-1 below:

![Figure-1: Updating VPC routing tables](.img/Figure-1.png)

In this Guidance, we'll see how to automatically add (or delete) routes to VPC routing tables every time a VPC is attached into (or deleted from) a Cloud WAN segment. This will provide you with a fully automated end-to-end routing solution wherein workloads residing within VPCs have network connectivity to each other irrespective of the region where a VPC resides.

This solution is event driven, in that the route addition/deletion is triggered when a VPC is attached into (or detached from) a Cloud WAN segment.

### Prerequisites:

- Working knowledge of [AWS Cloud WAN] (https://docs.aws.amazon.com/network-manager/latest/cloudwan/what-is-cloudwan.html
)
- An [AWS account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup&code_challenge_method=SHA-256&code_challenge=0HQWGyGWYR-1yDUXafaSt2wlhL8OZGAHfZx3sDZN4mE) that you are able to use for testing, that is not used for production or other purposes. NOTE: You will be billed for any applicable AWS resources used if you complete this lab that are not covered in the [AWS Free Tier](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all).

- To deploy this solution you will have to have the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed.

- If on Windows you will need a utility to execute Makefiles 

## How this solution works

Cloud WAN generates an event whenever a VPC attachment is created. The event has a JSON format and contains important information about the details of the VPC attachment. The structure of such an event is shown in Figure-3 below. Note that the "changeType" key has a value of "VPC_ATTACHMENT_CREATED"

![Figure-3: Structure of a VPC_ATTACHMENT_CREATED event]

Similarly, Cloud WAN generates an event every time a VPC is attachment is deleted. In such a case, the event's "changeType" key has a value of "VPC_ATTACHMENT_DELETED".

This solution uses Event Bridge and Lambda to detect the VPC_ATTACHMENT_CREATED and VPC_ATTACHMENT_DELETED events generated by Cloud WAN. Event Bridge then triggers a regional lambda function and passes the event's json object to the lambda function. The lambda function has the necessary logic that fetches the department specific CIDR from VPC IPAM. After getting the department CIDR, the Lambda function creates a Prefix List, and pushes a route into the VPC's routing table. The destination for this route is the Prefix List and the target for the route is the Cloud WAN core network.

The logical flow for this solution is shown in Figure-ABC below:

![Cloud WAN Solution](./img/solution.png)

Whenever a VPC attachment is deleted, this solution detects that event and deletes the corresponding route from the VPC's routing table.

## Step by step instructions for a net new deployment

Note that the instructions provided in this section build out all the necessary Cloud WAN, IPAM and VPC constructs that are shown in Figure-2 below. Please navigate to "Instructions for an existing or customized deployment" section in case you'd like to use this solution for an existing deployment wherein you already have a Cloud WAN network or your own IP addressing scheme.

### Scenario

You have a global network, across 2 regions. Cloud WAN provides global network orchestration across these regions, and there are 3 Cloud WAN segments, one each for departments named Finance, Sales and HR. VPC IPAM stores the CIDRs assigned to each of the 3 departments. Note that this solution uses VPC IPAM purely to store the CIDRs of each department. You can optionally use VPC IPAM to provide CIDRs for your VPCs. The target state of the deployment is shown in Figure-2 below.

![Figure-2: Cloud WAN Infrastructure](./img/cloud-wan-demo.png)

###  Step 1 - Deployment (Single command line deployment instructions)

This sample project is meant to be deployed to a single account and multiple regions. By default, AWS regions us-east-1, us-west-2 and ap-southeast-1 are in use.

#### Step 1.1 - Clone the repo and navigate to the correct directory

```bash
git clone https://github.com/aws-solutions-library-samples/guidance-for-end-to-end-fully-automated-global-network-on-aws.git && cd guidance-for-end-to-end-fully-automated-global-network-on-aws/samples
```

#### Step 1.2 - Deploy the CloudFormation stacks.

```bash
make deploy
```

>NOTE: This deployment will take greater than 15 minutes.

### Step 2 - Testing and validation

After the VPC has been attached to Cloud WAN give the solution about 5-10 minutes before the routing tables have been updated. 

1. In us-east-1 open the [EC2 console](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Instances:instanceState=running) and connect to the `cloud-wan-demo-instance` ec2 instance with SSM. 
2. In ap-southeast-1 open the [EC2 console](https://ap-southeast-1.console.aws.amazon.com/ec2/home?region=ap-southeast-1#Instances:instanceState=running) and copy the private ip of the instance. 
3. In the ec2 SSM terminal window type `ping ` and then paste the private ip address of the instance in ap-southeast-1.

You now have dynamic end-to-end routing between VPCs in AWS, even when your VPCs are deployed in different AWS Regions!

>For additional verification you can check the default routing tables in the hr vpcs in both regions. You will notice a prefix list that was added when your VPC was attached to Cloud WAN.  

### Step 3 - Clean up

Delete CloudFormation stacks. This will remove all the stacks. (cloud wan, ipam, regional infrastructure, eventbridge rules, and Cloud WAN VPC attachments). Note that this operation must be done from within the samples directory that you navigated to in Step 1.

```bash
make undeploy
```

>NOTE: Remember to do `make undeploy` when you are done with the lab and your tests, to avoid undesired charges.

### Optional - Detailed instructions (Deploy a single stack at a time for a net new deployment)

This section not needed if you deployed step 1. It is to deploy each stack individually. 

1. `make deploy-global-infra` - This will deploy Cloud WAN and IPAM in us-east-1. 
2. `make deploy-target-1` - This will deploy a VPC, private subnet, VPC endpoints for SSM, an EC2 instance, lambda function, eventbridge rule, and Cloud WAN attachment in us-east-1. 
3. `make deploy-target-2` - This will deploy a VPC, private subnet, VPC endpoints for SSM, an EC2 instance, lambda function, eventbridge rule, and Cloud WAN attachment in ap-southeast-1.

## Instructions for an existing or customized deployment

This section details out the instructions that you'll need to execute to use this solution with an existing deployment, or in case you'd like to customize your Cloud WAN or IPAM deployment according to your use case.

After you complete these steps, this solution will automate the addition (or deletion) of a department-specific route into VPCs' routing table every time the VPC is attached into (or detached from) a Cloud WAN segment. This solution pulls the CIDR of the route from VPC IPAM Pools. This solution deploys regional lambda functions that include the necesssary logic for fetching department specific CIDR from VPC IPAM, and pushing the route into VPC's routing table.

### Step 1: Create your IPAM deployment

This solution uses VPC IPAM to store the CIDRs that are needed for populating VPC's routing tables. To use this solution for an exsiting or customized deployment, the first step is to create VPC IPAM Pools. Check out [VPC IPAM documentation](https://docs.aws.amazon.com/vpc/latest/ipam/what-it-is-ipam.html) for details on how to create VPC IPAM pools.

Make a note of the VPC IPAM Pools' names. You'll use the same name as tag while creating VPC attachments into Cloud WAN. 

### Step 2: Create Cloud WAN deployment

Create your Cloud WAN deployment and the corresponding Cloud WAN policy based on your use-case. Check out [Cloud WAN documentation](https://docs.aws.amazon.com/network-manager/latest/cloudwan/what-is-cloudwan.html) for details.

### Step 3: Deploy Event Bridge and Lambda stacks

Clone this repo and navigate to the repo folder.

```bash
git clone https://github.com/aws-solutions-library-samples/guidance-for-end-to-end-fully-automated-global-network-on-aws.git && cd guidance-for-end-to-end-fully-automated-global-network-on-aws
```

Repeat steps 3.1 and 3.2 for each of your target regions. For example, if you have a Cloud WAN deployment in us-west-1 and us-east-1, you would repeat steps 3.1 and 3.2 once each for us-west-1 and us-east-1.

#### Step 3.1: Create regional stack in target region

Deploy the regional_lambda.yaml cloudformation template in the target region. This stack deploys two resources, a regional Event Bridge rule and a Lambda function that is triggered by the rule. The Lambda function includes the necessary code for fetching the correct CIDR from IPAM, adding and removing a prefix-list from the VPC's routing table.

After deploying the stack, make a note of the CloudFormation output named 'EventBridgeArn'.

#### Step-3.2: Deploy eventbridge_rule.yaml stack in us-west-2 region

Deploy the eventbridge_rule.yaml in us-west-2 Region. This stack creates the Event Bridge rule that traps on 'VPC attachment created' and 'VPC attachment deleted' events. Cloud WAN generates events in us-west-2 which will be sent to the event bridge rule that is created by the cloudformation stack.

### Step-4: Create VPCs

Create the VPCs that where you'll create your workloads. You can optionally use VPC IPAM to vend out CIDRs to these VPCs. Note that this solution will work even if you don't use VPC IPAM to vend out CIDRs to the VPCs.

### Step-5: Create VPC Attachments into Cloud WAN
 
At the time of creating VPC attachments into Cloud WAN, ensure that you provide a key:value tag pair on the attachment. The tag must be in the format of:
```bash
department:<value>
```

The value of the tag must match the name of the corresponding VPC IPAM pool that you created in Step 1 above.

For example: if you created a VPC IPAM pool named 'finance', ensure that you tag the VPC attachment with a tag key-value pair of:
```bash
department:finance
```

### Step-6: Verification

Navigate to VPC section on the console. Select the VPC that you connected into CloudWAN. Navigate to the VPC's main routing table. You should see a route that has a prefix list as the destination and Cloud WAN core network as the target. This is the route that this solution pushed into your VPC's routing table.

### Considerations
1. This solution is useful when your VPCs have multiple exit points. In the scenario drawing, each VPC has 2 exit points. One towards the Cloud WAN core-network, and second towards AWS Direct Connect gateway
2. In addition to the networking constructs, there's pricing associated with [EventBridge] (https://aws.amazon.com/eventbridge/pricing/) and with [Lambda] (https://aws.amazon.com/lambda/pricing/). Please check out service pricing pages for more details
3. This solution uses VPC IPAM to store CIDRs for storing department specific CIDRs. When enabled, VPC IPAM imports all IP addressing data from an existing deployment, and charges an hourly rate for every active IP address. Please check out [VPC IPAM pricing] (https://aws.amazon.com/vpc/pricing/) for more details

## References

- This solution was presented at re:Invent 2022 as a breakout session: [Demystifying VPC IP addressing & creating a complete routing solution](https://www.youtube.com/watch?v=rvJMCdjSZxU)
- [AWS Cloud WAN documentation.](https://docs.aws.amazon.com/network-manager/latest/cloudwan/what-is-cloudwan.html)
- [AWS Cloud WAN Workshop.](https://catalog.workshops.aws/cloudwan/en-US)
- Blog post: [Introducing AWS Cloud WAN (Preview).](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-aws-cloud-wan-preview/)
- Blog post: [AWS Cloud WAN and AWS Transit Gateway migration and interoperability patterns](https://aws.amazon.com/blogs/networking-and-content-delivery/aws-cloud-wan-and-aws-transit-gateway-migration-and-interoperability-patterns/)

Please reach out to aichadha@amazon.com or duanlig@amazon.com in case you have any questions about this repo.