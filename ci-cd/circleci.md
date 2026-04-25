# CircleCI

## What is CircleCI

CircleCI is a **CI/CD (Continuous Integration/Continuous Deployment)** <mark style="color:red;">platform that automates the process of building, testing</mark>, and deploying software applications. It helps developers integrate code frequently, ensuring that new code works seamlessly with existing code, while also enabling fast and reliable application delivery, while its primary purpose is to automate **building** and **testing** applications, it also supports **deployment** as part of its pipelines.

***

## How CircleCI works

1. **Integration with Your Code Repository**
   * CircleCI connects to your GitHub, Gitlab, or Bitbucket
   * It triggers Pipeline when new event happen to your Repo (Push, Pull request, and so on...)
2. **Configuration File**
   * define `.circleci` folder at the root of your project folder, And then create file named `config.yml` &#x20;
3. **Pipeline Execution**
   * represents the entirety of your configuration
4. **Environment Options**
   * CircleCI runs jobs inside containers, and VMs depending on your configurations
   * You can define the environment needed for your application (e.g., specific programming language, database, or tools).
5. **Parallelism and Caching**
   * CircleCI supports running multiple jobs in parallel to speed up the pipeline.
   * It also caches dependencies to avoid downloading or rebuilding them repeatedly (make pipeline faster every time).
6. **Reusable Configurations**
   * Using **Orbs** (predefined configurations), you can easily reuse common CI/CD tasks across multiple projects.

***

## CircleCI configuration

* [**Pipeline**](circleci.md#pipeline): Represents the <mark style="color:red;">entirety of your configuration</mark>.
* [**Workflows**](circleci.md#workflows): Responsible for <mark style="color:red;">orchestrating</mark> multiple _jobs_.
* [**Jobs**](circleci.md#jobs): Responsible for running a <mark style="color:red;">series of</mark> <mark style="color:red;"></mark>_<mark style="color:red;">steps</mark>_ that perform commands.
* [**Steps**](circleci.md#steps): <mark style="color:red;">Run commands and shell scripts</mark> to do the work required for your project.

***

### Pipeline

Pipeline is the whole process of workflows, We could say it is the entire YAML file

***

### Workflows

Workflows responsible for <mark style="color:red;">orchestrate</mark> jobs, A workflow lists jobs with correct order to run.

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption><p>Workflow</p></figcaption></figure>

Workflow has several types of order

*   #### Sequential job execution <a href="#sequential-job-execution" id="sequential-job-execution"></a>

    In this type, your just <mark style="color:red;">lists correct order of jobs</mark>

```yaml
workflows:
  build-test-and-deploy:
    jobs:
      - build
      - test1:
          requires:
            - build
      - test2:
          requires:
            - test1
      - deploy:
          requires:
            - test2
```

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption><p>Sequential job diagram</p></figcaption></figure>

*   #### Fan-out/fan-in workflow <a href="#fan-outfan-in-workflow" id="fan-outfan-in-workflow"></a>

    in this type you run multiple jobs in <mark style="color:red;">parallel</mark> if specific jobs successes

```yaml
workflows:
  build_accept_deploy:
    jobs:
      - build
      - acceptance_test_1:
          requires:
            - build
      - acceptance_test_2:
          requires:
            - build
      - acceptance_test_3:
          requires:
            - build
      - acceptance_test_4:
          requires:
            - build
      - deploy:
          requires:
            - acceptance_test_1
            - acceptance_test_2
            - acceptance_test_3
            - acceptance_test_4
```

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption><p>Fan out/In diagram</p></figcaption></figure>

*   Hold a workflow for a manual approve

    an approval job to configure a workflow to <mark style="color:red;">wait for a manual approval</mark>

```yaml
workflows:
  version: 2
  deploy-workflow:
    jobs:
      - build
      - test
      - hold:  # Approval job
          type: approval
          requires:
            - test
      - deploy:
          requires:
            - hold
```

<mark style="color:red;">note: you could name Approval job to (hold, pause, and wait)</mark>

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption><p>Approval Job</p></figcaption></figure>

***

### Jobs

Jobs are the building blocks of your configuration. Jobs are <mark style="color:red;">collections of steps</mark>, which runs executable commands/scripts, in Each job you must declare an executer that is either a docker or machine (Linux, windows, macOS)

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption><p>Job Format</p></figcaption></figure>

```yaml
  build2:
    machine: # Specifies a machine image that uses
      # an Ubuntu version 22.04 image
      image: ubuntu-2204:2024.01.2
    steps:
      - checkout
```

***

### Steps

A Collection of <mark style="color:red;">executable commands</mark> required to complete your job

```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: cimg/base:2024.02
    steps:
      - checkout # Special step to checkout your source code
      - run: # Run step to execute commands, see 
      # circleci.com/docs/configuration-reference/#run
          name: Running tests
          command: make test
          # executable command run in
          # non-login shell with /bin/bash -eo pipefail option
          # by default.
```

like in this Code we have `Checkout` it's first step that get your src code then using `run` to execute commands&#x20;

***

### Parallelism <a href="#parallelism" id="parallelism"></a>

If you have Specific job with multiple task take time to execute you can use parallelism so you can <mark style="color:red;">use more than one executer to run it in different machine or image in parallel</mark>

```yaml
jobs:
  build:
    docker:
      - image: cimg/go:1.18.1
    parallelism: 4
    resource_class: large
    steps:
      - run: go list ./... | circleci tests run --command "xargs gotestsum --junitfile junit.xml --format testname --" --split-by=timings --timings-type=name
```

A `resource class` is a configuration option that allows you to <mark style="color:red;">control available compute resources</mark> (CPU and RAM) for your jobs. When you specify an execution environment for a job, a default resource class value for the environment will be set, _unless_ you define the resource class in your configuration. It is best practice to define the resource class, as opposed to relying on a default.

## [Reference](https://circleci.com/docs/concepts/#projects)



