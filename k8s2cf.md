### Translate Kubernetes to Cloud Foundry

| Kubernetes                       | Cloud Foundry                   |
|:---------------------------------|:--------------------------------|
| kubectl get pods                 | cf apps                         |
| kubectl get nodes                | bosh vms                        |
| kubectl create namespace         | cf create-space                 |
| kubectl create -f xxxx.yml       | cf push app_name -f xxxx.yml    |
| kubectl apply -f xxxx.yml        | cf push app_name -f xxxx.yml    |
| kubectl set-context yyy --namespace xxxx | cf target -o yyy -s xxxx |
| kubectl create configmap my-config --from-literal=key1=config1 | cf set-env my-app key1 config1 |
