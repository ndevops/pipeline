# Tekton Pipelines Official Release Cheat Sheet

These steps provide a no-frills guide to performing an official release
of Tekton Pipelines. To follow these steps you'll need a checkout of
the pipelines repo, a terminal window and a text editor.

1. [Setup a context to connect to the dogfooding cluster](#setup-dogfooding-context) if you haven't already.

1. `cd` to root of Pipelines git checkout.

1. Select the commit you would like to build the release from, most likely the
   most recent commit at https://github.com/tektoncd/pipeline/commits/master
   and note the commit's hash.

1. Create a `.yaml` file containing PipelineResource for new version. e.g.

    ```yaml
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineResource
    metadata:
      name: # Example: tekton-pipelines-v0-11-2
    spec:
      type: git
      params:
      - name: url
        value: https://github.com/tektoncd/pipeline
      - name: revision
        value: # The commmit you selected in the last step, e.g. 33e0847e67fc9804689e50371746c3cdad4b0a9d
    ```

1. Apply file you just made to the dogfooding cluster: `kubectl --context dogfooding apply -f your-pipeline-resource-file.yaml`

1. Create environment variables for bash scripts in later steps.

    ```bash
    TEKTON_VERSION=# Example: v0.11.0
    TEKTON_RELEASE_GIT_RESOURCE=# Name of the resource you created, e.g.: tekton-pipelines-v0-11-0
    TEKTON_IMAGE_REGISTRY=gcr.io/tekton-releases # only change if you want to publish to a different registry
    ```

1. Confirm commit SHA matches what you want to release.

    ```bash
    kubectl --context dogfooding get pipelineresource "$TEKTON_RELEASE_GIT_RESOURCE" -o=jsonpath="{'Target Revision: '}{.spec.params[?(@.name == 'revision')].value}{'\n'}"
    ```

1. Execute the release pipeline.

    **If you are backporting include this flag: `--param=releaseAsLatest="false"`**

    ```bash
    tkn --context dogfooding pipeline start \
      --param=versionTag="${TEKTON_VERSION}" \
      --param=imageRegistry="${TEKTON_IMAGE_REGISTRY}" \
      --serviceaccount=release-right-meow \
      --resource=source-repo="${TEKTON_RELEASE_GIT_RESOURCE}" \
      --resource=bucket=pipeline-tekton-bucket \
      --resource=builtBaseImage=base-image \
      --resource=builtEntrypointImage=entrypoint-image \
      --resource=builtNopImage=nop-image \
      --resource=builtKubeconfigWriterImage=kubeconfigwriter-image \
      --resource=builtCredsInitImage=creds-init-image \
      --resource=builtGitInitImage=git-init-image \
      --resource=builtControllerImage=controller-image \
      --resource=builtWebhookImage=webhook-image \
      --resource=builtDigestExporterImage=digest-exporter-image \
      --resource=builtPullRequestInitImage=pull-request-init-image \
      --resource=builtGcsFetcherImage=gcs-fetcher-image \
      --resource=notification=post-release-trigger \
    pipeline-release
    ```

1. Watch logs of pipeline-release.

1. The YAMLs are now released! Anyone installing Tekton Pipelines will now get the new version. Time to create a new GitHub release announcement:

    1. Choose a name for the new release! The usual pattern is "< cat breed > < famous robot >" e.g. "Ragdoll Norby".
       (Check the previous releases to avoid repetition: https://github.com/tektoncd/pipeline/releases.)

    1. Create additional environment variables

    ```bash
    TEKTON_OLD_VERSION=# Example: v0.11.1
    TEKTON_RELEASE_NAME=# The release name you just chose, e.g.: "Ragdoll Norby"
    TEKTON_PACKAGE=tektoncd/pipeline
    ```

    1. Execute the Draft Release task.

    ```bash
    tkn --context dogfooding task start \
      -i source="${TEKTON_RELEASE_GIT_RESOURCE}" \
      -i release-bucket=pipeline-tekton-bucket \
      -p package="${TEKTON_PACKAGE}" \
      -p release-tag="${TEKTON_VERSION}" \
      -p previous-release-tag="${TEKTON_OLD_VERSION}" \
      -p release-name="${TEKTON_RELEASE_NAME}" \
      create-draft-release
    ```

    1. Watch logs of create-draft-release

    1. On successful completion, a URL will be logged. Visit that URL and look through the release notes.
      1. Manually add upgrade and deprecation notices based on the generated release notes
      1. Double-check that the list of commits here matches your expectations
         for the release. You might need to remove incorrect commits or copy/paste commits
         from the release branch. Refer to previous releases to confirm the expected format.

    1. Un-check the "This is a pre-release" checkbox since you're making a legit for-reals release!

    1. Publish the GitHub release once all notes are correct and in order.

1. Edit `README.md` on `master` branch, add entry to docs table with latest release links.

1. Push & make PR for updated `README.md`

1. Test release that you just made against your own cluster (note `--context my-dev-cluster`):

    ```bash
    # Test latest
    kubectl --context my-dev-cluster apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
    ```

    ```bash
    # Test backport
    kubectl --context my-dev-cluster apply --filename https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.11.2/release.yaml
    ```

1. Announce the release in Slack channels #general and #pipelines.

1. Update [the catalog repo](https://github.com/tektoncd/catalog) test infrastructure
to use the new release by updating the `RELEASE_YAML` link in [e2e-tests.sh](https://github.com/tektoncd/catalog/blob/master/test/e2e-tests.sh).

Congratulations, you're done!

## Setup dogfooding context

1. Configure `kubectl` to connect to
   [the dogfooding cluster](https://github.com/tektoncd/plumbing/blob/master/docs/dogfooding.md):

    ```bash
    gcloud container clusters get-credentials dogfooding --zone us-central1-a --project tekton-releases
    ```

1. Give [the context](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
   a short memorable name such as `dogfooding`:

   ```bash
   kubectl config rename-context gke_tekton-releases_us-central1-a_dogfooding dogfoodin
   ```

1. **Important: Switch `kubectl` back to your own cluster by default.**

    ```bash
    kubectl config use-context my-dev-cluster
    ```
