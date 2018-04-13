---
title: "Canary deployments of Lambda functions in Serverless apps"
description: "Never again fear breaking your Serverless application due to integration issues."
date: 2018-04-05
layout: Post
thumbnail: 'https://s3-us-west-2.amazonaws.com/assets.site.serverless.com/logos/serverless-square-icon-text.png'
authors:
  - DavidGarcia
---

Deployments in Serverless apps usually involves updating some of our Lambda functions, which is a one-step process where we replace our functions' code for new shiny versions. We build tests enough to be confident that we are not introducing any bug and we don't break anything. We even publish the update in different stages to check how it behaves in the cloud. However, a tingling runs down our spine every time we release to production, since we are not a 100% sure that we won't bump into an integration error or that we did not overlook any edge case. Fear no more, the [Canary Deployments Plugin](https://github.com/davidgf/serverless-plugin-canary-deployments) is your safety net.

## Lambda Weighted Aliases + CodeDeploy = Peace of mind

The AWS team recently introduced [traffic shifting for Lambda function aliases](https://docs.aws.amazon.com/lambda/latest/dg/lambda-traffic-shifting-using-aliases.html), which basically means that we can now **split the traffic** of our functions **between two different versions** of the same. We just have to specify the percentage of incoming traffic we want to direct to the new release, and Lambda will automatically load balance requests between versions when we invoke the alias. So, instead of completely replacing our function for another version, we could take a more conservative approach making the new code coexist with the old stable one and checking how it performs.

All this sounds great, but let's be honest, changing aliases weights and checking how new functions behave by ourselves is not going to make our lifes easier. Fortunately, AWS has, as usual, the service we need: **CodeDeploy**. [CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html) is capable of automatically updating our functions aliases weights according to our preferences. Not only so, but it will also roll back automatically if it notices that something goes wrong. Basically, our Lambda function deployments will be on autopilot.

So, let's implement all this in our Serverless application. We only need to create some AWS resources, but fortunately the Serverless Framework is integrated seamlessly with **CloudFormation**, so it shouldn't be difficult. We only need a CodeDeploy application, a DeploymentGroup and an Alias for each function, some new permissions here and there, replacing all the event sources to trigger the aliases instead of `$Latest`... Wait, it doesn't look so straightforward. If only there was a plugin for that...

## Integrating all this in your Serverless app

In fact, there is a plugin that makes gradual deployments simply a matter of configuration: the [Serverless Canary Deployments Plugin](https://github.com/davidgf/serverless-plugin-canary-deployments). In the next sections, we'll walk through the different options we can configure.

### Set up the environment

We'll create a simple Serverless service with one function exposed through API Gateway, making sure that we install the plugin. Our `serverless.yml` should look like this:

```yaml
service: sls-canary-example

provider:
  name: aws
  runtime: nodejs6.10

plugins:
  - serverless-plugin-canary-deployments

functions:
  hello:
    handler: handler.hello
    events:
      - http: get hello
```

Our function will simply return a message...

```javascript
module.exports.hello = (event, context, callback) => {
  const response = {
    statusCode: 200,
    body: 'Go Serverless v1.0! Your function executed successfully!'
  };

  callback(null, response);
};
```

... which we should get by calling the endpoint after deploying your service:

```shell
$ curl https://your-api-gateway-endpoint.com/dev/hello
Go Serverless v1.0! Your function executed successfully!
```

### Deploy the function gradually

This is where the fun starts. We'll tell the plugin to split the traffic between the last two versions of our function during the next deploy, and gradually shift more traffic to the new one until it receives all the load. There are three different fashions of gradual deployments:

* **Canary**: a specific amount of traffic is shifted to the new version during a certain period of time. When that time elapses, all the traffic goes to the new version.
* **Linear**: traffic is shifted to the new version incrementally in intervals until it gets all of it.
* **All-at-once**: the least gradual of all of them, all the traffic is shifted to the new version straight away.

All we need now is specifying the parameters and type of deployment, by choosing any of the [CodeDeploy's deployment preference presets](https://docs.aws.amazon.com/lambda/latest/dg/automating-updates-to-serverless-apps.html): `Canary10Percent30Minutes`, `Canary10Percent5Minutes`, `Canary10Percent10Minutes`, `Canary10Percent15Minutes`, `Linear10PercentEvery10Minutes`, `Linear10PercentEvery1Minute`, `Linear10PercentEvery2Minutes`, `Linear10PercentEvery3Minutes` or `AllAtOnce`. We'll pick `Linear10PercentEvery1Minute`, so that the new version of our function gets is incremented a 10% every minue, until it reaches the 100%. For that, we only have to set `type` and `alias` (the name of the alias we want to create) under `deploymentSettings` in our function:

```yaml
service: sls-canary-example

provider:
  name: aws
  runtime: nodejs6.10

plugins:
  - serverless-plugin-canary-deployments

functions:
  hello:
    handler: handler.hello
    events:
      - http: get hello
    deploymentSettings:
      type: Linear10PercentEvery1Minute
      alias: Live
```

If we now update and deploy the function...

```javascript
module.exports.hello = (event, context, callback) => {
  const response = {
    statusCode: 200,
    body: 'Wait, it is Serverless v1.26.11!'
  };

  callback(null, response);
};
```

... we'll notice that the requests are being load balanced between the two latest versions:

```shell
$ curl https://your-api-gateway-endpoint.com/dev/hello
Go Serverless v1.0! Your function executed successfully!
$ curl https://your-api-gateway-endpoint.com/dev/hello
Wait, it is Serverless v1.26.11!
```

### Make sure you don't break anything

Now our function is not deployed in a single flip, woohoo! So what? If we have to ensure that the whole system is behaving correctly by ourselves, we didn't really achieve anything impressive. However, we can add yet another AWS service to the mix to avoid it: **CloudWatch Alarms**. We can provide CodeDeploy with a list of alarms that would be tracked during the deployment process, cancelling it and shifting all the traffic to the old version if any of them turns into `ALARM` state. In this example, we'll monitor that our function doesn't have any invocation error, for which we'll make use of [A Cloud Guru's Alerts Plugin](https://github.com/ACloudGuru/serverless-plugin-aws-alerts):

```yaml
service: sls-canary-example

provider:
  name: aws
  runtime: nodejs6.10

plugins:
  - serverless-plugin-aws-alerts
  - serverless-plugin-canary-deployments

custom:
  alerts:
    dashboards: true

functions:
  hello:
    handler: handler.hello
    events:
      - http: get hello
    alarms:
      - name: foo
        namespace: 'AWS/Lambda'
        metric: Errors
        threshold: 1
        statistic: Minimum
        period: 60
        evaluationPeriods: 1
        comparisonOperator: GreaterThanOrEqualToThreshold
    deploymentSettings:
      type: Linear10PercentEvery1Minute
      alias: Live
      alarms:
        - HelloFooAlarm
```

The Canary Deployments plugin expects the logical ID of the CloudWatch alarms, which the Alerts plugin builds concatenating the function name, the alarm name and the string "Alarm" in Pascal case.
