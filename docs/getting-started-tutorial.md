# Getting Started with cf-for-k8s

This tutorial guides you through your first `cf push` experience.

In order to get there you first need to deploy cf-for-k8s, the platform that provides the developer experience on top of Kubernetes. So, let's get started!

## Prerequisites

### Machine requirements
This tutorial will guide you through installing a local Kubernetes cluster with the following requirements:

- have a minimum of 1 node
- have a minimum of 4 CPU, 6GB memory per node (preferably 6+ CPU and 8+ GB memory)

### Tooling

You’ll also need a few CLIs before you start:

- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/), the CNCF project for creating Kubernetes clusters locally
- [ytt](https://carvel.dev/#install), the CLI used to render Kubernetes templates
- [kapp](https://carvel.dev/#install), the CLI used to deploy cf-for-k8s
- [yq](https://github.com/mikefarah/yq); a CLI tool for extracting information from YAML documents
- [BOSH CLI](https://bosh.io/docs/cli-v2-install/#install); a handy tool to generate self signed certs and passwords, used by `generate-values`
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git); the tool we use to interact with the cf-for-k8s repository
- a [Docker Hub](https://hub.docker.com) account; cf-for-k8s will use this account to store your application images
- [cf](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html), the CLI used to interact with Cloud Foundry

## Installing Cf-for-K8s

1. First things first: git clone the cf-for-k8s repository, which contains our templated k8s YAML files.

    ```
    git clone https://github.com/cloudfoundry/cf-for-k8s.git -b main
    cd cf-for-k8s
    TMP_DIR=<your-tmp-dir-path>
    mkdir -p ${TMP_DIR}
    ```
    Note: if you would like to use the latest release, replace the branch reference in the clone command with that release tag. (e.g. `-b v1.0.0`)

1. Next, create the local Kubernetes cluster that we will deploy cf-for-k8s to presently.

    ```
    kind create cluster --config=./deploy/kind/cluster.yml --image kindest/node:v1.19.1
    ```
    Note: the versions of Kubernetes that cf-for-k8s supports are [here](https://github.com/cloudfoundry/cf-for-k8s/blob/master/supported_k8s_versions.yml).

1. Create your cf values file, which will be used to configure your deployment.

    ```
    $ ./hack/generate-values.sh -d vcap.me > ${TMP_DIR}/cf-values.yml
    $ cat << EOF >> ${TMP_DIR}/cf-values.yml
    app_registry:
      hostname: https://index.docker.io/v1/
      repository_prefix: "<my_username>"
      username: "<my_username>"
      password: "<my_password>"

    add_metrics_server_components: true
    enable_automount_service_account_token: true
    load_balancer:
      enable: false
    metrics_server_prefer_internal_kubelet_address: true
    remove_resource_requirements: true
    use_first_party_jwt_tokens: true
    EOF
    ```

    Notes:
    1. You must supply an OCI-compliant container registry. Docker Hub is recommended.
    2. The additional properties configure your cf-for-k8s to run on a local kind cluster.
    3. `-d vcap.me` sets the root DNS domain name for the CF install to `vcap.me`, a handy domain whose subdomains always resolve to `127.0.0.1` (a machine's local IP).

1. With our input values configured we are now ready to deploy cf-for-k8s:

    ```
    $ kapp deploy -a cf -f <(ytt -f config -f ${TMP_DIR}/cf-values.yml)
    ```

    Running this command should take about 10 minutes or less. `kapp` will provide updates as it progresses:

    ```
    ...
    4:08:19PM: ---- waiting on 1 changes [0/1 done] ----
    4:08:19PM: ok: reconcile serviceaccount/cc-kpack-registry-service-account (v1) namespace: cf-workloads-staging
    4:08:19PM: ---- waiting complete [5/10 done] ----
    ...
    ```

1. When deployment has finished you can log into Cloud Foundry:

    ```
    $ cf api api.vcap.me --skip-ssl-validation
    $ cf auth admin $(yq read ${TMP_DIR}/cf-values.yml 'cf_admin_password')
    ```

## Experiencing your first `cf push`

1. Before we can push an application we need to create an organization and space for your application to exist in.

    ```
    $ cf create-org my-org
    $ cf create-space my-space -o my-org
    $ cf target -o my-org -s my-space
    ```

    Note: Cloud Foundry supports the development workflows of very large organizations; you can read more about how Cloud Foundry achieves that [here](https://docs.cloudfoundry.org/concepts/roles.html).

1. Now, we are ready to push our first app:

    ```
    $ cd ..
    $ git clone https://github.com/cloudfoundry-samples/test-app.git
    $ cd test-app
    $ cf push test-app
    ```

    You might notice that what we pushed is source code, and not something like a container image or Kubernetes manifest. This highlights some of the magic of Cloud Foundry: it can perform the staging and deployment of your application from source code, letting you focus on more important things.

1. Once deployed, open your browser and visit https://test-app.vcap.me :

    ![](./assets/test-app.png)

Congratulations, you just pushed your first application to Cloud Foundry! As the tagline says:

> Here is my source code.  
Run it on the cloud for me.  
I do not care how.
