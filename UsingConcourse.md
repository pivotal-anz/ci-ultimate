# Using Concourse

# What is it?

Concourse is really just a flow definition tool a bit like Spring Cloud Web Flow (Spring XD) except that instead of a DSL it uses a YAML description file. Although the flow can be visualised, once specified, in the cool web GUI, you can't define it that way.

I personally found the documentation was not really clear until you knew what it was trying to explain, so here is a quick overview on how to use Concourse.

# Getting Started

## Local VM

To run Concourse locally you need Vagrant and Virtual Box installed.

1. It runs as a local VM using a Vagrant spec. In any directory run:
 
   ```
   vagrant init concourse/lite # creates ./Vagrantfile
   vagrant up                  # downloads the box & spins up the VM
   ```

1. You can then access its web interface in any browser at [http://192.168.100.4:8080](http://192.168.100.4:8080)

1. Finally download the Fly CLI from the main page where there are links to binaries for common platforms.  Save it in a sensible place.

1. `fly` is an executable so on Mac or Linux run `chmod a+x` to make it executable.  The Windows version is `fly.exe`, so you can skip this step.

1. Add `fly` to your `PATH` (on a Mac or Linux machine you can just copy it into `/usr/local/bin`).

1. By default, `fly` alawys tries to _target_ the local machine, so there is nothing more to do.

## Concourse at AWS

A concourse instance is running on AWS at ec2-54-86-210-135.compute-1.amazonaws.com.

1. You can access the dashboard at http://ec2-54-86-210-135.compute-1.amazonaws.com:8080.

1. There is the same option to download `fly` or, if you have `fly` already run `fly --target "http://ec2-54-86-210-135.compute-1.amazonaws.com:8080" sync`.  Either way a copy of `fly` is downloaded.  It
takes a while, so be patient.
 
1. If you downloaded `fly` from the console (instead of running `sync`):

   1. `fly` is an executable so on Mac or Linux run `chmod a+x` to make it executable.  The Windows version is `fly.exe`, so you can skip this step.

   1. Add `fly` to your `PATH` (on a Mac or Linux machine you can just copy it into `/usr/local/bin`).

1. Like Cloud Foundry's `cf` utility, you need to target your concourse VM however it doesn't work the same way.  You have to add a `-target URL` to every `fly` command, which is tedious, or save it like this:
 
    fly --target "http://ec2-52-7-64-81.compute-1.amazonaws.com:8080" configure save-target aws

No you can run commands like `fly -t aws ....`.  To target locally run `fly -t local ...`.

## Using Concourse

The Concourse [Getting Started](http://concourse.ci/getting-started.html) guide now gets you to create a YAML flow definition and run it, which you can do now if you wish.   However I think it helps to understand what Concourse does first and _then_ run a flow.

# Concepts

A flow, which Concourse calls a Job, consists of Tasks and Resources.

## Tasks

A task is simply a specification of work to run.  Most tasks are Unix scripts and or executables.  A task is specified by a YAML file.  This project has a task defined in `ci/ci-task.yml`:

```
---
platform: linux

image: docker:///java#8-jdk

inputs:
- name: cf-spring-trader-repo
- name: ci-ultimate-repo

run:
  path: ci-ultimate-repo/scripts/ci-test
```

Note that the task is actually executed by a Docker container, so I guess the Concourse VM has Docker installed internally.

There are many, many images at http://dockerhub.com, we are using a linux VM running Java V8 since we want to run continuous intgration tests on _Spring Trader_ which is a Java application

This task runs the `ci-test` script in `ci-ultimate-repo`.  Which brings us to resources.

## Resources

Resources are inputs and outputs.  Typically a Concourse flow is defined as a project on github and
is used as a resource for itself, like here.

The task above defines two input resources `ci-ultimate-repo` (this project) and `cf-spring-trader-repo`
(what we want to build, test and deploy).  We will deploy to Cloud Foundry which is an _output_ resource.

A resource is specified as part of the job flow like this:

```
resources:
- name: ci-ultimate-repo
  type: git
  source:
    uri: https://github.com/pivotal-anz/ci-ultimate
    branch: master
``` 

To keep things simple for now, let's just assume this Github repository is public, like this one.

The script we want to run is in `ci-ultimate-repo/scripts`.

Thus to modify the task, modify `ci-ultimate-repo/scripts/ci-test`, push the change and rerun the job.

Several types of predefined resources are provided by Concourse, including `git`.  If you look at the Concourse [github project](https://github.com/concourse?query=resource) you will see several resource sub-projects for both input (getting docker images, pulling from git, fetching data from Amazon S3) and output (pushing to Cloud Foundry, saving to S3).

## Jobs

We combine the resource and its task into a Job like this:

```
resources:
- name: ci-ultimate-repo
  type: git
  source:
    uri: https://github.com/pivotal-anz/ci-ultimate
    branch: master

jobs:
- name: ci-pipeline
  - get: ci-ultimate-repo  # Fetch the resource
    trigger: true  # Rerun automatically if repo changes
  - task: unit     # Run the unit test
    file: ci-ultimate-repo/ci/test-task.yml
```

Note:

1. `ci-ultimate-repo` contains both the test script (`ci-ultimate-repo/scripts/test`) and the task (`ci-ultimate-repo/ci/ci-task.yml`) to run it.
1. The task name `unit` is arbitrary.  There are _no_ predefined tasks.

A job consists of one or more "steps".  Just a few of the predefined job-steps are:

1. `get`: fetch a resource
1. `put`: update a resource
1. `task`: execute a task

Suppose our test script outputs to a log file `test-out.log`.  We can add a step to view it.  I have also modified the script to use my github id (paulc4):

```
resources:
- name: ci-ultimate-repo
  type: git
  source:
    uri: https://github.com/pivotal-anz/ci-ultimate
    branch: master

jobs:
- name: ci-pipeline
  plan:
  - get: ci-ultimate-repo
    trigger: true
  - task: unit
    file: ci-ultimate-repo/ci/ci-task.yml
```

This is the file `ci.yml`.

# Running a Job

From a command line (Terminal or CMD window), run:

```
fly -t aws configure ci-pipeline -c ci.yml --paused=false
```

You will be prompted with `apply configuration? (y/n)` - just answer `y`.  This sets up the new Job as a pipeline and makes it runable (`paused=false`).

__Note:__ Make sure the name of the pipeline specified in `fly configure` is the same as the name in the YAML file.  They don't have to be the same, it's just annoyingly confusing if they aren't.

Once `fly` has setup the pipeline, it should also tell you the URL to use to view the pipeline [http://192.168.100.4:8080/pipelines/cipipeline](http://192.168.100.4:8080/pipelines/ci-pipeline).

Open this URL now and you should see:

![Pipeline Page](https://github.com/paulc4/my-repo/blob/master/screenshots/pipeline-page.png)

Click on the grey-box and you see this:

![Pipeline Jobs Page](https://github.com/paulc4/my-repo/blob/master/screenshots/pipeline-jobs-page.png)

Click the + button on the right to run the job (run the flow).

It will go orange (running) and then green (succeeded).  If it goes red, the job failed.

The run number (#1) will have appeared and so will the steps.  Click on ci-ultimate-repo and unit to make them show their output.  You should see this:

![Jobs Output](https://github.com/paulc4/my-repo/blob/master/screenshots/job-output.png)

## Using the Web Interface

Takes a bit of getting used to.  The Home (house) icon takes you to the home page for the _current pipeline_.  Even going explicitly to [http://192.168.100.4:8080/](http://192.168.100.4:8080/) shows you the last pipeline used.

Once you have multiple pipelines you can see them by clicking on the tripple bar icon at the top _left_ (next to the house icon).  Select the pipeline you want.

The other triple bar icin (on the top _right_) shows you the builds that you have run.

# Online Documentation

This can be found here [http://concourse.ci](http://concourse.ci).  In case you are wondering, the Internet domain `ci` belongs to the Ivory Republic (Cote d'Ivoire in French).

I suggest looking at a complete pipeline first [http://concourse.ci/pipelines.html](http://concourse.ci/pipelines.html) to see a complete worked example.  Then work through the guide starting at [Getting Started](http://concourse.ci/getting-started.html).

Note that the Getting Started pipeline uses a special case of a task that embeds what it does instead of using an YAML file as described above, like this:

``` 
jobs:
- name: hello-world
  plan:
  - task: say-hello
    config:
      platform: linux
      image: "docker:///busybox"
      run:
        path: echo
        args: ["Hello, world!"]
```

The `say-hello` task is defined using the `config` sub-element instead of a YAML file.  Hence this flow requires no resources.

As a result, this flow appears in the Web GUI as a single grey box which doesn't look like a flow at all (since it has no input or output resources).

Whilst this is a nice simple first example, it is not typical and, personally, I found it more confusing than helpful.












 
