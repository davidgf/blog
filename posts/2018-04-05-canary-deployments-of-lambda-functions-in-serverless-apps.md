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

In fact, there is a plugin that makes gradual deployments simply a matter of configuration: the [Serverless Canary Deployments Plugin](https://github.com/davidgf/serverless-plugin-canary-deployments)


