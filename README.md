# Install gitlab

```
kubectl create ns gitlab 



helm -n gitlab upgrade --install gitlab --timeout 600s --set global.hosts.domain=example.com --set global.hosts.https=false --set global.ingress.tls.enabled=false --set nginx-ingress.enabled=false .

https://docs.gitlab.com/charts/installation/deployment.html
```

# Docker gitlab runners

```
1. register runner

docker run -it --name gitlab-runner --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd)/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:alpine-v13.2.4 register


# token can bi found in gitlab - settings - CI/CD - runners

2. run runner
docker run -d --name gitlab-runner --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd)/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:alpine-v13.2.4

for dind need to change privileget to true

Example:

concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "local docker runner"
  url = "https://gitlab.qzhub.net/"
  token = ""
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
  [runners.docker]
    tls_verify = false
    image = "bash:4.4"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
```

# Kubernetes gitlab runner
```
helm repo add gitlab https://charts.gitlab.io

kubectl create namespace gitlab


helm install --namespace gitlab gitlab-runner gitlab/gitlab-runner \
  --set rbac.create=true \
  --set runners.privileged=true \
  --set gitlabUrl= \  # gitlab url
  --set runnerRegistrationToken=  #token

kubectl get po -n gitlab
```


# Preparation for ci cd
```
kubectl create ns ci

kubectl create sa deploy -n ci

kubectl create rolebinding deploy \
  -n ci \
  --clusterrole edit \
  --serviceaccount ci:deploy

kubectl get secret -n ci \
  $(kubectl get sa -n ci deploy \
    -o jsonpath='{.secrets[].name}') \
  -o jsonpath='{.data.token}'

source: https://mcs.mail.ru/blog/launching-a-project-in-kubernetes
```

# CI/CD example
```
stages:
- build
- deploy
- delete

docker-build:
  stage: build

  tags:
    - docker

  image:
    # An alpine-based image with the `docker` CLI installed.
    name: docker:stable

  # This will run a Docker daemon in a container (Docker-In-Docker), which will
  # be available at thedockerhost:2375. If you make e.g. port 5000 public in Docker
  # (`docker run -p 5000:5000 yourimage`) it will be exposed at thedockerhost:5000.
  services:
   - name: docker:dind
     alias: thedockerhost

  variables:
    # Tell docker CLI how to talk to Docker daemon; see
    # https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker-executor
    DOCKER_HOST: tcp://thedockerhost:2375/
    # Use the overlayfs driver for improved performance:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
  script:
    - docker build -t "$CI_REGISTRY_IMAGE" .
    - docker tag "$CI_REGISTRY_IMAGE" "$CI_HARBOR_ADDRESS_ODOO""$CI_PROJECT_NAME":"$CI_PIPELINE_ID"
    - docker push "$CI_HARBOR_ADDRESS_ODOO""$CI_PROJECT_NAME":"$CI_PIPELINE_ID"

helm-chart-deploy:
  stage: deploy
  image: centosadmin/kubernetes-helm:3.1.2
  tags:
  - kubectl
  - kubernetes
  - local
  variables:
    K8S_NAMESPACE: ci
  before_script:
    # Set kubernetes credentials
    - kubectl config set-cluster cluster.qzhub --insecure-skip-tls-verify=true --server=$K8S_API_URL
    - kubectl config set-credentials ci --token="$(echo $K8S_CI_TOKEN | base64 -d)"
    - kubectl config set-context ci --cluster=cluster.qzhub --user=ci --namespace $K8S_NAMESPACE
    - kubectl config use-context ci
  script:
    - helm upgrade --install odoo$CI_PIPELINE_ID .helm --set odoo.name=odoo$CI_PIPELINE_ID --set odoo.label=odoo$CI_PIPELINE_ID --set odoo.pvc.odooData.name=odoo-data$CI_PIPELINE_ID --set odoo.pvc.odooAddons.name=odoo-addons$CI_PIPELINE_ID --set postgres.name=postgres$CI_PIPELINE_ID --set postgres.label=postgres$CI_PIPELINE_ID --set postgres.pvc.name=postgres-data-$CI_PIPELINE_ID --set odoo.ingress.host=testci2.qzhub.net --set odoo.image.repository=harbor.qzhub.net/odoo/odoo_image --set odoo.image.tag=$CI_PIPELINE_ID --set odoo.imagePullSecrets=odooharbor --wait --timeout 300s --atomic --debug


delete-helm-chart:
  stage: delete
  image: centosadmin/kubernetes-helm:3.1.2
  tags:
  - kubectl
  - kubernetes
  - local
  variables:
    K8S_NAMESPACE: ci
  before_script:
    - export KUBECONFIG=/tmp/kubeconfig
    - kubectl config set-cluster cluster.qzhub --insecure-skip-tls-verify=true --server=$K8S_API_URL
    - kubectl config set-credentials ci --token="$(echo $K8S_CI_TOKEN | base64 -d)"
    - kubectl config set-context ci --cluster=cluster.qzhub --user=ci --namespace $K8S_NAMESPACE
    - kubectl config use-context ci
  script:
    - helm delete odoo$CI_PIPELINE_ID
  when: manual
  allow_failure: true
```
