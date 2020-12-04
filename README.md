#Welcome to kubernetes-nodes-maintainer (KNM)

This repo contains an operator, internally named k25r, that will maintain
your nodes up to date for you.

I found the lack of configuration management for kubernetes lacking.
It's sad that everyone is re-using bash scripts to configure things inside
kubernetes. We could do better to maintain nodes in a kubernetes environment.

In other words: This is a "kubernetes native" configuration management
of a new generation, based on latest generation configuration management
instead of ugly idempotent shell scripts that are pushed into a daemonset.
It implements ONE SINGLE WAY (very opinionated) to include
flexible configuration management in a kubernetes world.

## How does it work?

Well, you most likely don't want to upgrade your nodes at the last moment
on your friday, and will avoid rebooting them during a peak period.
For that, KNM exposes a CRD, a NodeMaintenance.

NodeMaintenance contains:
* when you want to run operations on your machines (MaintenanceWindow)
* where do you want these operations to run (default to all the nodes) (NodeSeries)
* what operations (NodeOperations) will you run, and how frequent they run in
  the maintenance window (NodeOperationType).

The NodeOperationType defines the behaviour of a NodeOperation. It can be:
* "one-shot". This is the equivalent to one or many kubernetes jobs, on a
  specific set of machines.
* "cron". This is the equivalent to one or many kubernetes cronjobs, on a
  specific set of machines.
* "realtime". This has no kubernetes native equivalent for node operation,
  AFAIK. That's the interesting juice!

To avoid everyone reimplementing a cronjob "please run my apt-get update/upgrade
on monday morning", I have introduced a standard series of
NodeOperationsTemplates that get you up and running real quick.

While the one-shot and cron jobs can be easily done with kubernetes concepts
alone, the "realtime" required a different approach, as it is quite unique for
kubernetes world of _node management_.
(It's certainly not a new topic for configuration management).

To bring the same concepts of kubernetes "desired state" for your
nodes themselves, we will rely on kubernetes' etcd and its watchers,
associated with an agent-based configuration management (mgmt).

## Why do I need any of this?

- If you are used to automate your kubectl actions, create your (cron)jobs that
  way, that's good too. You then don't need to change. You are already used
  to work on your nodes. However, you have no ability to do "realtime" actions,
  and enjoy the same remediation loop that kubernetes has, on the nodes.
  Let's call this method the "kubernetes native configuration management".
  This method doesn't really manage the nodes, just do some operations
  at some point. Please note: Stop doing kubectl actions manually,
  use something like rancher's system-upgrade-controller instead.


- If you are used to have configuration management, continue configuring
  your nodes the way you are used to. It's good enough. It works.
  You might have to re-invent the wheel for some of this project content. But
  re-inventing in your configuration management is convenient, well integrated,
  fine grained to your situation. You might therefore not need this.
  Let's call this method the "traditional configuration management".
  This method suffer from one problem: tunnel vision.
  You will have to check what's going on within your configuration management
  to see what's happening on your nodes, and check in kubernetes logs to see
  what's happening in your cluster. Maybe it's fine because you forward all
  the logs to a central logging that regroups it all. But imagine now what
  you have to deal with: Some git repo somewhere, a CI, something that
  something that uses your git repo as source of truth for configuration
  management, your configuration management runner, your kubernetes deployment,
  your central logging. That's your software delivery.

What I am proposing here is the following:
- This project includes a configuration management, golang native based on
  mgmt, which integrates with kubernetes etcd.
- You define maintenances using kubectl.
  Kubectl is your interface to configuration management. It also means that
  you can use "gitops patterns" (git based definition of desired state).
- The operator creates the right policies/configuration management rules, and
  translates it for the agents on the node.
- The configuration management happens at the expected time, and the actions
  are logged into kubernetes, allowing the kubernetes operator to see
  everything.

## What's the downside of this

Yet another configuration management system...
So think about the two chefs in the same kitchen problem: this
doesn't prevent you to use anything else conflicting, so pay attention of
what you are running on your nodes!


## Why did I start this?

I had to manage a small kubernetes cluster, but large enough to not
do things manually. So I have automated the creation of a series of jobs and
cronjobs for my infrastructure. As I am fluent with Ansible,
I basically used an inventory fetching node data from kubeadm,
run a play on those.

All my logs are recorded in my ansible session. I needed to scale this
outside my laptop, so I now needed a runner. Enters AWX/Tower or any CI/CD
system. That plus central logging of course.
This was a hassle to setup for my environment. I wanted things more
self-contained, managed through kubernetes.

Also, I find that ansible is good for running jobs, but I find that
other configuration management tools like mgmt have some technical
advantages due to the inherent remediation features of an agent based
configuration management tool.

I also don't want to re-invent the wheel. So, behind the scenes, this
operator is just translating or sending everything the user asks for into
mgmt configuration management.

## Can it be used in a hybrid way: inside and outside kubernetes

YES! You will need to expose your kubernetes etcd to your machine outside
kubernetes (secure that traffic!) and run mgmt on it.
You can then adapt your NodeMaintenance objects in kubernetes and refer
to the external machine if necessary
