> NOTE: This is currently an incomplete work-in-progress

# Applying the AWS Well Architected Framework to EKS

Kubernetes, even when run as the Amazon Elastic Kubernetes Service (EKS) on AWS, is nearly a full cloud in its own right. That is why people can run it on bare-metal within their own data centres - not just in public clouds like AWS. So, running EKS is rather like running a cloud on a cloud - with *some* integrations between the two.

Running your workloads on top of EKS, thus, has major implications for the AWS Well Architected Framework.

AWS hasn't (yet?) addressed these implications themselves (outside of the [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)) - so this is my attempt to do so. 

> NOTE: This is my (Jason Umiker's) subjective view of this space. It doesn't come from AWS - though I do hope that they extend Well Architected with an EKS lens themselves someday soon. And I am open to any suggestions via Issues/Pull Requests against this repository.

## The Pillars

The AWS Well Architected Framework is made up of six pillars:
* Operational excellence
* Security
* Reliability
* Performance efficiency
* Cost optimization
* Sustainability

And then within each pillar there are variety of best practices that AWS suggests. I have gone through this list to evaluate each of them with a Kubernetes/EKS lens in mind. I'll only comment on those where a workload being on Kubernetes/EKS might change the way you'd think about it vs. a more native AWS approach (so you can assume if it isn't listed here that I consider the traditional AWS WAR guidance sufficient).

## Operational excellence

### OPS 4. How do you implement observability in your workload?

This set of best practices are focused around implementing observability in your workload so that you can understand its state and make data-driven decisions based on business requirements.

One area where EKS differs from most of the other AWS services is that there is no observability (metrics, logs or traces) enabled by default. The reason for that is that customers have a key decision to make - one that is strategic.

The three possible paths to observability with EKS are:
1. Use the traditional AWS native approaches (e.g. CloudWatch, CloudWatch Logs, X-Ray) - This approach is appropriate if all of your workloads are in AWS and so it makes sense to standardize on their native, yet AWS-proprietary, observability services. For metrics and logs AWS calls this approach CloudWatch Container Insights (https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html).
1. You can use the open-source vendor-agnostic tooling in this space (e.g. Prometheus, Elastic/OpenSearch, Jaeger) - This approach is approach is appropriate if you are trying to use consistent tooling both in and out of AWS. And Prometheus and Jaeger are not just open-source, but also governed by the same vendor-neutral organization that governs Kubernetes - the [Cloud Native Computing Foundation (CNCF)](https://www.cncf.io). And, [thanks in large part to AWS](https://aws.amazon.com/blogs/opensource/stepping-up-for-a-truly-open-source-elasticsearch/), ElasticSearch has been forked to be truly open and free in [OpenSearch](https://opensearch.org).
    1. AWS actually runs managed services for Prometheus as well as OpenSearch - giving you the best of both worlds (a managed service but based on a vendor-agnostic opensource tool and standard that you *could* run yourself anywhere)
        1. And at re:Invent this year they announced a tighter opt-in integration with their Managed Service for Prometheus with EKS - https://aws.amazon.com/about-aws/whats-new/2023/11/amazon-managed-service-prometheus-agentless-collector-metrics-eks/
1. You can use non-AWS commercial SaaS services in this observability space (Sysdig Monitor, Datadog, AppDynamics, NewRelic, Elastic, Splunk, SumoLogic etc.) - This approach can work for both hybrid and multi-cloud environments but at a cost of both money and a degree of lock-in.

Also, one of the key considerations for which path to choose is the degree of changes that you'll need to make to your agent(s) and your code. For metrics and traces, you often need to instrument your code with an SDK like CloudWatch and X-Ray or Prometheus and Jaegar. So changing tools means changes to your code (even though they are usually fairly minor/trivial). Many of the commercial SaaS offerings also have proprietary agents - so changing tools means changing agents too. This cost-to-change is being mitigated with the new OpenTelemetry agent/project - https://opentelemetry.io/ though. OpenTelemetry is a CNCF project focused on being one agent and SDK that can serve all your metrics, logging and tracing needs while being plugable to all the common backend tools/services.

Finally, this observability data is often needed by automation such as the Kubernetes Horizontal Pod Autoscaler (HPA) (to know when to scale your workload in/out) or by tools like Argo Rollouts or Flagger (to know when a deployment has issues and should be automatically rolled back). So, understanding what tools you plan on using and what observability backends they will need to integrate with is a consideration here as well.

### OPS 5. How do you reduce defects, ease remediation, and improve flow into production?

This set of best practices are focused on adopting approaches that improve flow of changes into production, that activate refactoring, fast feedback on quality, and bug fixing. These accelerate beneficial changes entering production, limit issues deployed, and achieve rapid identification and remediation of issues introduced through deployment activities.

What both AWS-native and Kubernetes/EKS has in common here are that you should:
* Manage your infrastructure via code (IaC)
* Version control both your code as well as the associated IaC in a source repository like git
* Check whether known vulnerabilities (CVEs) in your code, the opensource packages you use from npm/pip/maven/nuget etc. or the OS base layers of your container images
* Have automated pipelines to build as well as deploy your application ideally along with any cloud infrastructure on which it depends
* Test that your application and associated IaC work by *actually* deploying them 'for real' in a non-production but production-like integration environment (rather than just with static tests and mocks) before going to production

But there are some differences in both tooling and mindset when it comes to having a "best practice" Kubernetes deployment flow vs. native AWS.

#### Declarative GitOps for Kubernetes

Firstly, Kubernetes/EKS is different in that it is truly declarative rather than imperative like AWS. The imperative approach is to say "create a container with these characteristics right now". If you do that with CloudFormation or Terraform then you can then change that resource via the AWS Console or API - in which case it has drifted from your IaC. The declarative approach is to say "I want a container to exist with these characteristics". It might seem a subtle difference, but Kubernetes has a control loop where if you change the results so they no longer match what you declared you wanted it will put them back rather than allow any drift to exist.

Also, in Kubernetes everything is a declarative YAML IaC file. Even if you make what seems like an imperative command with `kubectl` it is just generating the IaC file and submitting it for you. The analogy with native AWS would be to imagine if you could only managed AWS with CloudFormation - where even the AWS Console and CLI just generated and submitted CloudFormation on your behalf - and AWS continually reverted any resource to match the CloudFormation template you declared if you tried to change it directly.

This difference allows for some significant changes in flow and tooling. This 'best pratice' flow in Kubernetes is called **GitOps**.

![](https://codefresh.io/wp-content/uploads/2023/06/Basic-GitOps-workflow-for-Kubernetes.png)

The two common Kubernetes GitOps tools are [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) and [Flux](https://fluxcd.io/). What they have in common is:
* They run on your cluster(s) and **pull** in merged changes from git repos/branches (rather than have a stage of your pipeline run a CLI command to **push** them)
    * They, therefore, move the security challenge from controlling access to the cluster (e.g. authx of `kubectl` to make the required changes) to controlling who can merge to the git repo - because whatever gets merged to particular repos/branches *will* be deployed immediately by the GitOps operator
        * This often takes the form of requiring changes via Pull Request (PR) with a pre-merge testing and approval flow
* You - or things like the Horizontal Pod Autoscaler (HPA) - can end up 'fighting' the GitOps operator should you attempt to change things from what is explicitly defined in the YAML manifest in the git repo (as it'll keep trying to change it back to what is in git)
    * This means that you need to omit certain parameters like a desired pod quantity in git so that it isn't something that GitOps will try to revert to as the HPA changes it based on your defined scaling rules
    * This also means that, in the event of an failed deployment or outage, you need to either disable the GitOps operator before making any changes directly to the cluster or, better, to lean *into* it by making any changes via new git
        * This means that to do a rollback you pull and then re-merge the old commit so it is the latest for the branch so that the GitOps operator deploys it
            * More on how to deal with failed deployments 'the right way' here is covered in OPS 6 below.

#### Container Vulnerability Scanning

We'll cover this in more detail under the security pillar, but it is both advisable - as well as quite easy - to scan your container images for known vulnerabilities, or CVEs, in your build pipelines *before* they get pushed to your registry or deployed. This can be done with AWS Inspector as well as a number of 3rd party tools such as [Trivy](https://github.com/aquasecurity/trivy), [Docker Scout (built into the Docker CLI)](https://docs.docker.com/scout/), [Synk Container](https://snyk.io/product/container-vulnerability-management/) and [Sysdig Secure](https://sysdig.com/solutions/vulnerability-management/).

### OPS 6. How do you mitigate deployment risks?

This set of best practices is focused on adopting approaches that provide fast feedback on quality and achieve rapid recovery from changes that do not have desired outcomes. Using these practices mitigates the impact of issues introduced through the deployment of changes.

Things that both Kubernetes and AWS have in common here are that:
* You should plan for unsuccessful deployments
* Deployments should be automated
* There should be automated testing to determine if deployments are successful
* Deployments should be via mechanisms/approaches such as:
    * **Blue/green** - where you run the new and old version at the same time and only flip the traffic once the new version has tested successfully
        * This usually involves leaving the old version running for a period of time alongside the new version, even if it is no longer getting traffic, so you can fail back instantly
    * **Canary** - where you send a subset of your traffic and/or customers to the new version and gradually move more and more of it across to the new version in an automated and controlled way
        * This allows you to only impact a subset of your users with any problems

#### Argo Rollouts and Flux Flagger

For Kubernetes, there are two tools dedicated to helping you achieve these goals that align with which of the GitOps operators that you choose - [Argo Rollouts](https://argo-rollouts.readthedocs.io/en/stable/) when using ArgoCD and [Flagger](https://fluxcd.io/flagger/) when using Flux. OPS 6 is saying that, should you embrace Kubernetes GitOps with those tools (ArgoCD or Flux) like we talked about for OPS 5, you should also embrace their associated deployment tooling to help mitigate any deployment risks.

These tools will do things like:
* Facilitate automated **blue/green** (run both the old and new version and flip the traffic seamlessly back/forth) or **canary** deployments (gradually shift the traffic to thew new version)
* Monitor and validate service level objectives (SLOs) like availability, error rate percentage, average response time and any other objective based on app specific metrics around the deployment - and automatically roll back deployments that fail these metrics with minimal impact to end-users
    * This requires you to have configured an appropriate observability tool as described in OPS 4 from which Rollouts/Flagger will get these metrics

![](https://d2908q01vomqb2.cloudfront.net/fc074d501302eb2b93e2554793fcaf50b3bf7291/2021/12/13/Fig4-canary.png)
(image from the AWS Blogpost https://aws.amazon.com/blogs/architecture/use-amazon-eks-and-argo-rollouts-for-progressive-delivery/)

#### Handling upgrades of Kubernetes/EKS itself
Well architected is usually focused on workloads rather than on the cluster(s) that they run on - but Kubernetes/EKS has a new version every quarter and the clusters also require regular upgrades as part of their ongoing operation as well. And those cluster upgrades can have impacts on the performance and availability of the apps running on them.

There are quite a few considerations here and AWS has documented it all quite well in their [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/) - so I'll refer you to the [relevant section](https://aws.github.io/aws-eks-best-practices/upgrades/) of that for more details.

## Security
