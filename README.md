> NOTE: This is currently an incomplete work-in-progress

# AWS Well Architected with EKS

Kubernetes, even when run as the Amazon Elastic Kubernetes Service (EKS) on AWS, is nearly a full cloud in its own right. That is why people can run it on bare-metal within their own data centres - not just in public clouds like AWS. So, running EKS is rather like running a cloud on a cloud - with *some* integrations between the two.

Running your workloads on top of EKS, thus, has major implications for the AWS Well Architected Framework.

AWS hasn't (yet?) addressed these implications themselves - so this is my attempting to do so. 

> NOTE: This is my (Jason Umiker's) subjective view of this space. It doesn't come from AWS - though I do hope that they extend Well Architected with an EKS lens themselves someday soon.

## The Pillars

The AWS Well Architected Framework is made up of five pillars:
* Operational excellence
* Security
* Performance efficiency
* Cost optimization
* Sustainability

And within each pillar there are variety of best practices that AWS suggests. I have gone through this list to evaluate each of them with a Kubernetes/EKS lens in mind. I'll only comment on those where a workload being on Kubernetes/EKS might change the way you'd think about it vs. more AWS native offerings.

## Operational excellence

### OPS 4. How do you implement observability in your workload?

One area where EKS differs from most of the other AWS services is that there is no observability (metrics, logs or traces) enabled by default. The reason for that is that customers have a key decision to make - one that is strategic.

The three paths to observability with EKS are:
1. Use the traditional AWS native approaches (i.e. CloudWatch, CloudWatch Logs, X-Ray) - This approach is appropriate if all of your workloads are in AWS and so you can standardize on their native yet proprietary tooling. For metrics and logs AWS calls this CloudWatch Container Insights (https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html).
1. You can use the open-source vendor-agnostic tooling in this space (i.e. Prometheus, Elastic/OpenSearch, Jaeger) - This approach is approach is appropriate if you are trying to use consistent tooling both in and out of AWS. Prometheus and Jaeger are not just open-source but governed by the same vendor-neutral organization that governs Kubernetes - the Cloud Native Computing Foundation (CNCF). And, thanks in large part to AWS, ElasticSearch has been forked to be fully open-source in OpenSearch.
    1. AWS actually runs managed services for Prometheus as well as OpenSearch - giving you the best of both worlds (a managed service but based on a vendor-agnostic opensource tool)
        1. And at re:Invent this year they announced a tighter opt-in integration with their Managed Service for Prometheus with EKS - https://aws.amazon.com/about-aws/whats-new/2023/11/amazon-managed-service-prometheus-agentless-collector-metrics-eks/
1. You can use SaaS services in this APM/observability space (Sysdig Monitor, DataDog, AppDynamics, NewRelic, Elastic, Splunk, SumoLogic etc.) - This approach can work for both hybrid and multi-cloud environments - but it often involves being locked in to the SaaS vendor.


If you pick path 2 or 3 then you often will need to stream various metrics in from CloudWatch in order to have a "single pane of glass" with all your metrics - including those for the AWS foundational/managed services only available there.

Also, one of the key considerations for which path to choose is the changes that you'll need to make to your agents and your code - both to adopt it and if you ever choose to change in the future. For metrics, you often need to instrument your code with an SDK like CloudWatch or Prometheus. So changing tools means changes to your code (even though they are usually fairly minor/trivial). Many of the commercial SaaS offerings also have proprietary agents - so changing tools means changing agents too. That is changing, hopefully, with the OpenTelemetry agent - https://opentelemetry.io/ though. OpenTelemetry is a CNCF project focused on being one agent that can serve all your metrics, logging and tracing needs while being plugable to all the common backend tools/services.

### OPS 5. How do you reduce defects, ease remediation, and improve flow into production?

This set of AWS best practices is around how you handle version control, you do builds and deployments as well as do testing of your application(s). There are many best practices for EKS that would be consistent with how you managed AWS including:
* You should manage your infrastructure via code (IaC)
* You should version control that IaC in git
* You should check whether known vulnerabilities (CVEs) in your OS or code packages that you could patch before deployment
* You should have automated pipelines to build as well as deploy your application as well as its associated cloud infrastructure
* You should test your application and associated IaC by *actually* deploying them 'for real' in a non-production environment and testing the results rather than just with static tests and mocks before going to production

#### Declarative GitOps for Kubernetes

What is different when it comes to Kubernetes/EKS is that it is truly declarative (rather than imperative) - and the tooling that you use and the flow into the cluster often looks quite different than a more traditional AWS approach. The Kubernetes 'GitOps' flow is declarative rather than imperative and it is a pull in rather than push out deployment flow.

The imperative approach is to say "create a container with these characteristics". Even if you do the "right thing" and do that via IaC with CloudFormation or Terraform you can then change it via the AWS Console or API and it has drifted from your template(s). The declarative approach is to say "I want a container to exist with these characteristics". It might seem a subtle difference, but what it means is that Kubernetes has a control loop where if you change the results so they no longer match what you declared you wanted it will put them back rather than allow that drift to exist.

Everything in Kubernetes is a declarative YAML file with such a control loop. And, so, a pattern has emerged where there are tools to reconcile all this YAML between a git repo and your cluster(s) called **GitOps**. The two common tools in this space are [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) and [Flux](https://fluxcd.io/) - but what they have in common is:
* They run on your cluster(s) and **pull** in merged changes from git repos/branches rather than have a stage of your pipeline run a CLI to **push** them
    * They, therefore, move the security challenge from controlling access to the cluster (e.g. to be able to run `kubectl` to make changes) to controlling who can merge to the git repo (because whatever gets merged to particular repos/branches *will* be deployed immediately by the GitOps operator)
        * This often takes the form of requiring changes via Pull Request (PR) with a pre-merge approval peer review
* You - or things like the Horizontal Pod Autoscaler (HPA) - can end up 'fighting' the GitOps operator should you attempt to change things from what is explicitly defined in the YAML manifest in the git repo
    * This means that you need to omit certain parameters like a desired pod quantity in git so that it isn't something that GitOps will try to revert to as the HPA changes it based on your defined scaling rules
    * This also means that, in the event of an failed deployment or outage, you need to either disable the GitOps operator before making any changes directly to the cluster that diverge from what is in git or, better, to lean *into* it by pulling an old commit and re-merging it to get it to action that for you rather than fighting you
        * How to deal with failed deployments 'the right way' here is covered in OPS 6 below.

#### Container Vulnerability Scanning

We'll cover this more under the security pillar, but it is both advisable - as well as quite easy - to scan your container images for known vulnerabilities, or CVEs, in your build pipelines before they get pushed to your registry. This can be done with AWS Inspector as well as a number of 3rd party tools such as [Trivy](), [Docker Scout (built into the Docker CLI)](), [Synk Container]() and [Sysdig Secure]().

The hard bit is that, when the scanner inevitably finds them, to fix all of them by upgrading your base layers/images as well as packages in npm/pip/Maven/Nuget etc. before you deploy the workload.

### OPS 6. How do you mitigate deployment risks?

This set of best practices is about making automated deployments safe (usually with more automation). It suggests:
* Deployments should be automated
* There should then be automated testing of whether the deployment is successful - and automated roll-backs if it isn't
* That your deployments should be via mechanisms such as:
    * Blue/green - where you run the new and old version at the same time and only flip the traffic once the new version has tested successfully
        * This usually involves leaving the old version running for a period of time alongside the new version, even if it is no longer getting traffic, so you can fail back instantly
    * Canary - where you send a subset of your traffic and/or customers to the new version and gradually move more and more of it across to the new version in an automated and controlled way
        * This allows you to only impact a subset of your users with any problems

For Kubernetes, there are tools such as [Argo Rollouts](https://argo-rollouts.readthedocs.io/en/stable/) when using ArgoCD and [Flagger](https://fluxcd.io/flagger/) when using Flux that help to manage and automate things like blue/green or canary deployments declaratively. OPS 6 is saying that, should you embrace Kubernetes GitOps with those tools (ArgoCD or Flux) like we talked about for OPS 5, you should also embrace their associated deployment tooling to help mitigate any deployment risks.

#### Handling upgrades of Kubernetes/EKS itself
Well architected is usually focused on workloads rather than on the cluster(s) that they run on. But Kubernetes and EKS have a new version every quarter and the clusters also require regular upgrades too.

There are two approaches to that:
1. Do in-place upgrades - AWS has great APIs/tooling to allow you to do upgrade from one version to the next. There are some downsides to this approach though:
    1. The Kubernetes API will go down briefly during the upgrade process. All the workloads *will* keep running but you'll temporarily lose the ability to make changes or even for the workloads to heal in response to failed probes during the upgrade.
    1. You need to do every version in order and can't skip any. So if you find yourself 3-4 versions back you might not want to do the in-place upgrade shuffle through all of them.
1. You can spin up a new EKS cluster at the new version, deploy all your same workloads to it and then cut across to the whole new cluster in a blue/green fashion. There are downsides to this approach as well:
    1. It requires great and complete automation via Infrastructure-as-Code and ideally Kubernetes GitOps.
        1. As it can work especially well if you can point the new cluster's ArgoCD or Flux at the same git repo for it's K8s manifests and it will pull down identical workloads and configurations.
    1. It can work well if all of your workloads are stateless - but if you have workloads utilizing persistent EBS volumes then migrating them across to the workload on the new cluster can be challenging.
    1. It requires you to be using DNS rather than IPs for service discovery outside th cluster as well as leveraging operators so that the NLBs/ALBs for Ingress and the associated Route 53 DNS records are automatically created and updated so as to ensure traffic will work the same way without disruption.
    1. Using IAM Roles for Service Accounts (IRSA) used to be challenging with changing clusters as the OIDC provider to make that work was tied to each cluster - but AWS has just released the new EKS Pod Identity which should help there https://aws.amazon.com/blogs/aws/amazon-eks-pod-identity-simplifies-iam-permissions-for-applications-on-amazon-eks-clusters/

Both approaches can work well - and which one you choose will depend a bit on your situation (such as whether your workload are stateful). But, whichever path you choose, EKS upgrades are a regular and critical operation you'll need to plan and prepare for well for in order to achieve "Operational excellence".

## Security

### SEC 1. How do you securely operate your workload? 

#### EKS multi-cluster strategy (vs. AWS multi-account strategy)
In sub-question SEC01-BP01 AWS suggests that you have a multi-account strategy and separate AWS accounts for things like prod vs. non-prod as well as consider doing it for different levels of data sensitivity etc. You can make many of the same arguments for when you need a separate Kubernetes/EKS cluster. Though, to be fair, if you put a cluster in each of your AWS accounts in-line with a multi-account strategy you often do that anyway.

With EKS you an have multiple **Namespaces** where multiple workloads or environments can share a cluster. I would say that this is a much less 'hard' boundary than an AWS account, though - as, while it helps segment access to what you can change via the Kubernetes API, you can 'escape' out of the container boundaries on hosts that all the workloads share.