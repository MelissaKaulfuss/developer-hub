---
title: Build and Push to GCR
description: Use a CI pipeline to build and push an image to GCR.
sidebar_position: 50
helpdocs_topic_id: gstwrwjwgu
helpdocs_category_id: mi8eo3qwxm
helpdocs_is_private: false
helpdocs_is_published: true
---

This topic explains how to configure the **Build and Push to GCR** step in a Harness CI pipeline. This step is used to build and push to [Google Container Registry (GCR)](https://cloud.google.com/container-registry).

You need:

* Access to GCR and a GCR repo.
* A [Harness CI pipeline](../prep-ci-pipeline-components.md) with a [Build stage](../set-up-build-infrastructure/ci-stage-settings.md).
* A [GCP connector](#gcp-connector).

## Kubernetes cluster build infrastructures require root access

With Kubernetes cluster build infrastructures, **Build and Push** steps use [kaniko](https://github.com/GoogleContainerTools/kaniko/blob/main/README.md). Other build infrastructures use [drone-docker](https://github.com/drone-plugins/drone-docker/blob/master/README.md). Kaniko requires root access to build the Docker image. It doesn't support non-root users.

If your build runs as non-root (`runAsNonRoot: true`), and you want to run the **Build and Push** step as root, you can set **Run as User** to `0` on the **Build and Push** step to use the root user for that individual step only.

If your security policy doesn't allow running as root, go to [Build and push with non-root users](./build-and-push-nonroot.md).

## Add a Build and Push to GCR step

In your pipeline's **Build** stage, add a **Build and Push to GCR** step and configure the [settings](#build-and-push-to-gcr-step-settings) accordingly.

Here is a YAML example of a minimum **Build and Push to GCR** step.

```yaml
              - step:
                  type: BuildAndPushGCR
                  name: BuildAndPushGCR_1
                  identifier: BuildAndPushGCR_1
                  spec:
                    connectorRef: YOUR_GCP_CONNECTOR_ID
                    host: us.gcr.io
                    projectID: my_project
                    imageName: my_image
                    tags:
                      - <+pipeline.sequenceId>
```

When you run a pipeline, you can observe the step logs on the [build details page](../viewing-builds.md). If the **Build and Push to GCR** step succeeds, you can find the uploaded image on GCR.

:::tip

You can also:

* [Build images without pushing](./build-without-push.md)
* [Build multi-architecture images](./build-multi-arch.md)

:::

## Build and Push to GCR step settings

The **Build and Push to GCR** step has the following settings. Depending on the stage's build infrastructure, some settings might be unavailable or optional. Settings specific to containers, such as **Set Container Resources**, are not applicable when using a VM or Harness Cloud build infrastructure.

### Name

Enter a name summarizing the step's purpose. Harness automatically assigns an **Id** ([Entity Identifier Reference](../../../platform/references/entity-identifier-reference.md)) based on the **Name**. You can change the **Id**.

### GCP Connector

The Harness GCP connector to use to connect to GCR. The GCP account associated with the GCP connector needs specific roles. For more information, go to [Google Cloud Platform (GCP) connector settings reference](/docs/platform/connectors/cloud-providers/ref-cloud-providers/gcs-connector-settings-reference).

This step supports GCP connectors that use access key authentication. It does not support GCP connectors that inherit delegate credentials.

### Host

The Google Container Registry hostname. For example, `us.gcr.io` hosts images in data centers in the United States in a separate storage bucket from images hosted by `gcr.io`. For a list of Container Registries, go to the Google documentation on [Pushing and pulling images](https://cloud.google.com/container-registry/docs/pushing-and-pulling).

:::info

The target GCR registry must meet the GCR requirements for pushing images. For more information, go to the Google documentation on [Pushing and pulling images](https://cloud.google.com/container-registry/docs/pushing-and-pulling).

:::

### Project ID

The [GCP resource manager project ID](https://cloud.google.com/resource-manager/docs/creating-managing-projects#identifying_projects).

### Image Name

The name of the image you want to build and push to the target container registry.

### Tags

Add [Docker build tags](https://docs.docker.com/engine/reference/commandline/build/#tag). This is equivalent to the `-t` flag.

Add each tag separately.

:::tip

When you push an image to a repo, you tag the image so you can identify it later. For example, in one pipeline stage, you push the image, and, in a later stage, you use the image name and tag to pull it and run integration tests on it.

Harness expressions are a useful way to define tags. For example, you can use the expression `<+pipeline.sequenceId>` as a tag. This expression represents the incremental build identifier, such as `9`. By using a variable expression, rather than a fixed value, you don't have to use the same image name every time.

For example, if you use `<+pipeline.sequenceId>` as a tag, after the pipeline runs, you can see the `Build Id` in the output.

![](./static/build-and-upload-an-artifact-15.png)

And you can see where the Build ID is used to tag your image:

![](./static/build-and-upload-an-artifact-12.png)

Later in the pipeline, you can use the same expression to pull the tagged image, such as `myrepo/myimage:<+pipeline.sequenceId>`.

:::

### Optimize

With Kubernetes cluster build infrastructures, select this option to enable `--snapshotMode=redo`. This setting causes file metadata to be considered when creating snapshots, and it can reduce the time it takes to create snapshots. For more information, go to the kaniko documentation for the [snapshotMode flag](https://github.com/GoogleContainerTools/kaniko/blob/main/README.md#flag---snapshotmode).

For information about setting other kaniko runtime flags, go to [Set kaniko runtime flags](#set-kaniko-runtime-flags).

### Dockerfile

The name of the Dockerfile. If you don't provide a name, Harness assumes that the Dockerfile is in the root folder of the codebase.

### Context

Enter a path to a directory containing files that make up the [build's context](https://docs.docker.com/engine/reference/commandline/build/#description). When the pipeline runs, the build process can refer to any files found in the context. For example, a Dockerfile can use a `COPY` instruction to reference a file in the context.

### Labels

Specify [Docker object labels](https://docs.docker.com/config/labels-custom-metadata/) to add metadata to the Docker image.

### Build Arguments

The [Docker build-time variables](https://docs.docker.com/engine/reference/commandline/build/#build-arg). This is equivalent to the `--build-arg` flag.

![](./static/build-and-push-to-gcr-step-settings-23.png)

### Target

The [Docker target build stage](https://docs.docker.com/engine/reference/commandline/build/#target), equivalent to the `--target` flag, such as `build-env`.

### Remote Cache Image

Enter the name of the remote cache image, such as `gcr.io/project-id/<image>`.

The remote cache repository must be in the same account and organization as the build image. For caching to work, the specified image name must exist.

Harness enables remote Docker layer caching where each Docker layer is uploaded as an image to a Docker repo you identify. If the same layer is used in later builds, Harness downloads the layer from the Docker repo. You can also specify the same Docker repo for multiple **Build and Push** steps, enabling these steps to share the same remote cache. This can dramatically improve build time by sharing layers across pipelines, stages, and steps.

### Run as User

With Kubernetes cluster build infrastructures, you can specify the user ID to use to run all processes in the pod if running in containers. For more information, go to [Set the security context for a pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod).

This step requires root access. You can use the **Run as User** setting if your build runs as non-root (`runAsNonRoot: true`), and you can run the **Build and Push** step as root. To do this, set **Run as User** to `0` to use the root user for this individual step only.

If your security policy doesn't allow running as root, go to [Build and push with non-root users](./build-and-push-nonroot.md).

### Set container resources

Set maximum resource limits for the resources used by the container at runtime:

* **Limit Memory:** The maximum memory that the container can use. You can express memory as a plain integer or as a fixed-point number using the suffixes `G` or `M`. You can also use the power-of-two equivalents `Gi` and `Mi`. The default is `500Mi`.
* **Limit CPU:** The maximum number of cores that the container can use. CPU limits are measured in CPU units. Fractional requests are allowed; for example, you can specify one hundred millicpu as `0.1` or `100m`. The default is `400m`. For more information, go to [Resource units in Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes).

### Timeout

Set the timeout limit for the step. Once the timeout limit is reached, the step fails and pipeline execution continues. To set skip conditions or failure handling for steps, go to:

* [Step Skip Condition settings](../../../platform/pipelines/w_pipeline-steps-reference/step-skip-condition-settings.md)
* [Step Failure Strategy settings](../../../platform/pipelines/w_pipeline-steps-reference/step-failure-strategy-settings.md)

### Conditions, looping, and failure strategies

You can find the following settings on the **Advanced** tab in the step settings pane:

* [Conditional Execution](../../../platform/pipelines/w_pipeline-steps-reference/step-skip-condition-settings.md): Set conditions to determine when/if the step should run.
* [Failure Strategy](../../../platform/pipelines/w_pipeline-steps-reference/step-failure-strategy-settings.md): Control what happens to your pipeline when a step fails.
* [Looping Strategies Overview -- Matrix, Repeat, and Parallelism](/docs/platform/pipelines/looping-strategies-matrix-repeat-and-parallelism): Define a looping strategy for an individual step.

### Set kaniko runtime flags

With Kubernetes cluster build infrastructures, **Build and Push** steps use [kaniko](https://github.com/GoogleContainerTools/kaniko/blob/main/README.md). Other build infrastructures use [drone-docker](https://github.com/drone-plugins/drone-docker/blob/master/README.md).

You can set kaniko runtime flags by adding [stage variables](/docs/platform/pipelines/add-a-stage/#option-stage-variables) formatted as `PLUGIN_FLAG_NAME`. For example, to set `--skip-tls-verify`, you would add a stage variable named `PLUGIN_SKIP_TLS_VERIFY` and set the variable value to `true`.

```yaml
        variables:
          - name: PLUGIN_SKIP_TLS_VERIFY
            type: String
            description: ""
            required: false
            value: "true"
```
