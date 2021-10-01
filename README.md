# Install gitlab

```
kubectl create ns gitlab 

helm -n gitlab upgrade --install   --timeout 600s --set global.hosts.domain=example.com .

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

