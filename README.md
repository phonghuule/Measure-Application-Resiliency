# **Measure Application Resilliency**

This lab is provided as part of **[AWS Innovate For Every Application Edition](https://aws.amazon.com/events/aws-innovate/apj/for-every-app/)**

Click [here](https://github.com/phonghuule/aws-innovate-fea-2022) to explore the full list of hands-on labs.

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Introduction

The purpose if this lab is to teach you the fundamentals of using tests to ensure your implementation is resilient to failure by injecting failure modes into your application. This may be a familiar concept to companies that practice Failure Mode Engineering Analysis (FMEA). It is also a key component of Chaos Engineering, which uses such failure injection to test hypotheses about workload resiliency. One primary capability that AWS provides is the ability to test your systems at a production scale, under load.

It is not sufficient to only design for failure, you must also test to ensure that you understand how the failure will cause your systems to behave. The act of conducting these tests will also give you the ability to create playbooks how to investigate failures. You will also be able to create playbooks for identifying root causes. If you conduct these tests regularly, then you will identify changes to your application that are not resilient to failure and also create the skills to react to unexpected failures in a calm and predictable manner.

In this lab, you will deploy a 2-tier resource, with a reverse proxy (Application Load Balancer), and Web Application on Amazon Elastic Compute Cloud (EC2).

The skills you learn will help you build resilient workloads in alignment with the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

## Goals:
* Reduce fear of implementing resiliency testing by providing examples in common development and scripting languages
* Resilience testing of EC2 instances
* Learn how to implement resiliency using those tests
* Learn how to think about what a failure will cause within your infrastructure
* Learn how common AWS services can reduce mean time to recovery (MTTR)

---

## Deploy the Infrastructure and Application
The first step of this lab is to deploy the static web application stack. 
* You will first deploy an Amazon Virtual Private Cloud (VPC)
* You will then deploy a Static WebApp hosted on Amazon EC2 instances

### 1.1 Log into the AWS console
Sign in to the AWS Management Console as an IAM user who has PowerUserAccess or AdministratorAccess permissions, to ensure successful execution of this lab.

### 1.2 Deploy the VPC infrastructure
1. Launch the AWS Cloud Formation template in **us-east-2 (Ohio) Region** [\
![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2019/10/30/LaunchCFN.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=WebApp1-VPC&templateURL=https://awsinnovatefea2022.s3.amazonaws.com/measure-application-resiliency/vpc-alb-app-db.yaml)
1. Enter the following details:
   * **Stack name**: The name of this stack. For this lab, use **WebApp1-VPC** and match the case.
   * **Parameters**: Parameters may be left as defaults, you can find out more in the description for each.
1. Review the information for the stack. When you're satisfied with the configuration, at the bottom of the page check **I acknowledge that AWS CloudFormation might create IAM resources with custom names** then click **Create stack**.
![cloudformation-vpc-createstack-final](/Images/cloudformation-vpc-createstack-final.png)
1. After a few minutes the final stack status should change from *CREATE_IN_PROGRESS* to *CREATE_COMPLETE*. You can click the **refresh** button to check on the current status.
You have now created the VPC stack (well actually CloudFormation did it for you).

### 1.3 Deploy the EC2s and Static WebApp infrastructure
1.  Launch the AWS Cloud Formation template in **us-east-2 (Ohio) Region** [\
![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2019/10/30/LaunchCFN.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=WebApp1-Static&templateURL=https://awsinnovatefea2022.s3.amazonaws.com/measure-application-resiliency/staticwebapp.yml)
1. Enter the following details:
   * **Stack name**: The name of this stack. For this lab, use **WebApp1-Static** and match the case.
   * **Parameters**: Parameters may be left as defaults, you can find out more in the description for each.
1. Review the information for the stack. When you're satisfied with the configuration, at the bottom of the page check **I acknowledge that AWS CloudFormation might create IAM resources with custom names** then click **Create stack**.
![cloudformation-vpc-createstack-final](/Images/cloudformation-vpc-createstack-final.png)
1. After a few minutes the final stack status should change from *CREATE_IN_PROGRESS* to *CREATE_COMPLETE*. You can click the **refresh** button to check on the current status.

#### Website URL
1. Go to the AWS CloudFormation console at <https://console.aws.amazon.com/cloudformation>.
      * Wait until **WebApp1-Static** stack **status** is _CREATE_COMPLETE_ before proceeding. This should take about four minutes
      * Click on the **WebApp1-Static** stack
      * Click on the **Outputs** tab
      * For the Key **WebsiteURL** copy the value.  This is the URL of your test web service
      * Save this URL - you will need it later
---

## Test Resiliency Using Failure Injection
**Failure injection** (also known as **chaos testing**) is an effective and essential method to validate and understand the resiliency of your workload and is a recommended practice of the [AWS Well-Architected Reliability Pillar](https://aws.amazon.com/architecture/well-architected/). Here you will initiate various failure scenarios and assess how your system reacts.

### Preparation

Before testing, please prepare the following:

1. Region must be the one you selected when you deployed your WebApp
      * We will be using the AWS Console to assess the impact of our testing
      * Throughout this lab, make sure you are in the correct region. For example the following screen shot shows the desired region assuming your WebApp was deployed to **Ohio** region

        ![SelectOhio](/Images/SelectOhio.png)

1. Get VPC ID
      * A VPC (Amazon Virtual Private Cloud) is a logically isolated section of the AWS Cloud where you have deployed the resources for your service
      * For these tests you will need to know the **VPC ID** of the VPC you created as part of deploying the service
      * Navigate to the VPC management console: <https://console.aws.amazon.com/vpc>
      * In the left pane, click **Your VPCs**
      * 1 - Tick the checkbox next to **WebApp1-VPC**
      * 2 - Copy the **VPC ID**

    ![GetVpcId](/Images/GetVpcId.png)

     * Save the VPC ID - you will use later whenever `<vpc-id>` is indicated in a command

1. Get familiar with the service website
      1. Point a web browser at the URL you saved from earlier
            * If you do not recall this, then in the _WebApp1-Static_ stack click the **Outputs** tab, and open the **WebsiteURL** value in your web browser, this is how to access what you just created)
      1. Note the **instance_id** (begins with **i-**) - this is the EC2 instance serving this request
      1. Refresh the website several times watching these values
      1. Note the values change. You have deployed two web servers per each of two Availability Zones.
         * The AWS Elastic Load Balancer (ELB) sends your request to any of these two healthy instances.

### 2.1 EC2 failure injection

This failure injection will simulate a critical problem with one of the two web servers used by your service.

1. Navigate to the EC2 console at <http://console.aws.amazon.com/ec2> and click **Instances** in the left pane.

1. There are two EC2 instances with a name beginning with **WebApp1**. For these EC2 instances note:
      1. Each has a unique *Instance ID*
      1. There is two instances per each Availability Zone
      1. All instances are healthy

      ![EC2InitialCheck](/Images/EC2InitialCheck.png)
1. Open up two more console in separate tabs/windows. From the left pane, open **Target Groups** and **Auto Scaling Groups** in separate tabs. You now have two console views open
      ![NavToTargetGroupAndScalingGroup](/Images/NavToTargetGroupAndScalingGroup.png)
1. Download the file [fail_instance.py](https://awsinnovatefea2022.s3.amazonaws.com/measure-application-resiliency/fail_instance.py), you will use this file later to inject failure into your environment
1. Navigate to [AWS CloudShell](https://us-east-2.console.aws.amazon.com/cloudshell/home?region=us-east-2#) and wait for **AWS CloudShell** to create environment

      ![CloudShell](/Images/CloudShell.png)
AWS CloudShell is a browser-based shell that gives you command-line access to your AWS resources in the selected AWS region. AWS CloudShell comes pre-installed with popular tools for resource management and creation. You have the same credentials as you used to log in to the console. It saved you time to configure AWS credentials and execution environment in your local machine.
1. Under **Action** in the right hand side, choose to **Upload file** [fail_instance.py](https://awsinnovatefea2022.s3.amazonaws.com/measure-application-resiliency/fail_instance.py) that you downloaded in previous step
You should see successful message after the upload
      
      ![CloudShell-Upload](/Images/CloudShell_Upload.png)

      ![CloudShell-Upload-Message](/Images/CloudShell_Upload_Message.png)

1. To fail one of the EC2 instances, use the VPC ID as the command line argument replacing `<vpc-id>` in the script below. (We are using **Python** script)
```
python3 fail_instance.py <vpc-id>
```
8. The specific output will include a reference to the ID of the EC2 instance and an indicator of success.  Here is the output:
```
[cloudshell-user@ip-10-0-48-94 ~]$ python3 fail_instance.py vpc-0c97496bdf2babb2a
vpc-0c97496bdf2babb2a
Terminating instance:
i-0c4f71684a880cf1e
[cloudshell-user@ip-10-0-48-94 ~]$ 
```
9. Go to the *EC2 Instances* console which you already have open (or [click here to open a new one](http://console.aws.amazon.com/ec2/v2/home?#Instances:))

      * Refresh it. (_Note_: it is usually more efficient to use the refresh button in the console, than to refresh the browser)
       ![RefreshButton](/Images/RefreshButton.png)
      * Observe the status of the instance reported by the script. In the screen cap below it is _shutting down_ as reported by the script and will ultimately transition to _terminated_.

        ![EC2ShuttingDown](/Images/EC2ShuttingDown.png)

### 2.2 System response to EC2 instance failure

Watch how the service responds. Note how AWS systems help maintain service availability. Test if there is any non-availability, and if so then how long.

#### 2.2.1 System availability

Refresh the service website several times. Note the following:

* Website remains available
* The remaining EC2 instance are handling all the requests (as per the displayed instance_id)

#### 2.2.2 Load balancing

Load balancing ensures service requests are not routed to unhealthy resources, such as the failed EC2 instance.

1. Go to the **Target Groups** console you already have open (or [click here to open a new one](http://console.aws.amazon.com/ec2/v2/home?#TargetGroups:))
     * If there is more than one target group, select the one with whose name begins with  **WebAp**

1. Click on the **Targets** tab and observe:
      * Status of the instances in the group. The load balancer will only send traffic to healthy instances.
      * When the auto scaling launches a new instance, it is automatically added to the load balancer target group.
      * In the screen cap below the _unhealthy_ instance is the newly added one.  The load balancer will not send traffic to it until it is completed initializing. It will ultimately transition to _healthy_ and then start receiving traffic.
      * Note the new instance was started in the same Availability Zone as the failed one. Amazon EC2 Auto Scaling automatically maintains balance across all of the Availability Zones that you specify.

        ![TargetGroups](/Images/TargetGroups.png)  

1. From the same console, now click on the **Monitoring** tab and view metrics such as **Unhealthy hosts** and **Healthy hosts**

   ![TargetGroupsMonitoring](/Images/TargetGroupsMonitoring.png)

#### 2.2.3 Auto scaling

Autos scaling ensures we have the capacity necessary to meet customer demand. The auto scaling for this service is a simple configuration that ensures at least two EC2 instances are running. More complex configurations in response to CPU or network load are also possible using AWS.

1. Go to the **Auto Scaling Groups** console you already have open (or [click here to open a new one](http://console.aws.amazon.com/ec2/autoscaling/home?#AutoScalingGroups:))
      * If there is more than one auto scaling group, select the one with the name that starts with **WebApp1**

1. Click on the **Activity History** tab and observe:
      * The screen cap below shows that instances were successfully started at 17:25
      * At 19:29 the instance targeted by the script was put in _draining_ state and a new instance ending in _...62640_ was started, but was still initializing. The new instance will ultimately transition to _Successful_ status

        ![AutoScalingGroup](/Images/AutoScalingGroup.png)  

_Draining_ allows existing, in-flight requests made to an instance to complete, but it will not send any new requests to the instance. *__Learn more__: After the lab [see this blog post](https://aws.amazon.com/blogs/aws/elb-connection-draining-remove-instances-from-service-with-care/) for more information on _draining_.*

*__Learn more__: After the lab see [Auto Scaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) to learn more how auto scaling groups are setup and how they distribute instances, and [Dynamic Scaling for Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html) for more details on setting up auto scaling that responds to demand*

#### 2.2.4 EC2 failure injection - conclusion

Deploying multiple servers and Elastic Load Balancing enables a service suffer the loss of a server with no availability disruptions as user traffic is automatically routed to the healthy servers. Amazon Auto Scaling ensures unhealthy hosts are removed and replaced with healthy ones to maintain high availability.

| |
|:---:|
|**Availability Zones** (**AZ**s) are isolated sets of resources within a region, each with redundant power, networking, and connectivity, housed in separate facilities. Each Availability Zone is isolated, but the Availability Zones in a Region are connected through low-latency links. AWS provides you with the flexibility to place instances and store data across multiple Availability Zones within each AWS Region for high resiliency.|
|*__Learn more__: After the lab [see this whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/global-infrastructure.html) on regions and availability zones*|


## Tear Down

The following instructions will remove the resources that you have created in this lab.

If you deployed the CloudFormation stacks as part of the prerequisites for this lab, then delete these stacks to remove all the AWS resources. If you need help with how to delete CloudFormation stacks then follow these instructions to tear down those resources:
### Delete the WebApp resource
1. Sign in to the AWS Management Console, select your preferred region, and open the CloudFormation console at [https://console.aws.amazon.com/cloudformation/](https://console.aws.amazon.com/cloudformation/).
2. Click the radio button on the left of the **WebApp1-Static** stack.
3. Click the **Actions** button then click **Delete stack**.
4. Confirm the stack and then click **Delete** button.

### Wait for this stack deletion to complete
### Delete the VPC resources
1. Sign in to the AWS Management Console, select your preferred region, and open the CloudFormation console at [https://console.aws.amazon.com/cloudformation/](https://console.aws.amazon.com/cloudformation/).
2. Click the radio button on the left of the *WebApp1-VPC* stack.
3. Click the **Actions** button then click **Delete stack**.
4. Confirm the stack and then click **Delete** button.

## Survey
Let us know what you thought of this session and how we can improve the presentation experience for you in the future by completing this [event session poll](https://amazonmr.au1.qualtrics.com/jfe/form/SV_1U4cxprfqLngWGy?Session=HOL11). Participants who complete the surveys from AWS Innovate Online Conference will receive a gift code for USD25 in AWS credits1, 2 & 3. AWS credits will be sent via email by September 29, 2023.
Note: Only registrants of AWS Innovate Online Conference who complete the surveys will receive a gift code for USD25 in AWS credits via email.

<sup>1</sup>AWS Promotional Credits Terms and conditions apply: https://aws.amazon.com/awscredits/ 

<sup>2</sup>Limited to 1 x USD25 AWS credits per participant.

<sup>3</sup>Participants will be required to provide their business email addresses to receive the gift code for AWS credits.
