

# Install y configurar aws cli

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install

aws --version
aws configure
```

# Install kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client
```

para conectarse a un cluster
aws eks update-kubeconfig --region region-code --name my-cluster

# Install eksctl

https://eksctl.io/installation/

```bash
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin


eksctl version
```


## install helm

```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh 

chmod 700 get_helm.sh 

./get_helm.sh
```



# Crear Cluster



```bash
eksctl create cluster --name my-cluster --region us-east-1 \
--nodegroup-name my-nodes --node-type t3.medium --nodes 1 \
--version 1.29

eksctl utils write-kubeconfig --cluster my-cluster
```



```bash
#BORRAR cluster
# eksctl delete cluster --name my-cluster --region us-east-1
```



# Install Argos CLI

```shell
# sudo passwd ubuntu  #para multippass
#install Brew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install argocd
```

Ver documentacion en https://argoproj.github.io/argo-cd/cli_installation/#linux-and-wsl

Ver documentacion de instalar brew https://www.codegrepper.com/code-examples/shell/install+brew+ubuntu+20.04



# Instalar ingress en Cluster

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx


helm upgrade --install ingress-nginx ingress-nginx \
             --repo https://kubernetes.github.io/ingress-nginx \
             --namespace ingress-nginx \
             --create-namespace
```





#### Crear namaespace dev

```shell
kubectl create ns dev
```



# CAMBIAR a otro equipo

aws configure

eksctl utils write-kubeconfig --cluster my-cluster




#### configurar los secretos para certificado https 

```shell
#para generar de manera temporal un key 
mkcert '*.edisaac.link'
openssl x509 -outform pem -in _wildcard.edisaac.link.pem -out _wildcard.edisaac.link.crt

cp _wildcard.edisaac.link.crt /usr/local/share/ca-certificates/_wildcard.edisaac.link.crt
sudo update-ca-certificates


kubectl create secret tls edisaac-tls-secret -n dev \
  --cert=<<ruta local del archivo crt>> \
  --key=<<ruta local del archivo key>> 
 
```

ejemplo

```shell
kubectl create secret tls edisaac-tls-secret -n dev  \
  --cert=_wildcard.edisaac.link.crt \
  --key=_wildcard.edisaac.link-key.pem

```

 


### Install Argos

```shell
#kubectl config get-contexts
#kubectl config use-context <<DEV CONTEXT NAME>>

kubectl create namespace argocd

#instalar version de ARGOCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
#CONFIGURAR EN MODO INSECURE
kubectl apply -n argocd -f https://raw.githubusercontent.com/edisaac/argo-cd/master/manifests/install.yaml


kubectl create secret tls edisaac-tls-secret -n argocd  \
  --cert=_wildcard.edisaac.link.crt \
  --key=_wildcard.edisaac.link-key.pem 
   
```

Ver documentacion en https://argoproj.github.io/argo-cd/getting_started/#1-install-argo-cd



# CAMBIAR A KUBE LOCAL




### Abrir puerto local a argocd

```shell
kubectl port-forward svc/argocd-server -n argocd 8888:80
```

### Ingresar a argo con usuario admin



para  obtener la contraseña inicial

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
```shell
argocd login 127.0.0.1:8888
```

para cambiar de contraseña ver  

https://github.com/argoproj/argo-cd/blob/master/docs/faq.md#i-forgot-the-admin-password-how-do-i-reset-it 
### Registrar repositorio usando ssh keys

##### Configuracion de gitlab

Generar los keys seguir el manual de gitlab https://docs.gitlab.com/ee/ssh/#generate-an-ssh-key-pair

Registrarlo en gitlab con el usuario que tenga permisos sobre el respositorio de developers.edisaac/devops/kubernetes-configuration   https://docs.gitlab.com/ee/ssh/#generate-an-ssh-key-pair




ssh-keygen -t ed25519 -C "edisaac.mejia@gmail.com" -f ~/.ssh/argocd


##### Configurar repo en kubernetes-configuration con shh key

```shell
argocd repo add git@github.com:edisaac/kubernetes-configuration.git --name  kubernetes-configuration --ssh-private-key-path ~/.ssh/argocd
```


##### Configurar repo en argocd-configuration con shh key

```shell
argocd repo add git@github.com:edisaac/argocd-configuration.git --name  argocd-configuration --ssh-private-key-path  ~/.ssh/argocd
```

 

### Listar los nombres de los cluster

```shell
kubectl config get-contexts
```

### Agregar el cluster NAME de DEV

```shell
argocd cluster add <<CLUSTER NAME DEV>> --name dev-cluster
```

### Aplicar el Application Argos Inicial

```shell
kubectl apply -f ./dev/inicial.yaml
```



Verificar en 

http://localhost:8888/

una vez termine de cargar ingresar por 

https://argocddev.edisaac.link/

 

