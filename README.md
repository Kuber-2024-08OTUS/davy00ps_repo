### Run minikube ###
minikube start

### Crate namespace ###
k create -f namespace.yaml
k config set-context --current --namespace=homework

### Create sa/deployments/service/ingress ###
k create -f monitoring.yaml
k create -f deployment.yaml && k create -f service.yaml

# Check access to endpoint "/metrics"
k exec -it pod/[tab] -- bash
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
-H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
https://kubernetes.default.svc/metrics | wc -l 

# Create token with duration 24 hours and put into "token" file
k create sa cd
k create token cd --duration 24h > token

# Create kubeconfig
export SA_SECRET_TOKEN=$(cat token)
export CLUSTER_NAME=$(kubectl config current-context)
export CLUSTER_CA_CERT=$(k config view -o jsonpath='{.clusters[0].cluster.certificate-authority}')
export CLUSTER_ENDPOINT=$(k config view -o jsonpath='{.clusters[0].cluster.server}')

cat << EOF > cd-config
apiVersion: v1
clusters:
- cluster:
    certificate-authority: ${CLUSTER_CA_CERT}
    server: ${CLUSTER_ENDPOINT}
  name: ${CLUSTER_NAME}
contexts:
- context:
    cluster: ${CLUSTER_NAME}
    namespace: homework
    user: cd
  name: ${CLUSTER_NAME}
current-context: ${CLUSTER_NAME}
kind: Config
users:
- name: cd
  user:
    token: ${SA_SECRET_TOKEN}
EOF

# Check access without giving permissions and after
k get all --kubeconfig=cd-config
k create -f cd.yaml
k create cd.yaml

### Create ingress ###
minikube addons enable ingress
k create -f ingress.yaml

curl --resolve "homework.otus:80:$(minikube ip)" \
-i http://homework.otus/metrics | wc -l
