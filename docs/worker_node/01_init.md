# Agregar un nuevo Worker node

## Prerrequisitos

- **Sincronización de tiempo:** Un mecanismo activo como `chronyd` o `ntpd`.
- **Sistema Operativo:** Instalación limpia de **Ubuntu 20.04** (LTS) con acceso a internet.
- **Drivers:** Controladores de **NVIDIA** y **CUDA** correctamente instalados y verificados.

!!! note "Notas de administración"
    - Se asume que todos los comandos son ejecutados por el usuario `root`.
    - El término **"admin"** se refiere exclusivamente al administrador del nodo maestro (*Control Plane*).

Deshabilitar `swap`

```bash
swapoff -a 
```

Este comando solo lo hace temporal. Para deshabilitar `swap` permanentemente en el nodo, comente la línea que diga `swap` en el archivo `/etc/fstab`.

---


## 1. Instalar y configurar el Runtime (containerd)

Basodo en la documentación de [kubernetes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd).

### 1.1. Configuración del sistema

Primero, cargamos los módulos del kernel necesarios para el enrutamiento de la red

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay

modprobe br_netfilter
```

Configuramos los parámetros de `sysctl`

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

### 1.2. Instalación del runtime

Basado en la documentación de [containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md).

#### 1.2.1. Instalación de CNI Plugins

Puede encontrar la última versión en el [repositorio oficial de CNI](https://github.com/containernetworking/plugins/releases).

Descargamos la versión de plugin. En este tutorial es uso la versión **v1.3.0**

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
```

Extraemos en el directorio `/opt/cni/bin`
```bash
mkdir -p /opt/cni/bin

tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
```

### 1.2.2. Instalación de containerd y runc

```bash
apt-get update
apt-get install -y containerd runc
```

### 1.2.3. Instalación y configuración de nvidia-container-toolkit

Basado en la [guía de instalación de NVIDIA](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).

Instalamos prerequisitos

```bash
apt-get update && sudo apt-get install -y --no-install-recommends \
   ca-certificates \
   curl \
   gnupg2
```

Configuramos el repositorio de NVIDIA
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt-get update
```

Instalamos los paquetes de NVIDIA
```bash
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.19.0-1
  apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

### 1.2.4. Configuración de containerd para NVIDIA

Configuramos runtime para que Kubernetes orqueste cargas de trabajo con GPU

```bash
nvidia-ctk runtime configure --runtime=containerd

sed -i 's/default_runtime_name = "runc"/default_runtime_name = "nvidia"/' /etc/containerd/config.d/99-nvidia.toml

systemctl restart containerd
```

## 2. Instalación de herramientas de Kubernetes

Basado en la documentación [Installing Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

### 2.1. Configuración del repositorio

Instalamos prerequisitos

```bash
apt-get update
apt-get install -y apt-transport-https ca-certificates curl # Últimos dos paquetes ya los instalamos en el paso anterior
```

Agregamos repositorio con sus llaves GPG

```bash
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.29/deb/](https://pkgs.k8s.io/core:/stable:/v1.29/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### 2.2. Instalación de paquetes (kubelet, kubeadm y kubectl)

Instalamos las herramientas y habilitamos el servicio `kubelet`

```bash
apt-get update
apt-get install -y kubelet kubeadm kubectl

systemctl enable --now kubelet
```

## 3. Instalación y configuración de la VPN

Basado en la documentación oficial de instalación de [wireguard](https://www.wireguard.com/install/).

Antes de empezar, contacte con el administrador del **control plane** para recibir una nueva dirección IP dentro de la VPN y los datos de servidor VPN.

!!! warning "Advertencia"
    Si se encuentra conectado dentro de una institución es posible que el equipo de seguridad informática tenga bloqueadas las conexiones. Si es su caso, favor de contactarlos y pedirles que habiliten las conexiones (IP y puertos que le administrador de PIG dicte). De lo contrario, no se podrá realizar la conexión con el **control plane**.


### 3.1. Instalación de wireguard

```bash
apt install wireguard
```

## 3.2. Crear clave privada del nodo

```bash
wg genkey
```

## 3.3. Configuración de wireguard

Creamos el archivo `/etc/wireguard/wg0.conf` con el siguiente contenido
```bash
[Interface]
Address = <Dirección IP asignada por el admin>/24
PrivateKey = <Clave generada en el paso anterior>
ListenPort = 21841

[Peer]
PublicKey = <Clave pública dada por el admin>
Endpoint = <IP:puerto dados por el admin>
AllowedIPs = <Segmento de red dado por el admin>/24
PersistentKeepalive = 25
```

## 3.4. Levantamiento de la interfaz virtual de la VPN

```bash
wg-quick up wg0

systemctl enable wg-quick@wg0
```

!!! note "Nota"
    Asegúrese de siempre levantar la interfaz si el nodo es reiniciado.

## 3.5. Obtener clave pública

```bash
wg show
```

Comparte con el admin la clave pública.

## 4. Conexión al clúster

El administrador de clúster  deberá generar el token / hash para integrar el nodo y enviarselos. Pida al admin los siguientes parámetros:

- IP:PUERTO (Del nodo maestro)
- TOKEN
- HASH

## 4.1. Incorporación del nodo al clúster 

Ejecutamos el comando `kubeadm join` con los parámetros anteriores dados por el admin

```bash
kubeadm join <IP:PUERTO> --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

## 4.2. Configuración de `kubeadm` para usar la interfaz virtual para conectarse al clúster

Editamos el arachivo `var/lib/kubelet/kubeadm-flags.env` y gregamos la dandera `--node-ip` seguida de la dirección IP asignada por el admin del *control plane*.

!!! Example "Ejemplo"
    ```bash
    KUBELET_KUBEADM_ARGS="--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
    --pod-infra-container-image=registry.k8s.io/pause:3.8 \
    --node-ip=<Segmento de red dado por el admin>”
    ```

## 4.3. Reiniciar `kubelet`

```bash
systemctl restart kubelet
```

## 5. Instalación de s3fs

```bash
apt-get install s3fs
```

!!! success "Éxito"
    El administrador del clúster le dirá si el nodo se agregó correctamente. De igual forma, si el nodo aparece al executar `kubectl get nodes` entonces se agregó con éxito.