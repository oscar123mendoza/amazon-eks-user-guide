# Network load balancing on Amazon EKS<a name="load-balancing"></a>

When you create a Kubernetes `Service` of type `LoadBalancer`, an AWS Network Load Balancer \(NLB\) or Classic Load Balancer \(CLB\) is provisioned that load balances network traffic\. To learn more about the differences between the two types of load balancers, see [Elastic Load Balancing features](https://aws.amazon.com/elasticloadbalancing/features/) on the AWS website\. For more information about a Kubernetes Service, see [Service](https://kubernetes.io/docs/concepts/services-networking/service/) in the Kubernetes documentation\. NLBs can be used with pods deployed to nodes or to AWS Fargate IP targets\. You can deploy an AWS load balancer to a public or private subnet\.

Network traffic is load balanced at L4 of the OSI model\. To load balance application traffic at L7, you deploy a Kubernetes `Ingress`, which provisions an AWS Application Load Balancer\. For more information, see [Application load balancing on Amazon EKS](alb-ingress.md)\. To learn more about the differences between the two types of load balancing, see [Elastic Load Balancing features](https://aws.amazon.com/elasticloadbalancing/features/) on the AWS website\.

In Amazon EKS, you can load balance network traffic to an NLB \(*instance* or *IP* target\) or a CLB \(*instance* target only\)\. For more information about NLB target types, see [Target type](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#target-type) in the User Guide for Network Load Balancers\. When you load balance network traffic to instance targets, the Kubernetes in\-tree controller creates the NLB or CLB\. To load balance network traffic to IP target types, the AWS Load Balancer Controller creates an NLB\. For more information, see [AWS load balancer controller](https://github.com/kubernetes-sigs/aws-alb-ingress-controller) on GitHub\.

**Prerequisites**

Before you can load balance network traffic to an application, you must meet the following requirements\.
+ Have an existing cluster\. If you don't have an existing cluster, see [Getting started with Amazon EKS](getting-started.md)\. If you're load balancing to IP targets, the cluster must be 1\.18 or later\. To update an existing cluster, see [Updating a cluster](update-cluster.md)\.
+ If you're load balancing to IP targets, you must have the AWS Load Balancer Controller provisioned on your cluster\. For more information, see [AWS Load Balancer Controller](aws-load-balancer-controller.md)\.
+ Public subnets must be tagged as follows so that Kubernetes knows to use only those subnets for external load balancers instead of choosing a public subnet in each Availability Zone \(in lexicographical order by subnet ID\)\. If you use `eksctl` or an Amazon EKS AWS CloudFormation template to create your VPC after March 26, 2020, then the subnets are tagged appropriately when they're created\. For more information about the Amazon EKS AWS CloudFormation VPC templates, see [Creating a VPC for your Amazon EKS cluster](create-public-private-vpc.md)\.    
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html)
+ Private subnets must be tagged as follows so that Kubernetes knows it can use the subnets for internal load balancers\. If you use `eksctl` or an Amazon EKS AWS CloudFormation template to create your VPC after March 26, 2020, then the subnets are tagged appropriately when they're created\. For more information about the Amazon EKS AWS CloudFormation VPC templates, see [Creating a VPC for your Amazon EKS cluster](create-public-private-vpc.md)\.    
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html)

**Considerations**
+ Use of the UDP protocol is supported with the load balancer on Amazon EKS clusters with the following platform versions\. For more information, see [Amazon EKS platform versions](platform-versions.md)\.    
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html)
+ You can only use NLB *IP* targets with the [Amazon EKS VPC CNI plugin](pod-networking.md)\. You can use NLB *instance* targets with the Amazon EKS VPC CNI plugin or [alternate compatible CNI plugins](alternate-cni-plugins.md)\.
+ You can only use *IP* targets with NLB\. You can't use IP targets with CLBs\.
+ The configuration of your load balancer is controlled by annotations that are added to the manifest for your service\. If you want to add tags to the load balancer when \(or after\) it's created, add the following annotation in your service specification\. For more information, see [Other ELB annotations](https://kubernetes.io/docs/concepts/services-networking/service/#other-elb-annotations) in the Kubernetes documentation\.

  ```
  service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags
  ```
+ If you're using Amazon EKS 1\.16 or later, you can assign [Elastic IP addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html) to the Network Load Balancer by adding the following annotation\. Replace the `<example-values>` \(including `<>`\) with the Allocation IDs of your Elastic IP addresses\. The number of Allocation IDs must match the number of subnets used for the load balancer\.

  ```
  service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-<xxxxxxxxxxxxxxxxx>,eipalloc-<yyyyyyyyyyyyyyyyy>
  ```
+ For each NLB that you create on 1\.16 or later clusters, Amazon EKS adds one inbound rule to the node's security group for client traffic and one rule for each load balancer subnet in the VPC for health checks\. On 1\.15 clusters, one rule is added for each CIDR block assigned to the VPC\. Deployment of a service of type `LoadBalancer` can fail if Amazon EKS attempts to create rules that exceed the quota for the maximum number of rules allowed for a security group\. For more information, see [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-security-groups) in Amazon VPC quotas in the Amazon VPC User Guide\. Consider the following options to minimize the chances of exceeding the maximum number of rules for a security group\.
  + Request an increase in your rules per security group quota\. For more information, see [Requesting a quota increase](https://docs.aws.amazon.com/servicequotas/latest/userguide/request-quota-increase.html) in the Service Quotas User Guide\.
  + Use [Load balancer – IP targets](#load-balancer-ip), rather than instance targets\. With IP targets, rules can potentially be shared for the same target ports\. Load balancer subnets can be manually specified with an annotation\. For more information, see [Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/#subnets) on GitHub\.
  + Use an Ingress, instead of a Service of type `LoadBalancer` to send traffic to your service\. The AWS Application Load Balancer \(ALB\) requires fewer rules than NLBs\. An ALB can also be shared across multiple Ingresses\. For more information, see [Application load balancing on Amazon EKS](alb-ingress.md)\.
  + Deploy your clusters to multiple accounts\.

## Load balancer – Instance targets<a name="load-balancer-instance"></a>

NLB or CLBs with instance targets are created by the Kubernetes in\-tree load balancing controller\. The in\-tree controller is included with Kubernetes, so you don't need to deploy it to your cluster\. You can use NLB instance targets with pods deployed to nodes, but not to Fargate\. To load balance network traffic across pods deployed to Fargate, you must use [IP targets](#load-balancer-ip)\. By default, external \(public\) Classic Load Balancers are created when you deploy a Kubernetes service of type `LoadBalancer`\. To deploy a Network Load Balancer instead, apply the following annotation to your service:

```
service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

**Important**  
Do not edit this annotation after creating your service\. If you need to modify it, delete the service object and create it again with the desired value for this annotation\.

To deploy a load balancer to a private subnet, your service specification must have the following annotation:

```
service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```

For internal load balancers, your Amazon EKS cluster must be configured to use at least one private subnet in your VPC\. Kubernetes examines the route table for your subnets to identify whether they are public or private\. Public subnets have a route directly to the internet using an internet gateway, but private subnets do not\. 

For an example service manifest that specifies a load balancer, see [Type LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) in the Kubernetes documentation\. For more information about using Network Load Balancer with Kubernetes, see [Network Load Balancer support on AWS](https://kubernetes.io/docs/concepts/services-networking/service/#aws-nlb-support) in the Kubernetes documentation\.

## Load balancer – IP targets<a name="load-balancer-ip"></a>

NLBs with IP targets are created by the AWS Load Balancer Controller \(you cannot use CLBs with IP targets\)\. You can use NLB IP targets with pods deployed to Amazon EC2 nodes or Fargate\. Your Kubernetes service must be created as type `LoadBalancer`\. For more information, see [Type LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) in the Kubernetes documentation\.

To create a load balancer that uses IP targets, add the following annotation to a service manifest and deploy your service\. 

```
service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
```

**Important**  
Do not edit this annotation after creating your service\. If you need to modify it, delete the service object and create it again with the desired value for this annotation\. You can only use NLB *IP* targets with clusters running at least Amazon EKS version 1\.18\. To upgrade your current version, see [Updating a cluster](update-cluster.md)\.

**To deploy a sample application**

1. Deploy a sample application\.

   1. Save the following contents to a `yaml` file on your computer\.

      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: sample-app
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: nginx
        template:
          metadata:
            labels:
              app: nginx
          spec:
            containers:
              - name: nginx
                image: public.ecr.aws/z9d2n7e1/nginx:1.19.5
                ports:
                  - name: http
                    containerPort: 80
      ```

   1. Apply the manifest to the cluster\.

      ```
      kubectl apply -f <file-name-you-specified>.yaml
      ```

1. Create a service of type `LoadBalancer` with an annotation to create an NLB with IP targets\.

   1. Save the following contents to a `yaml` file on your computer\.

      ```
      apiVersion: v1
      kind: Service
      metadata:
        name: sample-service
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
      spec:
        ports:
          - port: 80
            targetPort: 80
            protocol: TCP
        type: LoadBalancer
        selector:
          app: nginx
      ```

   1. Apply the manifest to the cluster\.

      ```
      kubectl apply -f <file-name-you-specified>.yaml
      ```

1. Verify that the service was deployed\.

   ```
   kubectl get svc sample-service
   ```

   Output

   ```
   NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)        AGE
   sample-service  LoadBalancer   10.100.240.137   k8s-default-samplese-xxxxxxxxxx-xxxxxxxxxxxxxxxx.elb.us-west-2.amazonaws.com   80:32400/TCP   16h
   ```

1. Open the [Amazon EC2 AWS Management Console](https://console.aws.amazon.com/ec2)\. Select **Target Groups** \(under **Load Balancing**\) in the left panel\. In the **Name** column, select the target group's name that matches the name in the `EXTERNAL-IP` column of the output in the previous step\. For example, you'd select the target group named r `k8s-default-samplese-xxxxxxxxxx` if your output were the same as the output above\. The **Target type** is `IP` because that was specified in the sample service deployment manifest\.

1. Select the **Target group** and then select the **Targets** tab\. Under **Registered targets**, you should see three IP addresses of the three replicas deployed in a previous step\. Wait until the status of all targets is** healthy** before continuing\. It may take several minutes before all targets are `healthy`\. The targets may have an `unhealthy` state before changing to `healthy`\.

1. Send traffic to the service replacing the example value with the value returned in a previous step for the `EXTERNAL-IP`\.

   ```
   curl <k8s-default-samplese-xxxxxxxxxx-xxxxxxxxxxxxxxxx.elb.us-west-2.amazonaws.com>
   ```

   Output

   ```
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ...
   ```

1. When you're finished with the sample deployment and service, remove them\.

   ```
   kubectl delete service sample-service
   kubectl delete deployment sample-app
   ```
