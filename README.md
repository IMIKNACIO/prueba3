# prueba3
Orquestacion de Kubernetes
FASE 1: Preparación Común de Nodos (Realizar en AMBAS máquinas)

1-Actualizacion de maquina

sudo apt update && sudo apt upgrade -y
sudo reboot

2-Instalar kubelet, kubeadm y kubectl + bloquear versiones

sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

descargar clave firma publica

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg


Agrega repositorio Kubernetes correspondiente

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


Actualiza índice de paquetes

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


habilitar servicio kubelet

sudo systemctl enable --now kubelet


3-Verificacion de instalación

kubelet --version
kubeadm version
kubectl version –client



4-Desactivar swap permanentemente


sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

verificar swap (debe estar en 0B)

free -h



5-Habilitar módulos del kernel y configurar sysctl

Cargar modulo kernel

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

Configurar sysctl

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl -–system



Verificaciones

lsmod | grep br_netfilter
lsmod | grep overlay

Verifica sysctl:

sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables


FASE 2: Instalación del Entorno de Ejecución (Realizar en AMBAS máquinas)

1.Instale las dependencias necesarias y agregue el repositorio oficial de Docker.

sudo apt-get update && sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release


Crear el directorio para llaves GPG (si no existe):

sudo mkdir -p /etc/apt/keyrings


Agregar la clave GPG oficial de Docker:

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg


Agregar el repositorio de Docker (estable):

echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


2.Instale el paquete containerd.io : 

sudo apt-get update
sudo apt-get install -y containerd.io


3.Genere el fichero de configuración por defecto para containerd en /etc/containerd/config.toml.
Crear el archivo de configuración por defecto:

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
Editar para activar SystemdCgroup:
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml


4 Reinicie y habilite el servicio containerd para que se inicie con el sistema. Verifique su estado.

sudo systemctl restart containerd
sudo systemctl enable containerd

systemctl status containerd



FASE 3: Inicialización del Nodo Maestro (Realizar SÓLO en el Master)

1.	Descargue las imágenes de contenedor requeridas por el Control Plane:

sudo kubeadm config images pull

2.	Configure el fichero /etc/hosts para que el hostname del nodo maestro :

sudo hostnamectl set-hostname k8scp


3.	resuelva a su propia IP estática. Utilice k8scp como hostname.
VIM /etc/hosts

agrega linea: Ipestatica k8scp


4.	Inicialice el cluster con kubeadm init utilizando los siguientes parámetros:
o	El Pod Network CIDR debe ser 172.24.0.0/16.
o	El socket del CRI debe ser el de containerd.
o	El endpoint del Control Plane debe ser k8scp.


sudo kubeadm init \
--pod-network-cidr=172.24.0.0/16 \
--cri-socket unix:///run/containerd/containerd.sock \
--control-plane-endpoint=k8scp



(si falla)
Ejecuta:
1-	sudo sysctl -w net.ipv4.ip_forward=1
2-	echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf 
3-	sudo sysctl -p
4- sudo kubeadm init \
--pod-network-cidr=172.24.0.0/16 \
--cri-socket unix:///run/containerd/containerd.sock \
--control-plane-endpoint=k8scp



4.Guarde el comando kubeadm join completo.

ejemplo : kubeadm join k8scp:6443 --token geqpul.yt1reo2bk52m1s27 \
        --discovery-token-ca-cert-hash sha256:4e88d66127187ef9f61299a97992e90c993ab772a3973884d8e55f89e963257a

se usara para hacer el join mas adelante



5.	Configure el acceso a kubectl para su usuario regular:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


6-Valide el estado del nodo. Se espera que esté NotReady:

kubectl get nodes




Fase 4: Instalación del CNI Calico (Realizar SÓLO en el Master)

1.Descargue los manifiestos tigera-operator.yaml y custom-resources.yaml de una versión estable de Calico:

curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

2.Aplique el manifiesto del operador
3.Modifique el fichero custom-resources.yaml para que el cidr del ipPools coincida con el utilizado en kubeadm init (172.24.0.0/16):

sudo vim calico.yaml
# name: CALICO_IPV4POOL_CIDR
#              value: "172.24.0.0/16"


4.Aplique el manifiesto de los recursos personalizados:

kubectl apply -f calico.yaml

5.Monitoree los pods del namespace calico-system hasta que estén todos en estado Running:

kubectl get pods -n kube-system


6.Verifique que el estado del nodo maestro ha cambiado a Ready: 

kubectl get nodes


(en caso de fallar )
free -h   ( verificar que swap este en 0B)
 en caso de presentar algo distinto ejecutar:
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab


ejecutar los siguientes módulos en master y worker

# Cargar módulos persistentes
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configuración sysctl persistente
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl –-system


***luego verifica***

lsmod | grep br_netfilter
lsmod | grep overlay

sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables


***verifica que estén corriendo calico (running) y k8scp (ready)****

kubectl get pods -n kube-system
kubectl get nodes



Fase 5: Unión del Nodo Worker (Realizar en Master y Worker)

1.En el Nodo Worker: Modifique el fichero /etc/hosts para que resuelva el hostname k8scp a la IP del nodo maestro:

Vim /etc/hosts
agregar linea 172.31.x.x k8scp

2.En el Nodo Worker: Ejecute el comando kubeadm join que guardó previamente.     AQUI EJECUTAMOS EL COMANDO GUARDADO DE LA FASE 4 

kubeadm join k8scp:6443 --token geqpul.yt1reo2bk52m1s27 \
        --discovery-token-ca-cert-hash sha256:4e88d66127187ef9f61299a97992e90c993ab772a3973884d8e55f89e963257a



3.En el Nodo Master: Verifique con kubectl get nodes que ambos nodos están en el cluster y en estado Ready:


kubectl get nodes


AMBOS NODOS DEBEN ESTAR READY EN EL CLUSTER
