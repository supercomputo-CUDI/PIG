# Mi primer pod en PIG

creamos un archivo `00-simple-gpu.yaml` que defina la configuración de nuestro pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-cuda
spec:
  restartPolicy: Never
  containers:
    - name: demo-cuda
      image: "nvidia/cuda:11.0.3-devel-ubi8"
      command: ["sh", "-c", "sleep infinity"]
      resources:
        limits:
          memory: 4G
          cpu: 2
          nvidia.com/gpu: 1
        requests:
          memory: 2G
          cpu: 1
          nvidia.com/gpu: 1
```

Para desplejar el pod en el clúster ejecutamos

```bash
kubectl create -f 00-simple-gpu.yaml
```

Este comando le dice a *Kubernetes* que cree los recursos definidos en el archivo *YAML*. Podemos monitorear el estado del pod con el siguiente comando

```bash
kubectl get pods
```

Una vez que el pod está en estado `Running`, podemos interactuar con él de dos formas:

- Mandando a ejecutar un comando

Por ejemplo

```bash
kubectl exec -it demo-cuda -nvidia-smi
```

Usamos el nombre `demo-cuda`, ya que es el que especificamos en el archivo *YAML*. Este comando ejecuta `nvidia-smi` dentro del pod para verificar el estado y la información de la GPU.

- Accediendo al pod de forma interactiva

```bash
kubectl exec -it -bash
```

Este comando nos da acceso a una *shell* interactiva dentro del contenedor, permitiéndonos ejecutar comandos directamente. Para salir del *shell*, use `exit` o presione ++ctrl+d++.

Cuando haya terminado de usar el pod, puede eliminarlo

```bash
kubectl delete pod demo-cuda
```

Este comando elimina el pod y libera todos los recursos asociados (CPU, memoria, GPU).

