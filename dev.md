Instalar gcloud https://cloud.google.com/sdk/docs/install?hl=es-419#deb


sudo apt-get install kubectl

sudo apt-get install google-cloud-cli-gke-gcloud-auth-plugin

# kubernetes-configuration

## Ingresar y Listar Proyectos proyecto Gcloud

```shell
gcloud auth login
gcloud projects list
```


## CREAR CLUSTER DEV

#### Seleccionar el proyecto edisaac dev

```shell
gcloud config set project dockers-edisaac-dev 
```

#### Crear Cluster de dev-cluster 

```bash
gcloud container clusters create dev-cluster \
--num-nodes=1 \
--machine-type=e2-standard-4 --zone us-east1-b \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform" \
--cluster-version 1.28 \
--enable-ip-alias 
```

#### Conectar al proyecto edisaac dev

```shell
gcloud container clusters get-credentials dev-cluster --zone us-east1-b --project civic-ripsaw-424218-u2
```

#### Crear namaespace dev

```shell
kubectl create ns dev
```

#### Crear configuracion de docker registry

```bash
kubectl create secret docker-registry regcred --docker-server=registry.hub.docker.com --docker-username=edisaac.mejia@gmail.com --docker-password=$PASSWORD_DOCKER -n dev
```

### Configuracion Ingress Traefik DEV

##### crear y ver ip statica para ingress para DEV

```shell
gcloud compute addresses create dev-static-ip-ingress  --region=us-east1

gcloud compute addresses describe dev-static-ip-ingress  --region=us-east1 \
  --format='value(address)'
```

##### Modificar el loadBalancerIP en la configuracion de DEV

con el ip creado en el branch **SNAPSHOT** de **developers.edisaac/devops/kubernetes-configuration.git**

modificar el archivo **/dev/config/ingress/traefik.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  loadBalancerIP: <<dev-static-ip-ingress>>
```

#### configurar los secretos para certificado https 

```shell
#para generar de manera temporal un key 
mkcert '*.edisaac.link'
openssl x509 -outform pem -in _wildcard.edisaac.link.pem -out _wildcard.edisaac.link.crt


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

Crear un disco para el traefik

```shell
gcloud compute disks create --size=10GB --zone=us-east1-b traefik-persistent
```



## Instalar Argos en cluster prd

##### Conectar al proyecto edisaac PROD

```shell
gcloud container clusters get-credentials dev-cluster --zone us-east1-b --project dockers-edisaac-dev
```

### Install Argos

```shell
kubectl config get-contexts
kubectl config use-context <<DEV CONTEXT NAME>>
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/edisaac/argo-cd/master/manifests/install.yaml
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

#kubectl patch  deployment argocd-server -n argocd  --type json -p='[{"op": "add", "path": "/spec/template/spec/containers/0/command/1", "value": "--insecure"}]'
```

Ejemplo de como quitar la linea insecure

kubectl patch  deployment argocd-server -n argocd  --type json -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/command/1"}]'

### Abrir puerto local a argocd

```shell
kubectl port-forward svc/argocd-server -n argocd 8888:80
```

### Ingresar a argo con usuario admin

```shell
argocd login 127.0.0.1:8888
```

para  obtener la contraseña inicial

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
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

