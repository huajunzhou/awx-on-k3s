<!-- omit in toc -->
# Workaround for the rate limit on Docker Hub

If your Pod for PostgreSQL is in `ErrImagePull` and its `Events` shows following events, this is due to [the Rate Limit on Docker Hub](https://docs.docker.com/docker-hub/download-rate-limit/).

```bash
$ kubectl -n awx describe pod awx-postgres-0
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  ...
  Warning  Failed            2s    kubelet            Failed to pull image "postgres:12": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/library/postgres:12": failed to copy: httpReadSeeker: failed open: unexpected status code https://registry-1.docker.io/v2/library/postgres/manifests/sha256:...: 429 Too Many Requests - Server message: toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
  ...
```

If you just follow the steps in this repository to deploy you AWX, your pull request to Docker Hub will be identified as a free, anonymous account. Therefore, you will be limited to 200 requests in 6 hours. The message "429 Too Many Requests" indicates that the limit has been exceeded. To solve this, you should pass your Docker Hub credentials to the Pod as ImagePullSecrets.

<!-- omit in toc -->
## Table of Contents

- [Procedure](#procedure)
  - [Create `base/config.json`](#create-baseconfigjson)
  - [Modify `base/kustomization.yaml`](#modify-basekustomizationyaml)
  - [Modify `base/awx.yaml`](#modify-baseawxyaml)
  - [Next Step](#next-step)
- [Appendix: Create Secret for ImagePullSecrets by Hand](#appendix-create-secret-for-imagepullsecrets-by-hand)
- [Appendix: Modify ImagePullSecrets for the Specific Service Account](#appendix-modify-imagepullsecrets-for-the-specific-service-account)
- [Appendix: Apply Docker Hub Credential at Cluster Level](#appendix-apply-docker-hub-credential-at-cluster-level)

## Procedure

There are several ways to achieve this, but this guide will show you an example of using Kustomize to save all your configuration as code. In addition to [the main guide](../), a few additional files need to be modified.

### Create `base/config.json`

First, prepare JSON file named `config.json` under your `base` directory.

```bash
$ DOCKERHUB_USERNAME=<Your Username for Docker Hub>
$ DOCKERHUB_PASSWORD=<Your Password or Personal Access Token for Docker Hub>
$ DOCKERHUB_AUTH=$(echo -n "${DOCKERHUB_USERNAME}:${DOCKERHUB_PASSWORD}" | base64)
$ cat <<EOF > base/config.json
{
    "auths": {
        "docker.io": {
            "auth": "${DOCKERHUB_AUTH}"
        }
    }
}
EOF
```

Ensure your `config.json` includes base64-encoded string as following. Note that this base64-encoded string includes your password in plain text and easily revealed by `echo "<Encoded String>" | base64 --decode` command.

```bash
$ cat base/config.json
{
    "auths": {
        "docker.io": {
            "auth": "ZXhhbXBsZS...MtdG9rZW4K"
        }
    }
}
```

### Modify `base/kustomization.yaml`

Then, add following four lines to under `secretGenerator` in `base/kustomization.yaml`.

```yaml
...
secretGenerator:
  ...
  - name: awx-registry-secret     👈👈👈
    type: kubernetes.io/dockerconfigjson     👈👈👈
    files:     👈👈👈
      - .dockerconfigjson=config.json     👈👈👈
  ...
resources:
  ...
```

### Modify `base/awx.yaml`

Finally, add following line to `base/awx.yaml`.

```yaml
...
spec:
  ...
  image_pull_secret: awx-registry-secret     👈👈👈
  ...
```

### Next Step

Now everything is ready to deploy, go back to [the main guide (`README.md`)](../) and run `kubectl apply -k base`.

## Appendix: Create Secret for ImagePullSecrets by Hand

If you want to create Secret manually instead of using Kustomize, you can choose alternative way. If you don't have `awx` namespace, create it first by `kubectl create ns awx`.

- **Method 1**: Create Secret by using existing `~/.docker/config.json`

  ```bash
  # Log in to Docker Hub via docker command first,
  $ docker login

  # Then create Secret by specifying ~/.docker/config.json which generated by Docker
  $ kubectl -n awx create secret generic awx-registry-secret \
      --from-file=.dockerconfigjson=${HOME}/.docker/config.json \
      --type=kubernetes.io/dockerconfigjson
  ```

- **Method 2**: Create Secret by specifying username and password manually

  ```bash
  # Create Secret by specifying username and password manually
  $ DOCKERHUB_USERNAME=<Your Username for Docker Hub>
  $ DOCKERHUB_PASSWORD=<Your Password or Personal Access Token for Docker Hub>
  $ kubectl -n awx create secret docker-registry awx-registry-secret \
      --docker-server=docker.io \
      --docker-username=${DOCKERHUB_USERNAME} \
      --docker-password=${DOCKERHUB_PASSWORD}
  ```

## Appendix: Modify ImagePullSecrets for the Specific Service Account

Once create new namespace, the default Service Account named `default` is also created by default. This `default` Service Account is used when no Service Account has been specified for the Pod, and you can modify ImagePullSecrets which used by default for this `default` Service Account too. This can be applied to [Private Git Repository](../git) and [Private Container Registry](../registry) included in this repository. Additionally, this can also be applied to the Pod for PostgreSQL created by AWX Operator, since the Service Account is not specified.

First, you should create Secret for the Service Account by referring [Appendix: Create Secret for ImagePullSecrets by Hand](#appendix-create-secret-for-imagepullsecrets-by-hand). Note that the namespace and the name of the Secret in the command should be changed to suit your environment.

Then patch the `default` service account.

```bash
kubectl -n <Your Namespace> patch serviceaccount default -p '{"imagePullSecrets": [{"name": "<Your Secret>"}]}'
```

## Appendix: Apply Docker Hub Credential at Cluster Level

You can also change the entire K3s configuration so that the specific credential for Docker Hub is always used, regardless of the namespace. Create `/etc/rancher/k3s/registries.yaml` and restart K3s service as follows.

```bash
# Create /etc/rancher/k3s/registries.yaml including your credential
$ DOCKERHUB_USERNAME=<Your Username for Docker Hub>
$ DOCKERHUB_PASSWORD=<Your Password or Personal Access Token for Docker Hub>
$ sudo tee /etc/rancher/k3s/registries.yaml <<EOF
configs:
  registry-1.docker.io:
    auth:
      username: ${DOCKERHUB_USERNAME}
      password: ${DOCKERHUB_PASSWORD}
EOF

# Then restart K3s. The K3s service can be safely restarted without affecting the running resources
$ sudo systemctl restart k3s
```