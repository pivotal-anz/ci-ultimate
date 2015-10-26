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

1. You can then access its web interface in any browser at [http://192.168.100.4:8080](http://192.168.100.4:8080). It should look like this:

![Initial Home Page](https://github.com/pivotal-anz/ci-ultimate/blob/master/screenshots/no-pipelines.png)

1. Next download the Fly CLI from the main page where there are links to binaries for common platforms.  Save it in a sensible place.

1. `fly` is an executable so on Mac or Linux run `chmod a+x` to make it executable.  The Windows version is `fly.exe`, so you can skip this step.

1. Add `fly` to your `PATH` (on a Mac or Linux machine you can just copy it into `/usr/local/bin`).

1. By default, `fly` alawys tries to _target_ the local machine, so there is nothing more to do.

## Concourse at AWS

A concourse instance is already setup at AWS at ec2-52-7-64-81.compute-1.amazonaws.com.

> To create an instance, use the EC2 dashboard to create a new instance (click `Launch Instance`).  Next, on the left, select > Community AMIs, enter `Concourse` in the search box, hit `Enter` and select one of the existing Concourse images - we used > `concourse-0.61.0` for the instance referred to in these notes.

1. You can access the dashboard at http://52.7.64.81:8080 or http://ec2-52-7-64-81.compute-1.amazonaws.com:8080.

1. There is the same option to download `fly` **but** if you have `fly` already *you must* run `fly --target "http://ec2-52-7-64-81.compute-1.amazonaws.com:8080" sync` instead (or it won't work properly later; the `fly`
utility seems to be intimately linked to the Concourse installation it came from).  Either way a copy of `fly` is
downloaded which takes a while.  Using `sync` takes an _excruciatingly_ long time to run, so just be patient.
 
1. If you downloaded `fly` from the console (instead of running `sync`):

   1. `fly` is an executable so on Mac or Linux run `chmod a+x` to make it executable.  The Windows version is `fly.exe`, so you can skip this step.

   1. Add `fly` to your `PATH` (on a Mac or Linux machine you can just copy it into `/usr/local/bin`).

1. Like Cloud Foundry's `cf` utility, you need to target your concourse VM however it doesn't work the same way.  You have to add a `-target URL` to every `fly` command, which is tedious, or save it like this:
    ```
    fly save-target --api "http://ec2-52-7-64-81.compute-1.amazonaws.com:8080" aws
    fly save-target --api "http://192.168.100.4:8080" local
    ````
Now you can run commands like this `fly -t aws ....`.  To target locally run `fly -t local ...`.

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

**Important Note:** Although the repository (project) on github is called `ci-ultimate`, because we have called the resource `ci-ultimate-repo`, it is cloned onto the Concourse VM into a directory also called `ci-ultimate-repo`.  Thus the script we want to run is `ci-ultimate-repo/scripts/ci-test`.

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

### Integration with Cloud Foundry

A `put` task can be used to push an application to Cloud Foundry.  The Cloud Foundry Foundation to use is also specified as
a resource, like this:

```
resources
- name: pcf-bedazzle
  type: cf
  source:
    api: <foundation>
    user: <user-name>
    password: <passowrd>
    organization: <org>
    space: <space>
    skip_cert_check: true
```

The `api` property is the public URL of the Cloud Controller API - for example `https://api.run.pivotal.io` to target PWS.

Then create a task to `put` to this resource - see next section.

### The Complete Job

This is defined by the file `ci.yml`:

```
# A flow that uses a Gradle Docker image to run integration tests and if successful 
# pushes the application to PCF
resources:
- name: cf-spring-trader-repo
  type: git
  source:
    #uri: https://github.com/cf-platform-eng/springtrader-cf
    uri: https://github.com/pivotal-anz/cf-SpringBootTrader
    branch: master
- name: ci-ultimate-repo
  type: git
  source:
    uri: https://github.com/pivotal-anz/ci-ultimate
    branch: master
- name: pcf-bedazzle
  type: cf
  source:
    #api: https://api.apj.fe.pivotal.io
    #username: fadzi
    api: https://api.pcf-demo.com
    user: ci@pcf-demo.com
    password: demo
    organization: Bedazzle
    space: development
    skip_cert_check: true

jobs:
- name: ci-pipeline
  plan:
  # Clone this repository
  - get: ci-ultimate-repo
    trigger: true
  # Clone the application to test
  - get: cf-spring-trader-repo
    trigger: true
  # Run the unit test
  - task: unit
    file: ci-ultimate-repo/ci/ci-task.yml
  # Push to PCF
  - put: pcf-bedazzle
    params:
      manifest: cf-spring-trader-repo/manifest-ci.yml
```

Job summary
 1. `get: ci-ultimate-repo` Fetch this repository from git hib
 1. `get: cf-spring-trader-repo` Fetch Spring Trader repository from github.
 1. `task: unit` Execute the task `ci-ultimate-repo/ci/ci-task.yml` which in turn runs the script `ci-ultimate-repo/scripts/ci-test`
 1. `put: pcf-bedazzle` Push the application to our Cloud Foundry instance using the specified manifest.

# Running a Job

## Use Concourse on AWS

From a command line (Terminal or CMD window), run:

```
fly -t aws configure ci-pipeline -c ci.yml --paused=false
```

You will be prompted with `apply configuration? (y/n)` - just answer `y`.  This sets up the new Job as a pipeline and makes it runable (`paused=false`).

__Note:__ Make sure the name of the pipeline specified in `fly configure` is the same as the name in the YAML file.  They don't have to be the same, it's just annoyingly confusing if they aren't.

Once `fly` has setup the pipeline, it should also tell you the URL to use to view the pipeline [http://192.168.100.4:8080/pipelines/cipipeline](http://192.168.100.4:8080/pipelines/ci-pipeline).

Open this URL now and you should see:

![Pipeline Page](https://github.com/pivotal-anz/ci-ultimate/blob/master/screenshots/pipeline-page.png)

Click on the grey-box and you see this:

![Pipeline Jobs Page](https://github.com/pivotal-anz/ci-ultimate/blob/master/screenshots/pipeline-jobs-page.png)

Click the + button on the right to run the job (run the flow).

It will go orange (running) and then green (succeeded).  If it goes red, the job failed.

The run number (#1) will have appeared and so will the steps.  Click on ci-ultimate-repo and unit to make them show their output.  You should see this:

![Jobs Output](https://github.com/pivotal-anz/ci-ultimate/blob/master/screenshots/job-output.png)

## Running Locally

```
fly -t local configure ci-pipeline -c ci.yml --paused=false
```

The URL to use to view the pipeline will be [http://192.168.100.4:8080/pipelines/cipipeline](http://192.168.100.4:8080/pipelines/ci-pipeline).

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

The `say-hello` task is defined using the `config` sub-element instead of a YAML file.  Moreover this flow requires no input or output resources.

As a result, this flow appears in the Web GUI as a single grey box which doesn't look like a flow at all (since it has no input or output).

Whilst this is a nice simple first example, it is not typical and, personally, I found it more confusing than helpful.
