---
title: "DNS, the DevOps way"
date: 2018-02-03
author: Reuben Honigwachs 
type: "post"
---
[Gareth Dwyer](https://twitter.com/sixhobbits) delivers entertaining talks one could describe as _tech talk standup_. I very much enjoy them. He's a brilliant software engineer and reignited my respect for technical writing. At a recent [DevOps Cape Town](https://devops.capetown/) meetup he presented a simple set of tasks and their realisation. In the end sparking a discussion with the the question: _"Is this the DevOps way?"_. 

As it is typical for backoffice applications, email plays a central role, which in turn means dealing with dangerously unusable web UI of rent-seeking local registrars. Uncertainty starts creeping up and how to get some DevOps in the mix.

Instead of [defining DevOps culture and practices](https://www.youtube.com/watch?v=uTEL8Ff1Zvk), let me set out two personal objectives I'd see for this case: 

* **Transparency.** Avoid not knowing who changed what when. 
* **Reliability.** Automate tests and deployments. 

I'm a fan of "infrastructure as code" where you have everything originate from a git repo and let automation take care of the deployment. On the _code_ side you reap the benefits in regards to permissions, review, change log, documentation. On the _automation_ side one enjoys determinism and repeatability. 

How could this look like for DNS? 

![CICD with DNS](https://reuben.honigwachs.de/img/dns-cicd.png)

In order to automate things you'll need DNS with an API. Next to running your own _BIND_ or _djbdns_, examples are: [Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html), [Cloudflare](https://api.cloudflare.com/#dns-records-for-a-zone-import-dns-records), [Dyn](https://help.dyn.com/dns-api-knowledge-base/) ... or **GCP Cloud DNS**. With all GCP services, including Cloud DNS, you get the holy trinity of [API](https://cloud.google.com/dns/docs/reference/v1/), [command line](https://cloud.google.com/sdk/gcloud/reference/dns/), and Web UI. 

Here _"DNS"_ is not _"registrar"_. You'd have your regisrar point to a DNS service of choice. 

In terms of CI/CD tooling everything is possible with GCP. I like [GitLab](https://about.gitlab.com/solutions/google-cloud-platform/) and [Travis](https://travis-ci.org/) very much and tend to opt for [Cloud Build](https://cloud.google.com/cloud-build/) which naturally works well with [GCP Cloud Source Repos](https://cloud.google.com/source-repositories/) where we will store the zone file for this example. 

For simplest implementation we assign the _DNS Administrator_ role to the [Cloud Build service account](https://cloud.google.com/cloud-build/docs/securing-builds/configure-access-control#service_account) within the project. Then we define a trigger for any push of the zone file to the master branch in the repo as follows:
 
![Screenshot of triger configuration](https://reuben.honigwachs.de/img/dns-build-trigger-conf.png)

The above referenced _cloudbuild.yaml_ looks as follows. Instead of assembling our own image we use the readily available _gcloud_ [cloud builder](https://github.com/GoogleCloudPlatform/cloud-builders) to clone the repo and import the zone file: 
```
steps:
- name: gcr.io/cloud-builders/gcloud
  args: ['source', 'repos', 'clone', 'honigwachs-info-zone']
- name: gcr.io/cloud-builders/gcloud
  args: ['dns', 'record-sets', 'import', 'honigwachs.info', '--zone=honigwachs-info', '--zone-file-format', '--delete-all-existing']
``` 

The tool `gcloud dns record-sets` is quite nifty and validates your input files at execution. Dependent on your testing strategy, this already might be sufficient. With `--zone-file-format` we can use the BIND format instead of yaml and make use of _bind9utils_ such as `named-checkzone` to test zone files. But _do_ note that in this example the `--delete-all-existing` flag is set to include SOA info and allow for a "pure" BIND experience when dealing with the zone file. 

Cloud Build publishes update messages to a [Cloud Pub/Sub](https://cloud.google.com/pubsub/) topic called `cloud-builds` which is automatically created when enabling the Cloud Build API. From there other actions such as notifications can be triggered. In general we're talking about a very low deploy time and once it's in GCP Cloud DNS we enjoy an astonishing propagation time we're used to from Google ;-).  

![Screenshot of cloud builder build time](https://reuben.honigwachs.de/img/dns-build-time.png)

Oh, yes, this is what a BIND zone file looks like: 
```
$ cat honigwachs.info 
honigwachs.info. 21600 IN NS ns-cloud-b1.googledomains.com.
honigwachs.info. 21600 IN NS ns-cloud-b2.googledomains.com.
honigwachs.info. 21600 IN NS ns-cloud-b3.googledomains.com.
honigwachs.info. 21600 IN NS ns-cloud-b4.googledomains.com.
honigwachs.info. 21600 IN SOA ns-cloud-b1.googledomains.com. cloud-dns-hostmaster.google.com. 1 21600 3600 259200 300

www.honigwachs.info. 300 IN A 151.101.1.195
www.honigwachs.info. 300 IN A 151.101.65.195

yetanother.honigwachs.info. 300 IN A 151.101.1.195
```

<small class="credits theme-by text-muted">[CC-BY-SA-4.0](https://creativecommons.org/licenses/by-sa/4.0/). Comment or improve on this note by filing an issue or PR [here](https://github.com/YaguraStation/notes/blob/master/devops-dns.md)</small><br />
<small class="credits theme-by text-muted">©2018 Google LLC, used with permission. Google and the Google logo are registered trademarks of Google LLC.</small>
