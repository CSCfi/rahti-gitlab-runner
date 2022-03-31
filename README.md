# Intention

Gitlab proposes to deploy the runner using Helm or GitLab operator. It requires additional
permissions. This approach problematically to use in OpenShift Container Platform if you don't have
cluster admin privileges. Moreover, default images are not designed to run using an arbitrarily
assigned user ID. This repo contains dockerfiles and an OpenShift template which allows you to
deploy the GitLab runner in OCP with minimum efforts.

In addition to the OpenShift related adoptions needed to run the Gitlab runner, we require some specific configuation before we can run it in Rahti/Rahti-int. Those additional configuations are included in this repo.

NOTE: This work has been mostly adopted from: https://github.com/RedHatQE/ocp-gitlab-runner.

## Usage

### Quick Rahti-int/Rahti Specifc Usage Instructions
1. Build and push the required images (Optional, otherwise a pre-built image will be used):
 ```sh
 docker build -t some_repo/some_image_name:some_tag -f runner.Dockerfile .
 docker push some_repo/some_image_name:some_tag

 docker build -t some_repo/some_other_image_name:some_tag -f helper.Dockerfile .
 docker push some_repo/some_other_image_name:some_tag
```

2. Deploy the runner by processing the template:
```sh
oc process -f rahti-gitlab-runner-template.yaml \
-p NAME="some_name" \
-p GITLAB_HOST="gitlab.ci.csc.fi" \
-p REGISTRATION_TOKEN="$(echo -n some_token | base64)" \
-p GITLAB_RUNNER_VERSION="v14.9.1" \
-p TLS_CA_CERT="$(openssl s_client -showcerts -connect gitlab.ci.csc.fi:443 < /dev/null 2>/dev/null | openssl x509 -outform PEM | base64)" \
-p TEMPLATE_CONFIG_FILE="$(cat config.template.toml)" | oc create -f -
```

3. Clean up after usage
```sh
oc delete secret,cm,sa,rolebindings,bc,is,deployment -l app=some_name
```

## Contents

### runner.Dockerfile

This image based on `registry.access.redhat.com/ubi8-micro`. It contains `gitlab-runner`
executable that talks to GitLab CI and spawns builder pods via `kubernetes` executor.

### helper.Dockerfile

GitLab's helper image with `gitlab-runner-helper` executable. The image based on
`registry.access.redhat.com/ubi8-minimal`.

### ocp-gitlab-runner-template.yaml

An OpenShift template that creates required objects and deploys the runner with minimum efforts.
Just provide a name, GitLab instance host, runner's registration token and desired number of
concurrent build pods.

#### Parameters

* NAME

    description: Name of DeploymentConfig and value of "app" label.

    required: true

* GITLAB_HOST

    description: Host of a GitLab instance.

    required: true

* GITLAB_RUNNER_VERSION

    description: GitLab runner version, e.g "v14.9.1".

    required: false

* REGISTRATION_TOKEN

    description: Runner's registration token. Base64 encoded string is expected.

    required: true

* CONCURRENT

    description: The maximum number of concurrent CI pods.

    required: true

* RUNNER_TAG_LIST

    description: Tag list.

    required: false

* GITLAB_BUILDER_IMAGE

    description: A default image which will be used in GitLab CI.

    required: false

* TLS_CA_CERT

    description: A certificate that is used to verify TLS peers when connecting to the GitLab
    server. Base64 encoded string is expected.

    required: false

* TEMPLATE_CONFIG_FILE

    description: A patch for config.toml which will be applied during runner registration. Details
    in <https://docs.gitlab.com/runner/register/#runners-configuration-template-file>

    required: false

* RUNNER_IMG

    description: An Image containing the gitlab-runner executable that is used to communicate with GitLab CI.

    required: true

* HELPER_IMG

    description: An image with a special compilation of GitLab Runner binary. It contains only a subset of available commands, as well as Git, Git LFS, SSL certificates store.

    required: true