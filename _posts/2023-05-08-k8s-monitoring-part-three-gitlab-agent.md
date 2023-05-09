---
layout: post
title: Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 3
summary: How to connect a Kubernetes cluster to Gitlab
featured-img:
categories: Linux Monitoring Kubernetes
tags: [ grafana, prometheus, loki, kubernetes, notes, linux, gitlab ]
---
- [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 1. Kubernetes cluster](https://timeforplanb123.github.io/k8s-monitoring-part-one-k8s-cluster/)
- [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 2. SNMP](https://timeforplanb123.github.io/k8s-monitoring-part-two-snmp/)
- [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 3. Gitlab Agent](https://timeforplanb123.github.io/k8s-monitoring-part-three-gitlab-agent/)

In this part, I wanted to write about the [SNMP Exporter](https://github.com/prometheus/snmp_exporter){:target="_blank"}, but I decided that it was time to think about automating changes in the cluster. Therefore, here I will connect the Kubernetes cluster to Gitlab using the Gitlab Agent. The result will be the ability to manage the cluster using the Gitlab pipeline. All this in the "How to" format .

Now I have a Kubernetes cluster, deployed on a Desktop machine (see Kubernetes manifests from [Part 1](https://timeforplanb123.github.io/k8s-monitoring-part-one-k8s-cluster/){:target="_blank"}. This is a development environment. This is where I constantly make changes and apply them manually using `kubectl apply`. Automating changes, using Gitlab + Gitlab Agent, in the development cluster is redundant, so I will create a virtual machine on Ubuntu Server 22.04 and deploy a Production Kubernetes cluster on it, which I will connect to Gitlab. I will use a non-public network and self-hosted services.

So, I build the local environment on the Production machine (new machine on Ubuntu 22.04):
- [isntall kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management){:target="_blank"}
- [install minikube](https://minikube.sigs.k8s.io/docs/start/){:target="_blank"}
- [install docker](https://docs.docker.com/engine/install/ubuntu/){:target="_blank"}
- [install ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/#minikube){:target="_blank"}
- copy Kubernetes manifests from Desktop to Production, apply them with `kubectl apply` command

Besides the new Production cluster, I need Gitlab and DNS. I already have a Gitlab 14.8 [Omnibus installation](https://docs.gitlab.com/omnibus/){:target="_blank"}, deployed on a separate machine, which is available at `http://gitlab.my.domain`. In Gitlab I created a k8s repository and added Kubernetes manifests from [Part 1](https://timeforplanb123.github.io/k8s-monitoring-part-one-k8s-cluster/){:target="_blank"} to it.

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/k8s_repo_manifests_only.png)

k8s - directory with manifests (grafana, loki, prometheus contain corresponding manifests):

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/k8s_repo_k8s_dir.png)

ingress - directory with Ingress file:

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/k8s_repo_ingress_dir.png)


## Kubernetes and Gitlab integration

### https

I start with this [link](https://docs.gitlab.com/ee/user/clusters/agent/ci_cd_workflow.html#enable-tls){:target="_blank"}. Here, the Gitlab Agent should interact with the Gitlab Kubernetes Agent Server (KAS) only via https, because credentials are not transmitted on http, and `kubectl` command, running in the Gitlab pipeline, will show this error: `error: You must be logged in to the server (the server has asked for the client to provide credentials)`.

### Certificates 

So, a certificate is a public key signed with the private key of a certification authority. To sign, you can:
- use a public certification authority, for example, LetsEncrypt
- create your own certificate authority
- use the private key that was used to generate the public key (without CA). It's self-signed certificate

The public part of the private key, used to sign the certificate, must be added to all web browsers that will trust the certificate.

So, I connect to the machine with Gitlab Omnibus, here I will create a certificate for Gitlab (this option is suitable for simple installations or educational purposes). This certificate will be used by Gitlab and Gitlab Kubernetes Agent Server (KAS). Gitlab will work on https.

```text
mkdir ~/certs && cd ~/certs

# create a private key with a length of 2048 bits
openssl genrsa -out gitlab.my.domain.key 2048

# create a certificate signing request (csr) to get a certificate
# write Common Name (CN) field only - gitlab.my.domain as an example
openssl req -key gitlab.my.domain.key -new -out gitlab.my.domain.csr
```

Now I need to sign the received .csr file with the private key of the certification authority. Since I don't have a public Gitlab, I won't be using public CAs (but this option can be used for non-public services). I can either sign the .csr file with the private key I created earlier (gitlab.my.domain.key) or create my own CA. For this task, the first option is fine, but I like the second one better.

I create my own certificate authority(CA). This is the directory with the self-signed CA certificate:

```text
mkdir ~/ca && cd ~/ca

# create a private key, certificate signing request and a public key (self-signed CA certificate) with one command 
openssl req -newkey rsa:2048 -nodes -keyout ca.key -x509 -days 3654 -out ca.crt
```

I sign the certificate signing request (.csr file) received earlier. From a certain version, Kubernetes considers the Common Name (CN) field as legacy and asks to use SAN (subjectAltName), so I add this field (SAN) to the certificate:

```text
openssl x509 -req -extfile <(printf "subjectAltName=DNS:gitlab.my.domain") -in ~/certs/gitlab.my.domain.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out gitlab.my.domain.crt -days 365
```

`3654` and `365` - lifetime of certificates. Here as an example. It is logical that CA has a larger number. And now I have a certificate for Gitlab + Gitlab KAS - `gitlab.my.domain.crt`.


### Gitlab + https

Now, I have the certificate (public key - `gitlab.my.domain.crt` file) and private key (`gitlab.my.domain.key`), I will configure Gitlab to work with https.

The Gitlab configuration is stored in the `/etc/gitlab/gitlab.rb` file. I do according to the [manual](https://docs.gitlab.com/omnibus/settings/ssl/#configure-https-manually){:target="_blank"}:

```text
# open Gitlab configuration file in vim editor
sudo vi /etc/gitlab/gitlab.rb

# configure this options
external_url "https://gitlab.my.domain"
letsencrypt['enable'] = false

# enable redirect from http to https:
nginx['redirect_http_to_https'] = true

# see this issue - https://gitlab.com/gitlab-org/omnibus-gitlab/-/issues/7356
gitlab_kas['env'] = {
    'SSL_CERT_DIR' => '/opt/gitlab/embedded/ssl/certs'
}
```

```text
# copy private key and certificate to /etc/gitlab/ssl
sudo mkdir -p /etc/gitlab/ssl
sudo chmod 755 /etc/gitlab/ssl
sudo cp gitlab.my.domain.key gitlab.my.domain.crt /etc/gitlab/ssl/
```

```text
# copy certificate to /etc/gitlab/trusted-certs
# see this issue - https://gitlab.com/gitlab-org/omnibus-gitlab/-/issues/7356
sudo cp gitlab.my.domain.crt /etc/gitlab/trusted-certs/
```

Allow https in iptables, if needed, and reconfigure gitlab:

```text
sudo gitlab-ctl reconfigure
```

Gitlab will now be available at `https://gitlab.my.domain`. [And here is usefull link for troubleshooting SSL](https://docs.gitlab.com/omnibus/settings/ssl/ssl_troubleshooting.html){:target="_blank"}


## Gitlab Runner

To run the Gitlab pipeline, I need to install [Gitlab Runner](https://docs.gitlab.com/runner/){:target="_blank"}. There can be many Gitlab Runners, it is recommended to install them on separate machines, not on the Gitlab machine. For this tutorial, I will make an exception to the rule and register 1 runner on the Gitlab machine. [Documentation](https://docs.gitlab.com/runner/install/){:target="_blank"} has many installation options, I will choose [Docker](https://docs.gitlab.com/runner/install/docker.html){:target="_blank"}

First, I need to generate a token for the new runner. Log in to `Gitlab UI as root > Configure GitLab > Features > Shared Runners > Register an Instance Runner > Copy registration token`.

Next, I add the public part of the previously generated `ca.crt` key to the `/srv/gitlab-runner/config/certs/` directory (Gitlab Runner will work over https and must trust my ca certificate). [The directory can be changed](https://docs.gitlab.com/runner/install/docker.html#installing-trusted-ssl-server-certificates){:target="_blank"}. This directory will be mounted into the Gitlab Runner container:

```text
# copy ca.crt to certs directory 
sudo cp ~/ca/ca.crt /srv/gitlab-runner/config/certs/

# install ant register a new Gitlab Runner
docker run --rm -it -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register
# Gitlab server - https://gitlab.my.domain
# paste the preveously generated token from web UI Gitlab
# runner name - docker-runner0
# tag to run in the Gitlab pipeline - docker
```

Now the new Runner will appear in the Gitlab web UI, the status should be `online`.

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/gitlab_runner_online.png)


## Kubernetes Agent Server and Kubernetes Agent

### Kubernetes Agent Server (KAS)

At first, enable Kubernetes Agent Server on Gitlab server ([Docs for Omnibus installation](https://docs.gitlab.com/ee/administration/clusters/kas.html#for-omnibus){:target="_blank"}:

```text
sudo vi /etc/gitlab/gitlab.rb
```

```text
# configure /etc/gitlab/gitlab.rb 
gitlab_kas['enable'] = true
```

Reconfigure Gitlab:

```text
sudo gitlab-ctl reconfigure
```

### Gitlab Agent

And now it's time to install the Gitlab Agent to my Kubernetes Production cluster. [Here is link from docs](https://docs.gitlab.com/ee/user/clusters/agent/install/){:target="_blank"} step by step:

- Create an agent configuration file `.gitlab/agents/k8s/config.yaml` in default branch `k8s` Gitlab repository with Gitlab UI

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/k8s_repo_with_gitlab_agent.png)

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/gitlab_agent_config.png)

- Authorize the agent to access your projects (if the project is the only one - this is optional, see [docs](https://docs.gitlab.com/ee/user/clusters/agent/ci_cd_workflow.html#authorize-the-agent){:target="_blank"}) with `.gitlab/agents/k8s/config.yaml` file:

    ```text
    # add to .gitlab/agents/k8s/config.yaml
    ci_access:
      projects:
        - id: my_gitlab_username/project
    ```
- Register the agent with GitLab UI. Select `k8s Gitlab project > Infrastructure > Kubernetes clusters > Actions > Connect with Agent > Select an Agent (k8s agent) > Register Agent` and copy the command under "Recommended installation method". For my Gitlab 14.8 it looks like:
    ```text
    docker run --pull=always --rm     registry.gitlab.com/gitlab-org/cluster-integration/gitlab-agent/cli:stable generate     --agent-token=my_token --kas-address=wss://gitlab.my.domain//-/kubernetes-agent     --agent-version stable     --namespace gitlab-kubernetes-agent
    ```

But, before installing gitlab-agent in the Production cluster with this command, I need to add `ca.crt` to the list of trusted certificates. This [issue](https://gitlab.com/gitlab-org/gitlab/-/issues/280518){:target="_blank"} describes how this can be done. I'll just repeat on a machine with a Production Kubernetes cluster:

```text
# create generic secret with encoded ca.crt
kubectl create secret generic 'ca' --namespace gitlab-kubernetes-agent --from-file="~/crt.ca"

# save kas configuration to kas.yaml
docker run --pull=always --rm     registry.gitlab.com/gitlab-org/cluster-integration/gitlab-agent/cli:stable generate     --agent-token=my_token --kas-address=wss://gitlab.my.domain//-/kubernetes-agent     --agent-version stable     --namespace gitlab-kubernetes-agent > kas.yaml

# open kas.yaml with vi
vi kas.yaml

# search kind: Deployment section 
# add ca.crt to spec:template:spec:containers:args
spec:
  containers:
  - args:
    - --token-file=/config/token
    - --kas-address
    - wss://gitlab.my.domain/-/kubernetes-agent/
    - --ca-cert-file=/certs/ca.crt

# add new volume with ca.crt secret object to spec:template:spec:volumes
volumes:
- name: custom-certs
  secret:
    secretName: ca

# mount new custom-certs volume to /certs directory in spec:template:spec:contaiers:volumeMounts section 
volumeMounts:
- name: custom-certs
  readOnly: true
  mountPath: /certs

# save and close kas.yaml with vi
:wq

# apply file with kubectl
kubectl apply -f kas.yaml
```

Now the agent status in Gitlab UI (`k8s project > Infrastructure > Kubernetes clusters`) will change to `Connected`.

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/gitlab_agent_status.png)


## Gitlab pipeline

Now my `k8s` repository contains Kubernetes manifests and authorization file for Gitlab Agent. I will add new `.gitlab-ci.yml` file in k8s repo with Gitlab UI.

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/k8s_repo.png)

`.gitlab-ci.yml` here is a simple 3 stage pipeline file (validate, deploy, report) just for example and testing the work Gitlab Agent with Kubernetes cluster:

```yaml
stages:
    - validate
    - deploy 
    - report 

default:
  tags:
    - docker

variables:
  KUBE_CONTEXT: my_gitlab_user/k8s:k8s

before_script:
  - kubectl config get-contexts
  - kubectl config get-clusters
  - kubectl config set-cluster gitlab --certificate-authority="$KAS_CA"
  - if [ -n "$KUBE_CONTEXT" ]; then kubectl config use-context "$KUBE_CONTEXT"; fi

lint-kubeconform:
  stage: validate
  image:
    name: ghcr.io/yannh/kubeconform:latest-alpine
    entrypoint: [""]
  before_script:
    - ''
  script:
  - ~/../kubeconform ./k8s/

deploying:
  stage: deploy
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  script:
  - kubectl apply -R -f ./k8s

viewing-resources:
  stage: report
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  script:
  - kubectl get deployment,pod,services,cm,pv,secrets -A
  - kubectl events -A
```

About this pipeline:
- pipeline contains 3 stages: validate, deploy, report. Validate stage uses [kubeconform validation tool](https://github.com/yannh/kubeconform){:target="_blank"}. Here I specified a k8s directory, that does not contain Kubernetes manifest (the k8s directory contains subdirectories with Kubernetes manifests), so validation will always succeed. Validation is not the purpose of this "how to" and is just an example here
- `tags: docker` runs pipeline in previously configured Gitlab runner
- `variables: KUBE_CONTEXT: my_gitlab_user/k8s:k8s` - the result of the `kubectl config get-contexts` command running in the pipeline (first `k8s` is the name of the repository, second `k8s` is the name of the directory with the `config.yaml` agent file)
- the most interesting is in the `before_script` section. Gitlab Agent can communicate with Gitlab KAS used `ca.crt` certificate, but kubectl, executed in pipeline, will fail to communicate with KAS, because k8s uses an its own self-generated certificate by default (check KUBECONFIG configuration with `kubect config view` command) and k8s client doesn't trust the certificate authority (CA) that signed KAS certificate. For example, `kubectl get pods` command, running in a pipeline, will show the following error:
    ```text
    $ kubectl get pods
    Unable to connect to the server: x509: certificate signed by unknown authority
    ```
    See [docs](https://docs.gitlab.com/ee/user/clusters/agent/ci_cd_workflow.html#environments-with-kas-that-use-self-signed-certificates){:target="_blank"}. 
    I like the solution from this [issue](https://gitlab.com/gitlab-org/gitlab/-/issues/280518#note_844576274){:target="_blank"}. KUBECONFIG in the pipeline must trust KAS ca.crt. Following this instruction, I created CI/CD variable with KAS_CA name and added the content of ca.crt there. The `kubectl config set-cluster gitlab --certificate-authority="$KAS_CA"` command will add ca.crt to the list oftrusted certificates in the gitlab cluster before deploy and report stages. 

    Other related issues:

    - [issue 1](https://gitlab.com/gitlab-org/gitlab/-/issues/366955){:target="_blank"}. Here is useful [note](https://gitlab.com/gitlab-org/gitlab/-/issues/366955#note_1190457929){:target="_blank"} about SSL_CERT_FILE and apk usage

    - [issue 2](https://gitlab.com/gitlab-org/gitlab/-/issues/366957){:target="_blank"}

Now, if I update the Kubernetes manifests in the Development environment and push changes to Gitlab, the Gitlab pipeline will update the Production Kubernetes cluster

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/kubernetes_monitoring/gitlab_agent/pipeline.png)


## Troubleshooting

How to check KAS logs (for Gitlab Omnibus):
```text
gitlab-ctl tail gitlab-kas
# or
sudo tail -f /var/log/gitlab/gitlab-kas/current
```

How to check Gitlab Kubernetes Agent logs:
```text
kubectl logs -f -l=app=gitlab-agent -n gitlab-kubernetes-agent
```

And docs:
- [KAS troubleshooting docs](https://docs.gitlab.com/ee/administration/clusters/kas.html#troubleshooting){:target="_blank"}
- [Gitlab Agent troubleshooting docs](https://docs.gitlab.com/ee/user/clusters/agent/troubleshooting.html){:target="_blank"}


## How to "git clone"

Since my Gitlab installation uses a self-signed certificate, "git clone" for https `k8s` repository might look like this:

```text
GIT_SSL_NO_VERIFY=true git clone https://gitlab.my.domain/my_gitlab_user/k8s.git

cd k8s
git config --local http.sslVerify "false"
```
