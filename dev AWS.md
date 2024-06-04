

install aws cli



install kubectl

install eksctl




aws ec2 create-key-pair --key-name eksKeyPair --query 'KeyMaterial' --output text > eksKeyPair.pem




eksctl create cluster --name my-cluster --region us-east-1 \
--nodegroup-name my-nodes --node-type t3.medium --nodes 1 \
--with-oidc \
--ssh-access \
--ssh-public-key eksKeyPair
 



eksctl utils write-kubeconfig --cluster my-cluster

# BORRAR eksctl delete cluster --name my-cluster --region us-east-1




## install helm

```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh 

chmod 700 get_helm.sh 

./get_helm.sh
```



helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx


helm upgrade --install ingress-nginx ingress-nginx \
             --repo https://kubernetes.github.io/ingress-nginx \
             --namespace ingress-nginx \
             --create-namespace


#### Crear namaespace dev

```shell
kubectl create ns dev
```




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

kubectl apply -n argocd -f https://raw.githubusercontent.com/edisaac/argo-cd/master/manifests/install.yaml


kubectl create secret tls edisaac-tls-secret -n argocd  \
  --cert=_wildcard.edisaac.link.crt \
  --key=_wildcard.edisaac.link-key.pem 
   
```

Ver documentacion en https://argoproj.github.io/argo-cd/getting_started/#1-install-argo-cd

### Install Argos CLI

```shell
#install Brew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install argocd
```

Ver documentacion en https://argoproj.github.io/argo-cd/cli_installation/#linux-and-wsl

Ver documentacion de instalar brew https://www.codegrepper.com/code-examples/shell/install+brew+ubuntu+20.04

### Modificar la imagen para que no use TLS interno

Agregar opcion insecure al deploy de argocd-server

```shell

kubectl apply -n argocd -f https://raw.githubusercontent.com/edisaac/argo-cd/master/manifests/install.yaml
 
```

Ejemplo de como quitar la linea insecure


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




#

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
