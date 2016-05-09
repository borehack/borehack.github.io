---
layout: post
title:  "Docker log to AWS CloudWatch"
date:   2016-05-09 07:30:00 +0000
---

How to log to [AWS Cloud Watch](https://aws.amazon.com/de/cloudwatch/) with [Docker](http://www.docker.com)?

follow the good description
https://docs.docker.com/engine/admin/logging/awslogs/

if you receive the following error message

docker: Error response from daemon: Failed to initialize logging driver: NoCredentialProviders: no valid providers in chain.

specify your credentials within the docker etc file.

/etc/default/docker

{% highlight bash %}
vi /etc/default/docker

export AWS_ACCESS_KEY_ID=<access key id>
export AWS_SECRET_ACCESS_KEY=<secret key>
export AWS_SHARED_CREDENTIALS_FILE=/root/.aws/credentials
{% endhighlight %}

the aws credentials file is only necessary if you don't specify the environment variables and vice versa.
