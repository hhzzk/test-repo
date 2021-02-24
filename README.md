# AWS Cluster Sizing

This page seeks to answer the question "How many compute resources (CPU, RAM,
disk space) should I provision for a customer of a given size?"

The amount of CPU, RAM, and disk space in AWS is determined by two things:

1. [The EC2 instance type](https://aws.amazon.com/ec2/instance-types/) and
2. the number of EC2 instances.

These are the two configurable values that you have to scale CXTM up or down
to adapt to customers of different sizes.

## Ways to scale

CXTM can scale in three ways:

1. By scaling the **app nodes** (which run the API and microservices)
    * When you have a lot of concurrent active users, you would need to scale **app nodes**.
2. By scaling the **job nodes** (which execute jobs)
    * When you have a lot of engineers running automated jobs, you would need to scale **job nodes**.

The `instance_type` and the number of instances (`size`) can be configured in
`infrastructure/aws/vars/prod.yml`:

```
eks_nodegroup:
  app:
    stack_name: cxtm-eks-cluster-prod-nodegroup-app
    name: cxtm-eks-nodegroup-app-prod
    instance_type: t3.small
    node_label: cx-tm.cisco.com=app
    size: 3
    size_min: 2
    size_max: 5
  job:
    stack_name: cxtm-eks-cluster-prod-nodegroup-job
    name: cxtm-eks-nodegroup-job-prod
    instance_type: t3.medium
    node_label: cx-tm.cisco.com=job
    size: 3
    size_min: 2
    size_max: 5
```

### App Node Scaling

NOTE: The following are estimates based on usage patterns of our internal
team, which consists of about 700 "active users" and 20 "automation
engineers". Scaling above or below that number is calculated by extrapolating,
which may not reflect real-world usage. These are simply baselines that can be
adjusted later as-needed.

NOTE: The dollar amount estimates below do not include the assumed 22% discount as would be observed from RunOn's billing dashboard. The estimated quarterly cost in each row represent the base AWS pricing. See [aws/billing](./billing.md) for more information about how the RunOn pricing relates to AWS base pricing.

| Active Users¹ | App Node Size     | Est. Quarterly Cost² |
|---------------|-------------------|----------------------|
| 1-1000        | 3x t3.small 30GG  | $147.27              |
| 1000-5000     | 3x t3.medium 30GB | $285.44              |

¹: "Active users" is defined as the number of unique people who will
log into CXTM in a given month. This number is a proxy for concurrent HTTP requests.

²: Costs are calculated using https://calculator.aws/ using "On Demand" pricing.
Note that AWS will discount EC2 costs about ~50% if the engagement is longer than 1 year:
https://aws.amazon.com/savingsplans/


### Job Node Scaling

| Automation Engineers³ | Est. concurrent jobs | Job Node Size      | Est. Quarterly Cost² |
|-----------------------|----------------------|--------------------|----------------------|
| 1-10                  | ~2                   | 3x t3.medium 300GB | $549.34              |
| 10-30                 | ~8                   | 3x t3.large 300GB  | $825.70              |
| 30-100                | ~20                  | 3x t3.xlarge 300GB | $1378.41             |

³: "Automation Engineers" is defined as the number of unique people who plan to execute
automated test cases in a given day. This number is a proxy for the number of concurrent jobs
that might be running at the same time. Each concurrent job requires 0.5 vCPU and 2 GB RAM.

### Additional Scaling Considerations

* If the engagement is expected to be longer than 3 years, you may want to
  split the database into its own node type to account for table growth.
* Job artifacts are stored in S3, which has modest charges per gigabyte
* Running many jobs that require lots of disk space may fill up the disk of the
  job nodes because it takes time for Kubernetes to do garbage collection. This
  will result in jobs going to the ERROR state and being "evicted". Consider
  adding more disk space or more job nodes if this occurs.
* Scaling beyond the numbers in the above tables might require some architectural
  changes. Please contact the dev team to double-check.
