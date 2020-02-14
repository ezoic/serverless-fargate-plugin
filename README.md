# serverless-fargate-plugin

Based on templates found here: https://github.com/nathanpeck/aws-cloudformation-fargate

#### About
This plugin will create a cluster, load balancer, vpc, subnets, and one or more services to associate with it. This plugin implements the Public VPC / Public Load Balancer / Public Subnet approach found in the templates above.

If you would like to reference the VPC elsewhere (such as in the [serverless-aurora-plugin](https://github.com/honerlaw/serverless-aurora-plugin)). The VPC will be called `VPC{stage}` where `{stage}` is the stage in the serverless.yml. The subnets will be called `SubnetName{stage}{index}` where `{stage}`is the stage in the serverless.yml, and `{index}` references the index of the subnet that was specified in the subnets array. *THESE ARE NOT ADDED TO OUTPUT*. So you can only reference them in the same serverless.yml / same cf stack.

#### Notes
- This implements a Public VPC / Public Subnet / Public Load Balancer approach, I may in the future add the other approaches but for now this suites my personal needs.
- This plugin only supports AWS
- Docker image must be built / uploaded / and properly tagged
- It is assumed that the process running in the docker container is listening for HTTP requests.

#### TODO
- Tests
- Better TS Definitions
- Option to not use ELB
- Add custom tagging
- Auto certification trhough plugin https://github.com/schwamster/serverless-certificate-creator
- More options

#### Options
```javascript
{
    executionRoleArn?: string; // execution role for services, generated if not specified
    vpc: {
        //if this options are specified it will create a VPC
        cidr: string;
        subnets: string[]; // subnet cidrs
        //If this options are specified it will attach to existing VPC.
        //all of then are required, if one missing it will turn to self-created 
        //VPC as described above
        vpcId: string;
        securityGroupIds: string[]
        subnetIds: string[]
    };
    services: Array<{
        name: string; // name of the service
        cpu: number;
        memory: number;
        public: boolean; //Will it be facing internet? This affects directly what security groups will be auto created
        port: number; // docker port (the port exposed on the docker image) - if not specified random port will be used - usefull for busy private subnets 
        entryPoint: string[]; // same as docker's entry point
        environment: { [key: string]: string }; // environment variables passed to docker container
        protocols: Array<{
            protocol: "HTTP" | "HTTPS";
            certificateArns?: string[]; // needed for https
        }>;
        autoScale: {
              min?: number; //default to 1
              max?: number; //default to 1
              metric: AutoScalingMetricType;
              cooldown?: number; //defaults to 30
              cooldownIn?: number; //defaults to cooldown but has priority over it
              cooldownOut?: number; //defaults to cooldown but has priority over it
              targetValue: number;
        }
        image?: string; // full image name, REPOSITORY[:TAG]
        imageRepository?: string; // image repository (used if image option is not provided)
        imageTag?: string; // image tag (used if image option is not provided)
        priority?: number; // priority for routing, defaults to 1
        path?: string; // path the Load Balancer should send traffic to, defaults to '*'
        desiredCount?: number; // number of tasks wanted by default - if not specified defaults to 1
        taskRoleArn?: string;
        healthCheckUri?: string; // defaults to "/"
        healthCheckProtocol?: string; // defaults to "HTTP"
        healthCheckInterval?: number // in seconds, defaults to 6 seconds
    }>
}
```

#### Examples
```yaml
service: example-service

provider:
  name: aws
  region: us-east-1
  stage: example

plugins:
- serverless-fargate-plugin

custom:
  fargate:
    clusterName: Test
    vpc:
      cidr: 10.0.0.0/16
      subnets:
      - 10.0.0.0/24
      - 10.0.1.0/24
    services:
    - name: example-service-name
      cpu: 512
      memory: 1024
      port: 80
      healthCheckUri: /health
      healthCheckInterval: 6
      imageTag: 1.0.0
      imageRepository: xxx.amazonaws.com/xxx
      autoScale:
        min: 1
        max: 10
        cooldownIn: 30
        cooldownOut: 60
        metric: ECSServiceAverageCPUUtilization
        targetValue: 75
      entryPoint:
      - npm
      - run
      - start
      environment:
        PRODUCTION: true
      protocols:
      - protocol: HTTP
      - protocol: HTTPS
        certificateArns:
        - xxxx

```

####Outputs
  For the configuration above CF will have the reference `ECSTestClusterExampleServiceNameServiceHTTP`
