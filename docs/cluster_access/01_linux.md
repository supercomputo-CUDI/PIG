# Usando linux

## Instalación y configuración

Empezamos clonando el repositorio de *github* y entramos al directorio descargado

```bash
git clone https://github.com/CUDI-PIG/PIG.git
cd PIG-Resources
```

Ahora instalamos lo necesario corriendo el archivo `linux-setup.sh`

```bash
chmod +x linux-setup.sh
./linux-setup.sh
```
Con el primer comando lo hacemos ejecutable. Después, configuramos *kubernetes* ejecutando el archivo `k8s-setup.sh`

```bash
./k8s-setup.sh
```

Se nos pedirá una llave que nos dará el administrador del clúster, como se ve en la siguiente imagen

![Set secret key](../assets/images/pig_access/secret_key.png){ style="display: block; margin: 0 auto; width: 1000px;"}

!!! info "Importante"
    Para obtener la llave, favor de contactar al administrador del sistema de PIG.

Por último, agregamos la siguiente ruta de la herramienta de línea de comandos para *kubernetes*, llamada *krew*, al archivo `~/.bashrc` (configura nuestra *shell* de bash) 

```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

Al agregar el `PATH` recargamos la *shell*

```bash
source ~/.bashrc
```

Para verificar que la instalación y configuración fue exitosa, usaremos el siguiente comando

```bash
kubectl get pods
```

Al ejecutar el comando se abrirá una página nueva en su navegador predeterminado como la siguiente

![Keycloak connection](../assets/images/pig_access/keycloak.png){ style="display: block; margin: 0 auto; width: 1000px;"}

Donde deberá ingresar las credenciales de su cuenta en PIG proporcionadas por el administrador. Si al momento de correr un comando de kubernetes no abre la página de keycloak, como se ve en la image, entonces puede agregar la bandera `--skip-open-browser` al archivo `k8s-setup.sh` para que le imprima la URI donde se redirecciona la página de keycloak. Quedaría el comando de `kubectl` de la siguiente manera

```bash

kubectl config set-credentials oidc --exec-command=kubectl \
    --exec-api-version=client.authentication.k8s.io/v1beta1 \
    --exec-arg="oidc-login" \
    --exec-arg="get-token" \
    --exec-arg="--oidc-issuer-url=https://sso.lamod.unam.mx/auth/realms/cudi" \
    --exec-arg="--oidc-client-id=k8s" \
    --exec-arg="--oidc-client-secret=$client_secret" \
    --exec-arg="--skip-open-broswer" \
    --kubeconfig=$KUBECONFIG
```

Debe correr el archivo de nuevo para que se apliquen los cambios.

Por predeterminado, se redirrecciona al `localhost:8000` o `localhost:18000`. Si tiene ocupados esos puertos puede , en lugar de agregar la bandera `--skip-open-browser`, agregar la bandera `--listen-adrress=127.0.0.1:puerto_deseado`.

Si la conexión fue exitosa, en la terminal obtendrá el resultado del comando de *kubernetes*

![Success connection](../assets/images/pig_access/success_con.png){ style="display: block; margin: 0 auto; width: 1000px;"}

Este comando nos muestra los pods actuales en PIG.

!!! Success "Éxito"
    Si obtiene un resultado similar al de la imagen ¡¡Felicidades ya puede usar el clúster de PIG!!




