# Concurrent Lambda

## Use Case
In the 2017 Re:Invent lecture [Become a Serverless Black Belt: Optimizing Your Serverless Applications](https://www.youtube.com/watch?v=oQFORsso2go), the speaker says that Lambda shouldn't be used for orchestration code since waiting time in Lambda is not efficient.  It is a much better fit for local compute bound operations like transforming data coming through Kinesis Firehose.  However, it would be nice to have the benefits of serverless scaling and resiliency when writing APIs that do a decent bit of waiting like 20 second calls to legacy systems.

## Business Case
In the past there has been an "insert code here" cloud PaaS built around VMs and containers, but not yet around functions as a service.
- **VM**: [Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk) and [Google App Engine Flexible](https://cloud.google.com/appengine/docs/flexible)
- **Container**: [Google App Engine Standard](https://cloud.google.com/appengine/docs/standard)
- **FaaS**: Sort of [Lambda](https://aws.amazon.com/lambda), but not efficient for all workloads.

As you go down the list, you get more responsive scaling and more efficient economics.  If AWS was to release the first FaaS-based PaaS, it could use that to differentiate itself from the other cloud providers.  Even though Lambda revenue might decrease in the short term, API Gateway would probably increase due to people hosting higher traffic APIs on Lambda.  Also more open source attention would be drawn to running Express, Hapi, and the other web/api frameworks on Lambda.

## User-Facing Configuration Options
1. **Max Parallel Requests**: The user would need to set a maximum parallel requests option.  The default of 0 would maintain the current behavior of a new Lambda invocation on each request.  Most cases would probably be between 10-20 parallel requests before you would want to scale to another Lambda invocation.
2. **Max Runtime**:
    1. Keep the current hard limit 5 minute max runtime and use the max duration parameter to limit a single event's processing time.  For example, with a 30 second max duration, the Lambda *could* (but not required to if all current requests had completed) continue to launch tasks up until the 4:30 mark.
    2. Keep the current parameter controlling the invocation runtime and introduce a new parameter limiting a single event's processing time.
