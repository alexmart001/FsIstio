# Istio

Neste repositório estarão disponíveis nosso *Workshop* de implementação fazendo uso da tecnologia [Kubernetes](https://kubernetes.io/), [Minikube](https://minikube.sigs.k8s.io/docs/start/) e [Istio](https://istio.io/)

Esse material é baseado nesse tutorial: [Istio Tutorial](https://github.com/redhat-scholars/istio-tutorial)

## Pré Requisitos

- [Minikube](https://minikube.sigs.k8s.io/docs/start/)

## Implementação

0. [Minikube Setup](#workshop-minikube-setup)
1. [Deploy Customer Microservice](#workshop-deploy-customer-msa)
2. [Deploy Preference Microservice](#workshop-deploy-preference-msa)
3. [Deploy Recommendation Microservice](#workshop-deploy-recommendation-msa)
4. [Observability - Grafana](#workshop-observability-grafana)
5. [Observability - Prometheus](#workshop-observability-prometheus)
6. [Observability - Jaeger](#workshop-observability-jaeger)
7. [Observability - Kiali](#workshop-observability-kiali)
8. [Traffic Control](#workshop-trafficcontrol-simple)
9. [Service Resiliency - Retry](#workshop-serviceresiliency-retry)
10. [Service Resiliency - Timeout](#workshop-serviceresiliency-timeout)

## Implementação

### 0 - Minikube Setup <a name="workshop-minikube-setup">

* Clone/Download do repositório: `git clone https://github.com/redhat-developer-demos/istio-tutorial istio-tutorial`

* Inicie o minikube com os seguintes atributos: `minikube start --memory=8192 --cpus=3 --kubernetes-version=v1.18.6 --vm-driver=virtualbox -p istio-devnation`

* Habilite o serviço do istio: `istioctl manifest apply --set profile=demo --set values.global.proxy.privileged=true`

* Verifique os pods do **Istio**:

  ```
  kubectl config set-context $(kubectl config current-context) --namespace=istio-system

  kubectl get pods -w      
  NAME                                      READY   STATUS    RESTARTS   AGE
  grafana-6b65874977-n76pc                  1/1     Running   1          155d
  istio-citadel-7bcc69d486-6tkjk            1/1     Running   2          155d
  istio-egressgateway-6b6f694c97-vmwsk      1/1     Running   1          155d
  istio-galley-6cf8f58d7b-z888s             1/1     Running   1          155d
  istio-ingressgateway-8c9c9c9f5-jmj2c      1/1     Running   1          155d
  istio-pilot-c6dbc54b9-lw77b               1/1     Running   1          155d
  istio-policy-66c4cf95c5-h9kpb             1/1     Running   5          155d
  istio-sidecar-injector-79ccdf54c4-t7qxb   1/1     Running   1          155d
  istio-telemetry-7bf96cb54f-qdm7t          1/1     Running   5          155d
  istio-tracing-c66d67cd9-fcbcq             1/1     Running   2          155d
  kiali-8559969566-znxgm                    1/1     Running   1          155d
  prometheus-66c5887c86-7nnsv               1/1     Running   1          155d
  ```

### 1 - Deploy Customer Microservice <a name="workshop-deploy-customer-msa">

* Garanta que voce está logado: `kubectl config current-context`

* Crie o namespace *Tutorial*:

  ```
  kubectl create namespace tutorial
  kubectl config set-context $(kubectl config current-context) --namespace=tutorial
  ```

* Para facilitar a operação desse tutorial, crie uma variável de ambiente que referencie o local de *checkout/clone* do repositório:

  ```
  export TUTORIAL_HOME=$PWD
  echo $TUTORIAL_HOME
  ```

* Deployment do **Customer Microservice**

  ```
  kubectl apply -f <(istioctl kube-inject -f $TUTORIAL_HOME/customer/kubernetes/Deployment.yml) -n tutorial

  serviceaccount/customer created
  deployment.apps/customer configured

  kubectl create -f $TUTORIAL_HOME/customer/kubernetes/Service.yml -n tutorial
  service/customer created

  kubectl create -f $TUTORIAL_HOME/customer/kubernetes/Gateway.yml -n tutorial
  gateway.networking.istio.io/customer-gateway created
  virtualservice.networking.istio.io/customer-gateway created

  kubectl get svc istio-ingressgateway -n istio-system

  export INGRESS_HOST=$(minikube ip -p istio-devnation)
  export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

  export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
  ```

* Verifique se **Customer Microservice** foi implementado com sucesso:

  ```
  curl $GATEWAY_URL/customer


  customer => UnknownHostException: preference

  kubectl -n tutorial logs customer-5c4d9c784f-mplvx customer -f
  Caused by: java.net.UnknownHostException: preference
  ```

### 2 - Deploy Preference Microservice <a name="workshop-deploy-preference-msa">

* Deployment do **Preference Microservice**

  ```
  kubectl apply -f <(istioctl kube-inject -f $TUTORIAL_HOME/preference/kubernetes/Deployment.yml)  -n tutorial

  serviceaccount/preference created
  deployment.apps/preference-v1 created

  kubectl create -f $TUTORIAL_HOME/preference/kubernetes/Service.yml -n tutorial
  service/preference created
  ```

* Verifique se **Preference Microservice** foi implementado com sucesso:

  ```
  curl $GATEWAY_URL/customer


  customer => Error: 503 - preference => UnknownHostException: recommendation

  kubectl -n tutorial logs -f preference-v1-bb965dfd8-w4n7t preference
  Caused by: java.net.UnknownHostException: recommendation
  ```

### 3 - Deploy Recommendation Microservice <a name="workshop-deploy-recommendation-msa">

* Deployment do **Recommendation Microservice**

  ```
  kubectl apply -f <(istioctl kube-inject -f $TUTORIAL_HOME/recommendation/kubernetes/Deployment.yml) -n tutorial

  serviceaccount/recommendation created
  deployment.apps/recommendation-v1 created

  kubectl create -f $TUTORIAL_HOME/recommendation/kubernetes/Service.yml -n tutorial
  service/recommendation created
  ```

* Verifique se **Recommendation Microservice** foi implementado com sucesso:

  ```
  curl $GATEWAY_URL/customer

  customer => preference => recommendation v1 from 'recommendation-v1-55675885b5-l65cx': 1
  ```

### 4 - Observability - Grafana <a name="workshop-observability-grafana">

* Realize alguns *requests* na API: `sh $TUTORIAL_HOME/scripts/run.sh $GATEWAY_URL/customer`

* Acesse o serviço do **Grafana**: `istioctl dashboard grafana`

* Visualize os dashboards disponíveis em: `Home -> Istio`

### 5 - Observability - Prometheus <a name="workshop-observability-prometheus">

* Acesse o serviço do **Prometheus**: `istioctl dashboard prometheus`

* Navegue por métricas do **Istio**

* Crie métricas customizadas:

  ```
  kubectl create -f $TUTORIAL_HOME/istiofiles/recommendation_requestcount.yml -n istio-system

  instance.config.istio.io/recommendationrequestcount created
  handler.config.istio.io/recommendationrequestcounthandler created
  rule.config.istio.io/recommendationrequestcountprom created

  while true; do curl $GATEWAY_URL/customer; sleep .5;  done
  ```

* Coloque a seguinte métrica na interface do **Prometheus**:

 ```
 istio_requests_total{destination_service="recommendation.tutorial.svc.cluster.local"}
 container_memory_rss{namespace="tutorial",container=~"customer|preference|recommendation"}
 ```

### 6 - Observability - Jaeger <a name="workshop-observability-jaeger">

* Acesse o serviço do **Jaeger**: `istioctl dashboard jaeger`

### 7 - Observability - Kiali <a name="workshop-observability-kiali">

* Acesse o serviço do **Kiali**: `istioctl dashboard kiali`

  * as credenciais são: admin / admin

### 8 -  Traffic Control <a name="workshop-trafficcontrol-simple">

* Deployment do **Recommendation-v2 Microservice**: `kubectl apply -f <(istioctl kube-inject -f $TUTORIAL_HOME/recommendation/kubernetes/Deployment-v2.yml) -n tutorial`

* Abra uma novo terminal e execute o seguinte *script* que faz requests na **API:** `while true; do curl $GATEWAY_URL/customer; sleep 1;  done`

* Escale alguma das versões para um número de réplicas maior: `kubectl scale --replicas=2 deployment/recommendation-v2 -n tutorial`

* Retome as configurações iniciais: `kubectl scale --replicas=1 deployment/recommendation-v2 -n tutorial`

* Todos as requisições para **Recommendation-v2 Microservice**:

  ```
  kubectl create -f $TUTORIAL_HOME/istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
  kubectl create -f $TUTORIAL_HOME/istiofiles/virtual-service-recommendation-v2.yml -n tutorial
  ```

* Todos as requisições para **Recommendation-v1 Microservice**:

  ```
  kubectl replace -f $TUTORIAL_HOME/istiofiles/virtual-service-recommendation-v1.yml -n tutorial
  kubectl get virtualservice -n tutorial
  kubectl get virtualservice -o yaml -n tutorial
  ```

* Todas as requisições no modo padrão: `kubectl delete -f $TUTORIAL_HOME/istiofiles/virtual-service-recommendation-v1.yml -n tutorial`

* 90%/10%: `kubectl create -f $TUTORIAL_HOME/istiofiles/virtual-service-recommendation-v1_and_v2.yml -n tutorial`

* 75%/25%: `kubectl replace -f $TUTORIAL_HOME/istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial`

* CleanUp: `sh $TUTORIAL_HOME/scripts/clean.sh tutorial`

* Usuários Safari para **Recommendation-v2 Microservice**:

  ```
  -- todos para v1
  kubectl create -f $TUTORIAL_HOME/istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
  kubectl create -f $TUTORIAL_HOME/istiofiles/virtual-service-recommendation-v1.yml -n tutorial

  -- criando regra para Safari
  kubectl replace -f $TUTORIAL_HOME/istiofiles/virtual-service-safari-recommendation-v2.yml -n tutorial
  kubectl get virtualservice -n tutorial
  -- removendo regra Safari
  kubectl delete -f $TUTORIAL_HOME/istiofiles/virtual-service-safari-recommendation-v2.yml -n tutorial

  -- mobile para v2
  kubectl create -f $TUTORIAL_HOME/istiofiles/virtual-service-mobile-recommendation-v2.yml -n tutorial
  curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" $GATEWAY_URL/customer
  ```

* CleanUp: `sh $TUTORIAL_HOME/scripts/clean.sh tutorial`

### 9 -  Service Resiliency - Retry <a name="workshop-serviceresiliency-retry">

* Faça todas requisições para **Recommendation-v1 Microservice** falharem:

  ```
  kubectl exec -it -n tutorial $(kubectl get pods -n tutorial|grep recommendation-v2|awk '{ print $1 }'|head -1) -c recommendation /bin/bash
  curl localhost:8080/misbehave
  curl localhost:8080/behave
  ```

### 10 -  Service Resiliency - Timeout <a name="workshop-serviceresiliency-timeout">

  ```
  kubectl patch deployment recommendation-v2 -p '{"spec":{"template":{"spec":{"containers":[{"name":"recommendation", "image":"quay.io/rhdevelopers/istio-tutorial-recommendation:v2-timeout"}]}}}}' -n tutorial
  kubectl create -f $TUTORIAL_HOME/istiofiles/virtual-service-recommendation-timeout.yml -n tutorial

  kubectl patch deployment recommendation-v2 -p '{"spec":{"template":{"spec":{"containers":[{"name":"recommendation", "image":"quay.io/rhdevelopers/istio-tutorial-recommendation:v2"}]}}}}' -n tutorial
  ```

* CleanUp: `sh $TUTORIAL_HOME/scripts/clean.sh tutorial`
