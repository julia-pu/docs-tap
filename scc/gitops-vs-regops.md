# GitOps vs. RegistryOps

Regardless of the supply chain that a workload goes through, in the end,
some Kubernetes configuration is pushed to an external entity, either to a Git
repository or to a container image registry.

For example:

  ```console
  Supply Chain

    -- fetch source
      -- test
        -- build
          -- scan
            -- apply-conventions
              -- push config        * either to Git or Registry
  ```

This topic dives into the specifics of that last phase of the supply chains
by pushing configuration to a Git repository or an
image registry.

>**Note:** For more information about providing source code either from a
local directory or Git repository,
see [Building from Source](building-from-source.md).  

## <a id="gitops"></a>GitOps

The GitOps approach differs from local iteration in that GitOps configures the supply
chains to push the Kubernetes configuration to a remote Git repository. This
allows users to compare configuration changes and promote those changes through
environments by using GitOps principles.

Typically associated with an outerloop workflow, the GitOps approach is only activated if certain
parameters are set in the supply chain:

- `gitops.repository_prefix`: configured during the Out of the Box Supply
  Chains package installation.

- `gitops_repository`: configured as a workload parameter.

For example, assuming the installation of the supply chain packages through
Tanzu Application Platform profiles and a `tap-values.yaml`:

  ```yaml
  ootb_supply_chain_basic:
    registry:
      server: REGISTRY-SERVER
      repository: REGISTRY-REPOSITORY

    gitops:
      repository_prefix: https://github.com/my-org/
  ```

Workloads in the cluster with the
Kubernetes configuration produced throughout the supply chain is pushed to
the repository whose name is formed by concatenating
`gitops.repository_prefix` with the name of the workload. In this case,
for example, `https://github.com/my-org/$(workload.metadata.name).git`.

  ```console
  Supply Chain
    params:
        - gitops_repository_prefix: GIT-REPO_PREFIX


  workload-1:
    `git push` to GIT-REPO-PREFIX/workload-1.git

  workload-2:
    `git push` to GIT-REPO-PREFIX/workload-2.git

  ...

  workload-n:
    `git push` to GIT-REPO-PREFIX/workload-n.git
  ```


Alternatively, you can force a workload to publish the configuration
in a Git repository by providing the `gitops_repository` parameter
to the workload:

  ```bash
  tanzu apps workload create tanzu-java-web-app \
    --app tanzu-java-web-app \
    --type web \
    --git-repo https://github.com/sample-accelerators/tanzu-java-web-app \
    --git-branch main \
    --param gitops_ssh_secret=GIT-SECRET-NAME \
    --param gitops_repository=https://github.com/my-org/config-repo
  ```

In this case, at the end of the supply chain, the configuration for this
workload is published to the repository provided under the `gitops_repository`
parameter.

### <a id="auth"></a>Authentication

Regardless of how the supply chains are configured, if pushing to
Git is configured by repository prefix or repository name, you must provide
credentials for the remote provider (for example, GitHub) by using a Kubernetes
secret in the same namespace as the workload attached to the workload
`ServiceAccount`.

Because the operation of pushing requires elevated permissions, credentials are
required by both public and private repositories.

#### <a id="http-auth"></a>HTTP(S) Basic-auth / Token-based authentication

If the repository at which configuration is published uses
`https://` or `http://` as the URL scheme, the Kubernetes secret must
provide the credentials for that repository as follows:

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: GIT-SECRET-NAME  # `git-ssh` is the default name.
                          #   - operators can change such default by using the
                          #     `gitops.ssh_secret` property in `tap-values.yaml`
                          #   - developers can override by using the workload parameter
                          #     named `gitops_ssh_secret`.
    annotations:
      tekton.dev/git-0: GIT-SERVER        # ! required
  type: kubernetes.io/basic-auth          # ! required
  stringData:
    username: GIT-USERNAME
    password: GIT-PASSWORD
  ```

>**Note:** Both the Tekton annotation and the `basic-auth` secret type must be
set. `GIT-SERVER` must be prefixed with the appropriate URL scheme and the Git
server. For example, for https://github.com/vmware-tanzu/cartographer,
https://github.com must be provided as the GIT-SERVER.

After the `Secret` is created, attach it to the `ServiceAccount` used by the
workload. For example:

  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: default
  secrets:
    - name: registry-credentials
    - name: tap-registry
    - name: GIT-SECRET-NAME
  imagePullSecrets:
    - name: registry-credentials
    - name: tap-registry
  ```

For more information about the credentials and setting up the Kubernetes
secret, see [Git Authentication's HTTP section](git-auth.md#http).

### <a id="ssh"></a>SSH

If the repository to which configuration is published uses
`https://` or `http://` as the URL scheme, the Kubernetes secret must
provide the credentials for that repository as follows:

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: GIT-SECRET-NAME  # `git-ssh` is the default name.
                          #   - operators can change such default via the
                          #     `gitops.ssh_secret` property in `tap-values.yaml`
                          #   - developers can override by using the workload parameter
                          #     named `gitops_ssh_secret`.
    annotations:
      tekton.dev/git-0: GIT-SERVER
  type: kubernetes.io/ssh-auth
  stringData:
    ssh-privatekey: SSH-PRIVATE-KEY     # private key with push-permissions
    identity: SSH-PRIVATE-KEY           # private key with pull permissions
    identity.pub: SSH-PUBLIC-KEY        # public of the `identity` private key
    known_hosts: GIT-SERVER-PUBLIC-KEYS # git server public keys
  ```

After the `Secret` is created, attach it to the `ServiceAccount` used by the
workload. For example:

  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: default
  secrets:
    - name: registry-credentials
    - name: tap-registry
    - name: GIT-SECRET-NAME
  imagePullSecrets:
    - name: registry-credentials
    - name: tap-registry
  ```

For more information about the credentials and setting up the Kubernetes
secret, see [Git Authentication's SSH section](git-auth.md#sh).


### <a id="gitops-params"></a>GitOps workload parameters

While installing `ootb-*`, operators can configure `gitops.repository_prefix` to
indicate what prefix the supply chain must use when forming the name of the
repository to push to the Kubernetes configurations produced by the supply
chains.

To change the behavior to use GitOps,
set the source of the source code to a Git repository. As the
supply chain progresses, configuration is pushed to a repository named
after `$(gitops.repository_prefix) + $(workload.name)`.

For example, configure `gitops.repository_prefix` to `git@github.com/foo/` and
create a workload as follows:

  ```console
  tanzu apps workload create tanzu-java-web-app \
    --git-branch main \
    --git-repo https://github.com/sample-accelerators/tanzu-java-web-app
    --label app.kubernetes.io/part-of=tanzu-java-web-app \
    --type web
  ```

Expect to see the following output:

  ```console
  Create workload:
        1 + |---
        2 + |apiVersion: carto.run/v1alpha1
        3 + |kind: Workload
        4 + |metadata:
        5 + |  labels:
        6 + |    apps.tanzu.vmware.com/workload-type: web
        7 + |    app.kubernetes.io/part-of: tanzu-java-web-app
        8 + |  name: tanzu-java-web-app
        9 + |  namespace: default
      10 + |spec:
      11 + |  source:
      12 + |    git:
      13 + |      ref:
      14 + |        branch: main
      15 + |      url: https://github.com/sample-accelerators/tanzu-java-web-app
  ```

As a result, the Kubernetes configuration is pushed to
`git@github.com/foo/tanzu-java-web-app.git`.

Regardless of the setup, developers can also manually override the repository
where configuration is pushed to by tweaking the following parameters:

-  `gitops_ssh_secret`: Name of the secret in the same namespace as the
   workload where SSH credentials exist for pushing the configuration produced
   by the supply chain to a Git repository.
   Example: `ssh-secret`

-  `gitops_repository`: SSH URL of the Git repository to push the Kubernetes
   configuration produced by the supply chain to.
   Example: `ssh://git@foo.com/staging.git`

-  `gitops_branch`: Name of the branch to push the configuration to.
   Example: `main`

-  `gitops_commit_message`: Message to write as the body of the commits
   produced for pushing configuration to the Git repository.
   Example: `ci bump`

-  `gitops_user_name`: User name to use in the commits.
   Example: `Alice Lee`

-  `gitops_user_email`: User email address to use in the commits.
   Example: `alice@example.com`


## <a id="registryops"></a>RegistryOps

RegistryOps is typically used for inner loop flows where configuration is treated as an
artifact from quick iterations by developers. In this scenario, at the end
of the supply chain, configuration is pushed to a container image registry in
the form of an [imgpkg bundle](https://carvel.dev/imgpkg/docs/v0.27.0/). You can think
of it as a container image whose sole purpose is to carry arbitrary files.

To enable this mode of operation, the supply chains must be configured
**without** the following parameters being configured during the installation
of the `ootb-` packages or overwritten by the workload by using the following
parameters:

- `gitops_repository_prefix`
- `gitops_repository`

If none of the parameters are set, the configuration is pushed to the
same container image registry as the application image. That is, to
the registry configured under the `registry: {}` section of the `ootb-`
values.

For example, assuming the installation of Tanzu Application Platform by using
profiles, configure the `ootb-supply-chain*` package as follows:

  ```yaml
  ootb_supply_chain_basic:
    registry:
      server: REGISTRY-SERVER
      repository: REGISTRY_REPOSITORY
  ```

The Kubernetes configuration produced by the supply chain is pushed
to an image named after `REGISTRY-SERVER/REGISTRY-REPOSITORY` including
the workload name.

In this scenario, no extra credentials need to be set up, because the secret
containing the credentials for the container image registry were already configured during the setup of the workload namespace.
