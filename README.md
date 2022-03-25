# Intention

Gitlab proposes to deploy the runner using Helm or GitLab operator. It requires additional
permissions. This approach problematically to use in OpenShift Container Platform if you don't have
cluster admin privileges. Moreover, default images are not designed to run using an arbitrarily
assigned user ID. This repo contains dockerfiles and an OpenShift template which allows you to
deploy the GitLab runner in OCP with minimum efforts.

## Usage

Add and instantiate the template. You should replace `GITLAB_RUNNER_VERSION` by required version of
`gitlab-runner`, e.g. `v14.9.1`, `main`, etc. For any changes in version tagging conventions check the repositoy at <https://gitlab.com/gitlab-org/gitlab-runner>.

```sh
oc process -f https://raw.githubusercontent.com/RedHatQE/ocp-gitlab-runner/GITLAB_RUNNER_VERSION/ocp-gitlab-runner-template.yaml \
-p NAME="some_name" \
-p GITLAB_HOST="example.com" \
-p REGISTRATION_TOKEN="$(echo -n some_token | base64)" \
-p GITLAB_RUNNER_VERSION="v14.9.1" \
-p TLS_CA_CERT="$( openssl s_client -showcerts -connect example.com:443 < /dev/null 2>/dev/null | openssl x509 -outform PEM | base64 )" \
-p CONCURRENT="number_of_concurrent_pods" | oc create -f -
```

In some cases the `TLS_CA_CERT` parameter might not be required and can be removed. When required it can also alternatively be provided by first first downloading the Gitlab instance's certificate as:
`openssl s_client -showcerts -connect example.com:443 < /dev/null 2>/dev/null | openssl x509 -outform PEM > /etc/gitlab-runner/certs/gitlab.example.com.crt` 

And set the parameter after using: `-p TLS_CA_CERT="$( cat /etc/gitlab-runner/certs/example.com.crt | base64 )"`. 

In order to delete all created objects:

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

    description: GitLab runner version, e.g "v13.1.2".

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

* TEMPLATE_REPO

    description: A repo url with this template. It might be useful for development puproses.

    required: false

* TEMPLATE_REF

    description: A ref of the repo with this template. It might be useful for development puproses.

    required: false
