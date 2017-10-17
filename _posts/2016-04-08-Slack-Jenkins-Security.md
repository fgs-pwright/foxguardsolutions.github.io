---
layout: post
title: Securing Slack Authentication Tokens in Jenkins
image: /images/2016-04-08-Slack-Jenkins-Security/Slack_Mark_Web.png
author: Drew Budwin
excerpt: Adding the ability to safely pass Slack integration tokens as a secret to the Slack plugin for Jenkins.
---

![Cover]({{ site.baseurl }}/images/2016-04-08-Slack-Jenkins-Security/securityholes.jpg "Security Holes Everywhere")

For the uninitiated, [Jenkins](https://jenkins.io/index.html "Jenkins") is a free, extensible continuous integration (CI) web-based tool for building software.  It has hundreds of useful [plugins](https://wiki.jenkins-ci.org/display/JENKINS/Plugins "plugins") to make Jenkins perform a variety of tasks from providing you with [Chuck Norris quotes](https://wiki.jenkins-ci.org/display/JENKINS/ChuckNorris+Plugin "Chuck Norris quotes") to [running static analysis](https://wiki.jenkins-ci.org/display/JENKINS/Cppcheck+Plugin "running static analysis") on your software.

At FoxGuard Solutions, we are big fans of using [Slack](https://slack.com/ "Slack") to communicate.  We have Jenkins configured to send notifications to certain Slack channels to update us on the status of a build.  Jenkins will send messages to Slack that tell us if the build passed or failed and the results of running our test suite.  It is really useful for keeping our feedback loop tight to speed up development.  The sooner we know there is a problem, the quicker we can fix the problem.

Recently, we have had a big push to get better at [DevOps](https://en.wikipedia.org/wiki/DevOps "DevOps"), and a facet of that is [configuration as code](https://en.wikipedia.org/wiki/Infrastructure_as_Code "configuration as code").  By default, Jenkins is configured by a lot of check boxes and text fields on a webpage.  Behind the scenes, Jenkins converts these values in the web form into a [config.xml file](https://greasyfork.org/en/scripts/18092-jenkins-config-xml-file-viewer "config.xml file").  This system works, but is highly problematic for tracking and reviewing changes and creating a standard across all of our Jenkins jobs.

To support configuration as code for Jenkins, we have started using a plugin developed by Netflix called [Job DSL](https://github.com/jenkinsci/job-dsl-plugin "Job DSL").  This plugin allows us to define jobs in [Groovy](http://groovy-lang.org/learn.html "Groovy") files which are part of our source control.  Developers submit changes to Groovy files as pull requests which are subject to peer review prior to merging into the codebase.  Jenkins will detect these changes to the master branch using a Git hook which will trigger a seed job to run in Jenkins.  This seed job creates or updates jobs in Jenkins automatically based on the configuration defined in the Groovy files.

Jenkins comes preinstalled with the [Credentials plugin](https://wiki.jenkins-ci.org/display/JENKINS/Credentials+Plugin "Credentials plugin") to store secrets like username and password combinations.  There is another plugin called [Plain Credentials](https://wiki.jenkins-ci.org/display/JENKINS/Plain+Credentials+Plugin "Plain Credentials") which is an extension to the Credentials plugin that allows for storing tokens like API keys in a secret.  You can then later refer to the secrets using custom named IDs, making passing things like tokens and passwords safe.

Job DSL has pretty robust support for the common plugins people use in Jenkins.  If a plugin is not supported, it is possible to just write XML itself in the Groovy files that represent the equivalent XML in the `config.xml` file.  While creating a Groovy class to define a common usage of posting Slack messages across all of our Jenkins jobs, we had a difficult time getting it to work using credentials.  We had no issues getting GitHub working with credentials and figured it would be just as easy for Slack.  After all, you would not pass your GitHub secrets in plain text, why should you pass your Slack tokens in plain text?

After a lot of digging, we realized that the [Slack plugin](https://wiki.jenkins-ci.org/display/JENKINS/Slack+Plugin "Slack plugin") to Jenkins did not support the Plain Credentials plugin forcing all of the roughly 12,000 Slack plugin installations to store their Slack integration token in plain text in the `config.xml` file for a job.  FoxGuard Solutions is a cyber security company; it would be ironic if we let such an obvious vulnerability exist in our own environment.  Luckily, we are also very friendly to the open source community and found a quick way to make a contribution.

We forked the Slack plugin to our GitHub account so we could internally develop the feature before creating a pull request to the official [Slack plugin GitHub repo](https://github.com/jenkinsci/slack-plugin "Slack plugin GitHub repo").  The development was pretty straight forward.  It consisted of adding two dependencies to the `pom.xml` file (Credentials and Plain Credentials) then adding a new parmater to enter the ID for the token created using the Plain Credentials plugin which is decrypted at the last moment possible before the message gets posted to Slack.

Even though this feature is still an [open pull request](https://github.com/jenkinsci/slack-plugin/pull/208 "open pull request") on the Slack plugin GitHub page, it has already had a positive impact for other projects with [people referencing the commit](https://github.com/deis/jenkins-jobs/pull/27 "people referencing the commit") as a feature they want to use to improve their security.

#####Here is the step-by-step guide for making your Slack plugin more secure in Jenkins:
1. Make sure the Slack, Credentials, and Plain Credentials plugins are installed
2. Click "Credentials" on the left hand side of Jenkins
3. Click "Global credentials"
4. Click "Add Credentials" on the left
5. For the "Kind," choose "Secret text"
6. Enter the desired value of the "Secret"
7. Add a description if you would like
8. Before hitting "OK" hit the "Advanced..." button and enter in a name for the ID.  If you skip this step, it will generated a GUID for you that is harder to remember and non-descript.  While this step is optional, it is probably best to give it an ID.
9. Go to the configure page for a job
10. In the "Slack Notifications" section of the configuration, leave the "Integration Token" field blank and for the "Integration Token Credential ID" enter the ID name from step 8.
11. Press the "Test Connection" button; if it says "Success" you are all set
