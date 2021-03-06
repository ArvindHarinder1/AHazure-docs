---
title: Tutorial - Automate container image builds with Azure Container Registry Build
description: In this tutorial, you learn how to configure a build task to automatically trigger container image builds in the cloud when you commit source code to a Git repository.
services: container-registry
author: mmacy
manager: jeconnoc

ms.service: container-registry
ms.topic: tutorial
ms.date: 04/23/2018
ms.author: marsma
ms.custom: mvc
# Customer intent: As a developer or devops engineer, I want to trigger
# container image builds automatically when I commit code to a Git repo.
---

# Tutorial: Automate container image builds with Azure Container Registry Build

In addition to [Quick Build](container-registry-tutorial-quick-build.md), ACR Build supports automated Docker container image builds with the *build task*. In this tutorial, you use the Azure CLI to create a build task that automatically triggers image builds in the cloud when you commit source code to a Git repository.

In this tutorial, part two in the series:

> [!div class="checklist"]
> * Create a build task
> * Test the build task
> * View build status
> * Trigger the build task with a code commit

This tutorial assumes you've already completed the steps in the [previous tutorial](container-registry-tutorial-quick-build.md). If you haven't already done so, complete the steps in the [Prerequisites](container-registry-tutorial-quick-build.md#prerequisites) section of the previous tutorial before proceeding.

> [!IMPORTANT]
> ACR Build is in currently in preview, and is supported only by Azure container registries in the **EastUS** region. Previews are made available to you on the condition that you agree to the [supplemental terms of use][terms-of-use]. Some aspects of this feature may change prior to general availability (GA).

[!INCLUDE [cloud-shell-try-it.md](../../includes/cloud-shell-try-it.md)]

If you'd like to use the Azure CLI locally, you must have Azure CLI version 2.0.31 or later installed. Run `az --version` to find the version. If you need to install or upgrade the CLI, see [Install Azure CLI 2.0][azure-cli].

## Prerequisites

### Get ACR Build and sample code

This tutorial assumes you've already completed the steps in the [previous tutorial](container-registry-tutorial-quick-build.md) to install ACR Build, and fork and clone the sample repository. If you haven't already done so, complete the steps in the [Prerequisites](container-registry-tutorial-quick-build.md#prerequisites) section of the previous tutorial before proceeding.

### Container registry

You must have an Azure container registry in your Azure subscription to complete this tutorial. If you need a registry, see the [previous tutorial](container-registry-tutorial-quick-build.md), or [Quickstart: Create a container registry using the Azure CLI](container-registry-get-started-azure-cli.md).

## Build task

A build task defines the properties of an automated build, including the location of the container image source code and the event that triggers the build. When an event defined in the build task occurs, such as a commit to a Git repository, ACR Build initiates a container image build in the cloud. By default, it then pushes a successfully built image to the Azure container registry specified in the task.

ACR Build currently supports the following build task triggers:

* Commit to a Git repository
* Base image update

## Create a build task

In this section, you first create a GitHub personal access token (PAT) for use with ACR Build. Then, you create a build task that triggers a build when code is committed to your fork of the repository.

### Create a GitHub personal access token

To trigger a build on a commit to a Git repository, ACR Build needs a personal access token (PAT) to access the repository. Follow these steps to generate a PAT in GitHub:

1. Navigate to the PAT creation page on GitHub at https://github.com/settings/tokens/new
1. Enter a short **description** for the token, for example, "ACR Build Task Demo"
1. Under **repo**, enable **repo:status** and **public_repo**

   ![Screenshot of the Personal Access Token generation page in GitHub][build-task-01-new-token]

1. Select the **Generate token** button (you may be asked to confirm your password)
1. Copy and save the generated token in a **secure location** (you use this token when you define a build task in the following section)

   ![Screenshot of the generated Personal Access Token in GitHub][build-task-02-generated-token]

### Create the build task

Now that you've completed the steps required to enable ACR Build to read commit status and create webhooks in a repository, you can create a build task that triggers a container image build on commits to the repo.

First, populate these shell environment variables with values appropriate for your environment. This isn't strictly required, but makes executing the multiline Azure CLI commands in this tutorial a bit easier. If you don't populate these, you must manually replace each value wherever they appear in the example commands.

```azurecli-interactive
ACR_NAME=mycontainerregistry # The name of your Azure container registry
GIT_USER=gituser             # Your GitHub user account name
GIT_PAT=personalaccesstoken  # The PAT you generated in the previous section
```

Now, create the build task by executing following [az acr build-task create][az-acr-build-task-create] command.

This build task specifies that any time code is committed to the *master* branch in the repository specified by `--context`, ACR Build will build the container image from the code in that branch. The `--image` argument specifies a parameterized value of `{{.Build.ID}}` for the version portion of the image's tag, ensuring the built image correlates to a specific build, and is tagged uniquely.

```azurecli-interactive
az acr build-task create \
    --registry $ACR_NAME \
    --name buildhelloworld \
    --image helloworld:{{.Build.ID}} \
    --context https://github.com/$GIT_USER/acr-build-helloworld-node \
    --branch master \
    --git-access-token $GIT_PAT
```

Output from a successful [az acr build-task create][az-acr-build-task-create] command is similar to the following:

```console
$ az acr build-task create \
>     --registry $ACR_NAME \
>     --name buildhelloworld \
>     --image helloworld:{{.Build.ID}} \
>     --context https://github.com/$GIT_USER/acr-build-helloworld-node \
>     --branch master \
>     --git-access-token $GIT_PAT
{
  "additionalProperties": {},
  "alias": "buildhelloworld",
  "creationDate": "2018-04-18T23:14:45.905395+00:00",
  "id": "/subscriptions/<subscriptionID>/resourceGroups/myResourceGroup/providers/Microsoft.ContainerRegistry/registries/mycontainerregistry/buildTasks/buildhelloworld",
  "location": "eastus",
  "name": "buildhelloworld",
  "platform": {
    "additionalProperties": {},
    "cpu": 1,
    "osType": "Linux"
  },
  "provisioningState": "Succeeded",
  "resourceGroup": "myResourceGroup",
  "sourceRepository": {
    "additionalProperties": {},
    "isCommitTriggerEnabled": true,
    "repositoryUrl": "https://github.com/gituser/acr-build-helloworld-node",
    "sourceControlAuthProperties": null,
    "sourceControlType": "Github"
  },
  "status": "enabled",
  "tags": null,
  "timeout": null,
  "type": "Microsoft.ContainerRegistry/registries/buildTasks"
}

```

## Test the build task

You now have a build task that defines your build. To test the build definition, trigger a build manually by executing the [az acr build-task run][az-acr-build-task-run] command:

```azurecli-interactive
az acr build-task run --registry $ACR_NAME --name buildhelloworld
```

By default, the `az acr build-task run` command streams the log output to your console when you execute the command. Here, the output shows that build **eastus2** has been queued and built.

```console
$ az acr build-task run --registry mycontainerregistry --name buildhelloworld
Queued a build with build-id: eastus2.
Starting to stream the logs...
Cloning into '/root/acr-builder/src'...
time="2018-04-19T00:06:20Z" level=info msg="Running command git checkout master"
Already on 'master'
Your branch is up to date with 'origin/master'.
ffef1347389a008c9a8bfdf8c6a0ed78b0479894
time="2018-04-19T00:06:20Z" level=info msg="Running command git rev-parse --verify HEAD"
time="2018-04-19T00:06:20Z" level=info msg="Running command docker build --pull -f Dockerfile -t acr22818.azurecr.io/helloworld:eastus2 ."
Sending build context to Docker daemon  182.8kB
Step 1/5 : FROM node:9-alpine
9: Pulling from library/node
Digest: sha256:bd7b9aaf77ab2ce1e83e7e79fc0969229214f9126ced222c64eab49dc0bdae90
Status: Image is up to date for node:9-alpine
 ---> aa3e171e4e95
Step 2/5 : COPY . /src
 ---> e1c04dc2993b

[...]

6e5e20cbf4a7: Layer already exists
b69680cb4898: Pushed
b54af9b858b7: Pushed
eastus2: digest: sha256:9a7b73d06077ced2a02f7462f53e31a3e51e95ea5544fbcdb01e2fef094da1b6 size: 2423
time="2018-04-19T00:06:51Z" level=info msg="Running command docker inspect --format \"{{json .RepoDigests}}\" acr22818.azurecr.io/helloworld:eastus2"
"["acr22818.azurecr.io/helloworld@sha256:9a7b73d06077ced2a02f7462f53e31a3e51e95ea5544fbcdb01e2fef094da1b6"]"
time="2018-04-19T00:06:51Z" level=info msg="Running command docker inspect --format \"{{json .RepoDigests}}\" node:9-alpine"
"["node@sha256:bd7b9aaf77ab2ce1e83e7e79fc0969229214f9126ced222c64eab49dc0bdae90"]"
ACR Builder discovered the following dependencies:
[{"image":{"registry":"acr22818.azurecr.io","repository":"helloworld","tag":"eastus2","digest":"sha256:9a7b73d06077ced2a02f7462f53e31a3e51e95ea5544fbcdb01e2fef094da1b6"},"runtime-dependency":{"registry":"registry.hub.docker.com","repository":"node","tag":"9","digest":"sha256:bd7b9aaf77ab2ce1e83e7e79fc0969229214f9126ced222c64eab49dc0bdae90"},"buildtime-dependency":null,"git":{"git-head-revision":"ffef1347389a008c9a8bfdf8c6a0ed78b0479894"}}]
Build complete
Build ID: eastus2 was successful after 39.789138274s
```

## View build status

You may occasionally find it useful to view the status of an ongoing build you've not triggered manually. For example, while troubleshooting builds triggered by source code commits. In this section, you trigger a manual build, but suppress the default behavior of streaming the build log to your console. Then, you use the `az acr build-task logs` command to monitor the ongoing build.

First, trigger a build manually as you've done previously, but specify the `--no-logs` argument to suppress logging to your console:

```azurecli-interactive
az acr build-task run --registry $ACR_NAME --name buildhelloworld --no-logs
```

Next, use the `az build-task logs` command to view the log of the currently running build:

```azurecli-interactive
az acr build-task logs --registry $ACR_NAME
```

The log for the currently running build is streamed to your console, and should appear similar to the following output (shown here truncated):

```console
$ az acr build-task logs --registry $ACR_NAME
Showing logs for the last updated build...
Build-id: eastus3

[...]

Build complete
Build ID: eastus3 was successful after 30.076988169s
```

## Trigger a build with a commit

Now that you've tested the build task by manually running it, trigger it automatically with a source code change.

First, ensure you're in the directory containing your local clone of the [repository][sample-repo]:

```azurecli-interactive
cd acr-build-helloworld-node
```

Next, execute the following commands to create, commit, and push a new file to your fork of the repo on GitHub:

```azurecli-interactive
echo "Hello World!" > hello.txt
git add hello.txt
git commit -m "Testing ACR Build"
git push origin master
```

You may be asked to provide your GitHub credentials when you execute the `git push` command. Provide your GitHub username, and enter the personal access token (PAT) that you created earlier for the password.

```console
$ git push origin master
Username for 'https://github.com': <github-username>
Password for 'https://githubuser@github.com': <personal-access-token>
```

Once you've pushed a commit to your repository, the webhook created by ACR Build fires and kicks off a build in Azure Container Registry. Display the build logs for the currently running build to verify and monitor the build progress:

```azurecli-interactive
az acr build-task logs --registry $ACR_NAME
```

Output is similar to the following, showing the currently executing (or last-executed) build:

```console
$ az acr build-task logs --registry $ACR_NAME
Showing logs for the last updated build...
Build-id: eastus4

[...]

Build complete
Build ID: eastus4 was successful after 28.9587031s
```

## List builds

To see a list of the builds that ACR Build has completed for your registry, run the [az acr build-task list-builds][az-acr-build-task-list-builds] command:

```azurecli-interactive
az acr build-task list-builds --registry $ACR_NAME --output table
```

Output from the command should appear similar to the following. The builds that ACR Build has executed are displayed, and "Git Commit" appears in the TRIGGER column for the most recent build:

```console
$ az acr build-task list-builds --registry $ACR_NAME --output table
BUILD ID    TASK             PLATFORM    STATUS     TRIGGER     STARTED               DURATION
----------  ---------------  ----------  ---------  ----------  --------------------  ----------
eastus4     buildhelloworld  Linux       Succeeded  Git Commit  2018-04-20T22:50:27Z  00:00:35
eastus3     buildhelloworld  Linux       Succeeded  Manual      2018-04-20T22:47:19Z  00:00:30
eastus2     buildhelloworld  Linux       Succeeded  Manual      2018-04-20T22:46:14Z  00:00:55
eastus1                                  Succeeded  Manual      2018-04-20T22:38:22Z  00:00:55
```

## Next steps

In this tutorial, you learned how to use a build task to automatically trigger container image builds in Azure when you commit source code to a Git repository. Move on to the next tutorial to learn how to create build tasks that trigger builds when a container image's base image is updated.

> [!div class="nextstepaction"]
> [Automate builds on base image update](container-registry-tutorial-base-image-update.md)

<!-- LINKS - External -->
[terms-of-use]: https://azure.microsoft.com/support/legal/preview-supplemental-terms/
[sample-repo]: https://github.com/Azure-Samples/acr-build-helloworld-node

<!-- LINKS - Internal -->
[azure-cli]: /cli/azure/install-azure-cli
[az-acr-build-task]: /cli/azure/acr#az-acr-build-task
[az-acr-build-task-create]: /cli/azure/acr#az-acr-build-task-create
[az-acr-build-task-run]: /cli/azure/acr#az-acr-build-task-run
[az-acr-build-task-list-builds]: /cli/azure/acr#az-acr-build-task-list-build

<!-- IMAGES -->
[build-task-01-new-token]: ./media/container-registry-tutorial-build-tasks/build-task-01-new-token.png
[build-task-02-generated-token]: ./media/container-registry-tutorial-build-tasks/build-task-02-generated-token.png
