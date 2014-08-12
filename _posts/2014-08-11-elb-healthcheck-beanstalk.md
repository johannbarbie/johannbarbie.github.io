---
layout: post
title: Application Level Health-Checks for Elastic Beanstalk
excerpt: "Configuring ELB health-check URLs for Auto Scaling Groups"
modified: 2014-08-11
tags: [ELB, Beanstalk, Auto-Scaling, AWS]
comments: true
image:
  feature: texture-feature-04.jpg
  credit: Texture Lovers
  creditlink: http://texturelovers.com
---

We use [AWS Elastic Beanstalk](http://blog.newrelic.com/2012/12/05/deploying-a-scalable-application-with-aws-elastic-beanstalk-and-new-relic/) to run some of our applications. Is is a great solution that simply takes your app and provides deployment, scaling, load balancing and health monitoring for it. It can even tie into one of Amazon's managed data stores. 

##Auto Scaling blind for Applications

Recently memory issues emerged and instances stared dying the OOM-death. I was happy to see that affected instances immediately turned unhealthy in the management console. When more instances died the environment transitioned from GREEN to YELLOW to RED and I was very surprised to see that environment grinded to a halt with the simple message "Elastic Load Balancer has zero healthy instances". Beanstalk never terminated the unhealthy instances nor did it provision new ones.

After some research on the forum I found [this post](https://forums.aws.amazon.com/message.jspa?messageID=330498) describing a similar issue. The load balancers use a custom health-check URL that usually points to the application itself. Auto Scaling Groups operate on the instance level and are agnostic to the application level. They use a dedicated EC2 health-check to verify that the instance is still operational. Naturally, it is no surprise that the JVM crash stayed unnoticed.

##Manually updating the Configuration

AWS Auto scaling groups offer an API to update the configuration and pull in the ELB health-check. It can be achieved like this:
<ol>
<li>Download <a href="http://aws.amazon.com/developertools/2535">auto scaling command line tools</a>.<br><br></li>
<li><a href="http://docs.aws.amazon.com/AutoScaling/latest/GettingStartedGuide/SetupCLI.html">Install tools</a><br><br></li>
<li>List all auto scaling groups with this command:
    {% highlight bash %}
          as-describe-auto-scaling-groups --region ap-northeast-1
    {% endhighlight %}</li>
<li>Update the desired auto scaling group with this command:
    {% highlight bash %}
          as-update-auto-scaling-group <group-name> 
              --health-check-type ELB 
              -grace-period 240  
              --region us-west-1
    {% endhighlight %}</li>
</ol>

##The Caveat

Unfortunately, those changes are not saved in the settings of the Beanstalk environment. Whenever an environment is rebuild the configuration will be gone, and those instructions will have to be executed again.