# Stack Tools

This tooling uses built-in features of Terraform to support both multiple components in the same repository and deploying multiple instances of the same component to the same environment.

Each infrastructure component is a separate Terraform root module. The Terraform root modules are referred to as *stacks*.

You can add the *stacktools* to any project by copying a  file into the project repository. The tooling only requires a UNIX shell, the Make utility and the [jq](https://stedolan.github.io/jq/) command-line tool.

By default, the tools use Docker and a container to provide Terraform. You can override this to either provide your own container image, or use a separate copy of Terraform.

> **The Stacks Specification:** The *stacktools* follow a documented and versioned set of conventions. This means that you can use these modules without the *stacktools*, and develop other tools to work with the stacks. If you fork this repository, update the URL in the file *make/tools/stacktools/stacktools-core.mk* to point to the conventions README in your fork.

## Set Up

### Visual Studio Code

This project includes the configuration for a [development container](https://containers.dev/). This means that Visual Studio Code and Visual Studio can set up a working environment for you on any operating system.

To run the project on Visual Studio Code:

1. Ensure that Docker is running
2. Ensure that the **Dev Containers** extension is installed in Visual Studio Code
3. Open the project as a folder in Visual Studio Code
4. Accept the option to reopen the project in a development container when prompted.
5. Run *make stackrunner-build* to create the Docker container for Terraform

> The development containers configuration provides a Debian container for compatibility with Python code.

## Adding Stack Tools to a Project

The Stack Tools require:

- A UNIX shell
- *GNU Make* 3 or above
- [jq](https://stedolan.github.io/jq/)
- EITHER: Docker OR provide Terraform 1.x separately

The tooling is compatible with the UNIX shell and Make versions that are provided by macOS. You can install *jq* and Docker on macOS with whatever tools that you prefer.

> /!\ **WSL:** The project and tooling are not yet tested on WSL. They should work correctly on any WSL environment that has Make, jq and Docker installed. Alternatively, use a development container.

Create the directory *make/tools/stacktools*:

    mkdir -p make/tools/stacktools

Copy this file into the new directory:

    make/tools/stacktools/stacktools-core.mk

The following *include* and *variables* must be present in the top-level Makefile for your project:

```make
PROJECT_DIR		:= $(shell pwd)
ENVIRONMENT		?= dev

include make/tools/stacktools/*.mk
```

> This does not interfere with any other uses of Make. All of the targets and variables in these *mk* files are namespaced.

Once you have added the Make configuration, run this command to set up the Stack Tools:

    make stacktools-init

All of the files for Terraform are in the directory *terraform1/*. Specify the Terraform remote state settings for each environment. Edit the *backend.json* for each environment in the *terraform1/stacks/environments/* directory.

Refer to the conventions for the expected directory structure.

## Using the Stack Tools

Use Make to run the appropriate commands.

Before you run other commands, use the *stackrunner-build* target to create a Docker container image for Terraform:

    make stackrunner-build

This creates the container image *stacktools-runner*. By default, the tooling runs every Terraform command in a temporary container with this image.

Make targets for Terraform stacks use the prefix *stack-*. For example, *stack-new* creates the directories and files for a new stack:

    make stack-new STACK_NAME=example_app

Specify *ENVIRONMENT* to create a deployment of the stack in the target environment:

    make stack-plan ENVIRONMENT=dev STACK_NAME=example_app
    make stack-apply ENVIRONMENT=dev STACK_NAME=example_app

Specify *STACK_VARIANT* to create an alternate deployment of the same stack in the same environment:

    make stack-plan ENVIRONMENT=dev STACK_NAME=example_app STACK_VARIANT=feature1
    make stack-apply ENVIRONMENT=dev STACK_NAME=example_app STACK_VARIANT=feature1

To delete a variant from the cloud, first *destroy* the instance of the stack, and then use *forget* to delete the Terraform state:

    make stack-destroy ENVIRONMENT=dev STACK_NAME=example_app STACK_VARIANT=feature1
    make stack-forget ENVIRONMENT=dev STACK_NAME=example_app STACK_VARIANT=feature1

> The *stack-forget* target only works on variants, not on the default instances of the stack.

To run Terraform without a container, set *ST_RUN_CONTAINER=false*. For example:

    make stack-fmt STACK_NAME=example_app ST_RUN_CONTAINER=false

To specify a different container image for Terraform, set the *STACK_RUNNER_IMAGE* variable to the name of the container:

    make stack-fmt STACK_NAME=example_app STACK_RUNNER_IMAGE=mytools-runner:1.4.5

### *stack* Targets

| Name           | Description                                                               |
|----------------|---------------------------------------------------------------------------|
| stack-apply    | *terraform apply* for a stack                                             |
| stack-console  | *terraform console* for a stack                                           |
| stack-destroy  | *terraform apply -destroy* for a stack                                    |
| stack-fmt      | *terraform fmt* for a stack                                               |
| stack-forget   | Delete state for a stack variant                                          |
| stack-info     | Show information for a stack                                              |
| stack-init     | *terraform init* for a stack                                              |
| stack-new      | Add source code for a new stack. Copies content from *template* directory |
| stack-plan     | *terraform plan* for a stack                                              |
| stack-rm       | Delete source code for a stack                                            |
| stack-shell    | Open a shell                                                              |
| stack-validate | *terraform validate* for a stack                                          |

### *stacks* Targets

| Name                | Description                           |
|---------------------|---------------------------------------|
| stacks-list         | List the stacks                       |
| stacks-environments | List environments for stacks          |

### *stackrunner* Targets

| Name              | Description           |
|-------------------|-----------------------|
| stackrunner-build | Build container image |
| stackrunner-info  | Show details          |

### *stacktools* Targets

| Name              | Description                     |
|-------------------|---------------------------------|
| stacktools-info   | Show details for stacktools     |
| stacktools-init   | Set up Stack Tools              |
| stacktools-update | Update the Stack Tools          |

## Terraform State

Each stack always has a separate Terraform state file for each environment. The variants feature uses [Terraform workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces). This means that Terraform creates an extra state file for each variant.

## The stacktools-runner Container Image

The *stacktools-runner* images are built to provide a complete environment for running Terraform and the *stacktools*. This means that they include Make and jq as well as Terraform.

The defaults for the *stacktools-runner* container image use Alpine Linux to produce small images.

The default image tag uses the version *developer*. This enables you to rebuild the container image on your machine at any time. If you publish an image to a shared repository, set the version to a fixed number.

Use the *ST_RUNNER_VERSION* variable to build a container image with a fixed version:

    make stackrunner-build ST_RUNNER_VERSION=1.4.5

## Automation and CI/CD

You can use a *stacktools-runner* container to provide an environment to deploy Terraform with Continuous Integration. In most cases, use your own container image instead.

You can run *stacktools* in any container that includes Make, *jq*, a UNIX shell and either Docker-in-Docker (DIND) or Terraform.

If you run all of the deployment process for your project in a container, include a copy of Terraform in the container and use *ST_RUN_CONTAINER=false* to prevent the tooling from creating a new temporary container for each command.

    make stack-apply ENVIRONMENT=dev STACK_NAME=example_app ST_RUN_CONTAINER=false

### Cross-Architecture Support

By default, *stackrunner-build* builds the *stacktools-runner* container image for the same CPU architecture as the machine that the command is run on. To build for another CPU architecture, override *ST_RUNNER_TARGET_CPU_ARCH*. For example, specify *arm64* for ARM:

    make stackrunner-build ST_RUNNER_TARGET_CPU_ARCH=arm64

---

## TODOs

- Add guards in Make for undefined PROJECT_DIR, STACK_NAME or ENVIRONMENT variables.
- More friendly error message for *stack-forget* calls that try to delete default Terraform workspaces.
- Define standard path structure for Parameter Store.
- Provide guidance on handling of secrets.
- Provide guidance on executing commands on multiple stacks.
- Improve guidance on cross-stack references.
- Guidance for CI/CD pipelines
