###deploy the production deployments
kubectl apply -f example-myapp-production/deployment.yaml

###deploy the production deployments svc
kubectl apply -f example-myapp-production/svc.yaml

###create nginx for testing
kubectl run nginx --image=nginx

###curl the myapp-production service from nginx prod
kubectl exec -it nginx bash
curl myapp-production-service.default.svc.cluster.local:80

###helm 
https://helm.sh/docs/intro/install/


###install monitoring components using helm

kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

### debugging 
kubectl patch ds prometheus-prometheus-node-exporter --type "json" -p '[{"op": "remove", "path" : "/spec/template/spec/containers/0/volumeMounts/2/mountPropagation"}]'

### expose svc
kubectl expose service prometheus-grafana --type=NodePort --name=grafana-lb -n monitoring
edit the svc once created to use port 3000
OR
kubectl expose service prometheus-grafana --type=NodePort --name=grafana-lb --port=3000 --target-port=3000 -n monitoring
kubectl expose service prometheus-kube-prometheus-prometheus --type=NodePort --name=prometheus-lb  -n monitoring

curl localhost on the above NodePort
Username: admin
Password: prom-operator

### configure service monitoring ### for this to work the app image needs to be updated 
kubectl apply -f example-myapp-production

### customise promethus to only see your smon / endpoints, update this in the promethus object :
  serviceMonitorNamespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: default
  serviceMonitorSelector: {}

### Before running HPA, metrics server should be installed if not present by default in your cluster
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  # if READY value is 0/1 for deployment.apps/metrics-server after running  below command 
  # kubectl get deploy -n kube-system
  # then do following as per https://dev.to/docker/enable-kubernetes-metrics-server-on-docker-desktop-5434
    # wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    # add below arg at Deployment->spec->template->spec->args
    # --kubelet-insecure-tls
    
### configure HPA 
kubectl autoscale deployment myapp-production-deployment --cpu-percent=50 --min=1 --max=5

### generate load 
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "count=0; while sleep 0.01; count=$((count+1)); do wget -q -O- http://myapp-production-service.default.svc.cluster.local:80; echo "\n"; done"



