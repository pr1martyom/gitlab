# Install gitlab

```
helm repo add gitlab https://charts.gitlab.io/

helm repo update

helm pull gitlab/gitlab --untar

cd gitlab

kubectl create ns gitlab 

vi values.yaml

find and change:

---
hosts:
  domain: example.com
  hostSuffix:
  https: false
---

ingress:
  configureCertmanager: true
  provider: nginx
  annotations: {}
  enabled: true
  tls:
    enabled: false
---

nginx-ingress:
  enabled: false
---

helm -n gitlab upgrade --install gitlab --timeout 600s --set certmanager-issuer.email=me@example.com  --set global.hosts.domain=example.com .

https://docs.gitlab.com/charts/installation/deployment.html
```

# Change ingress in kubectl

kubectl -n gitlab edit ingress 

delete from ingress 
```
      kubernetes.io/ingress.class: gitlab-nginx
      kubernetes.io/ingress.provider: nginx
```

add  ingressClassName: nginx-controller

example:
```
spec:
  ingressClassName: nginx-controller
  rules:
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

# Kubernetes gitlab runner example 1
```
link:
https://adambcomer.com/blog/install-gitlab-runner-kubernetes/
          
# gitlab-runner-service-account.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-admin
  namespace: gitlab-runner
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: gitlab-runner
  name: gitlab-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: gitlab-admin
  namespace: gitlab-runner
subjects:
  - kind: ServiceAccount
    name: gitlab-admin
    namespace: gitlab-runner
roleRef:
  kind: Role
  name: gitlab-admin
  apiGroup: rbac.authorization.k8s.io
  
  
# gitlab-runner-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-runner-config
  namespace: gitlab-runner
data:
  config.toml: |-
    concurrent = 4
    [[runners]]
      name = "Kubernetes Demo Runner"
      url = "https://gitlab.com/ci"
      token = "[TOKEN]"
      executor = "kubernetes"
      [runners.kubernetes]
        namespace = "gitlab-runner"
        poll_timeout = 600
        cpu_request = "1"
        service_cpu_request = "200m"
        
# cat runner.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: runner
  namespace: runner
  labels:
    app: runner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: runner
  template:
    metadata:
      labels:
        app: runner
    spec:
      serviceAccountName: runner
      containers:
        - name: runner
          image: "gitlab/gitlab-runner:alpine-v13.2.4"
          volumeMounts:
          - mountPath: /etc/gitlab-runner/config.toml
            name: config
            subPath: config.toml
      volumes:
      - name: config
        configMap:
          name: runner

```

# Kubernetes gitlab runner example 2

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
