# Agregar un nuevo Worker node con AlmaLinux

## Prerrequisitos

- **Sincronización de tiempo:** Un mecanismo activo como `chronyd` o `ntpd`.
- **Sistema Operativo:** Instalación limpia de **AlmaLinux 9.6** con acceso a internet.
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
dnf install -y runc

dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y containerd.io containerd config default | sudo tee /etc/containerd/config.toml
```

En el archivo `/etc/containerd/config.toml` activamos `SystemdCgroup`

```bash
SystemdCgroup = true
```

Habilitamos el servicio

```bash
systemctl enable --now containerd
```

### 1.2.3. Instalación y configuración de nvidia-container-toolkit


Instalamos prerequisitos

```bash
dnf install libseccomp
dnf install libseccomp-devel
dnf install -y apt-transport-https ca-certificates curl gpg socat conntrack
```

Configuramos el repositorio de NVIDIA
```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
```

Instalamos los paquetes de NVIDIA

```bash
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.18.0-1
dnf install -y \
   nvidia-container-toolkit-${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
   nvidia-container-toolkit-base-${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
   libnvidia-container-tools-${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
   libnvidia-container1-${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

### 1.2.4. Configuración de containerd para NVIDIA

Configuramos runtime para que Kubernetes orqueste cargas de trabajo con GPU

```bash
nvidia-ctk runtime configure --runtime=containerd

sed -i 's/default_runtime_name = "runc"/default_runtime_name = "nvidia"/' /etc/containerd/config.d/99-nvidia.toml

systemctl restart containerd
```

## 2. Instalación de herramientas de Kubernetes


### 2.1. Descargamos `crictl`


```bash
export DOWNLOAD_DIR="/usr/local/bin"
mkdir -p "$DOWNLOAD_DIR"
CRICTL_VERSION="v1.29.0"
export ARCH="amd64"

curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz

chown root.root /usr/local/bin/crictl
```


### 2.2. Decargamos `kubeadm` y `kubelet`

```bash
export RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"

cd $DOWNLOAD_DIR
curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}

chmod +x {kubeadm,kubelet}
export RELEASE_VERSION="v0.16.2"

curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
mkdir -p /etc/systemd/system/kubelet.service.d

curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl enable --now kubelet
```

## 3. Instalación y configuración de la VPN

Basado en la documentación oficial de instalación de [wireguard](https://www.wireguard.com/install/).

Antes de empezar, contacte con el administrador del **control plane** para recibir una nueva dirección IP dentro de la VPN y los datos de servidor VPN.

!!! warning "Advertencia"
    Si se encuentra conectado dentro de una institución es posible que el equipo de seguridad informática tenga bloqueadas las conexiones. Si es su caso, favor de contactarlos y pedirles que habiliten las conexiones (IP y puertos que le administrador de PIG dicte). De lo contrario, no se podrá realizar la conexión con el **control plane**.


### 3.1. Instalación de wireguard

```bash
dnf install wiregard-tools -y
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
dnf install s3fs-fuse -y
```

!!! Warning "Advertencia"
    Es posible que se requiera el siguiente paso: `ln -s /etc/resolv.conf /run/systemd/resolve/resolv.conf`


!!! success "Éxito"
    El administrador del clúster le dirá si el nodo se agregó correctamente. De igual forma, si el nodo aparece al executar `kubectl get nodes` entonces se agregó con éxito.