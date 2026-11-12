# Montaje, puntos de acceso y modalidad de acceso

Una vez creado el sistema de archivos, con sus destinos de montaje disponibles y el grupo de seguridad correctamente configurado, queda **montarlo en los clientes**. Este apartado describe el montaje con el asistente de AWS, el montaje permanente, los **puntos de acceso** y las distintas formas de establecer la **modalidad de acceso** (solo lectura o lectura/escritura).

---

## Montaje con el asistente de EFS (recomendado)

AWS recomienda instalar el paquete **`amazon-efs-utils`**, que incluye el asistente de montaje. Simplifica el comando, resuelve el DNS, añade **cifrado en tránsito (TLS)** y **autenticación IAM**, y aplica automáticamente las opciones NFS recomendadas.

```bash
# Amazon Linux 2023
sudo dnf install -y amazon-efs-utils

sudo mkdir -p /mnt/efs

# Montaje básico
sudo mount -t efs fs-0abc123:/ /mnt/efs

# Con cifrado en tránsito
sudo mount -t efs -o tls fs-0abc123:/ /mnt/efs

# Con cifrado en tránsito e identidad IAM (necesario si la política evita el acceso anónimo)
sudo mount -t efs -o tls,iam fs-0abc123:/ /mnt/efs

df -hT /mnt/efs        # tipo nfs4, tamaño aparente de 8 EiB
```

!!! note "8 exabytes libres"
    `df` muestra un tamaño de **8 EiB** porque EFS es elástico y no tiene un límite aprovisionado. No es un error: el espacio realmente ocupado se consulta en la consola, con `describe-file-systems` o con la métrica `StorageBytes`.

En otras distribuciones (Ubuntu, Debian), `amazon-efs-utils` se compila o se instala desde el repositorio del proyecto en GitHub. La consola, en el botón **Asociar** del sistema de archivos, muestra los comandos exactos para cada método.

## Montaje permanente con `/etc/fstab`

Para que el montaje se mantenga tras un reinicio se añade una entrada a `/etc/fstab`:

```bash
# Asistente de EFS con TLS
fs-0abc123:/  /mnt/efs  efs  _netdev,noresvport,tls  0  0

# Asistente de EFS con TLS e IAM
fs-0abc123:/  /mnt/efs  efs  _netdev,noresvport,tls,iam  0  0

```

La opción **`_netdev`** indica que es un sistema de ficheros **de red**: el sistema espera a que la red esté disponible antes de montarlo. Sin ella, la instancia puede intentar montar EFS demasiado pronto y quedarse bloqueada en el arranque. Como con EBS, se comprueba con `sudo umount /mnt/efs && sudo mount -a` antes de reiniciar.

---

## Puntos de acceso (*access points*)

Un **punto de acceso** es un punto de entrada específico de aplicación al sistema de archivos. Permite que cada aplicación vea **solo su parte** del sistema de archivos y trabaje **con una identidad fija**, independientemente del usuario que use el cliente.

| Propiedad | Función |
| --- | --- |
| **Directorio raíz** (*root directory*) | El cliente que monta a través del punto de acceso ve ese directorio como `/`. No puede salir de él |
| **Permisos de creación** | Si el directorio raíz no existe, EFS lo **crea** con el propietario (UID/GID) y los permisos indicados |
| **Usuario POSIX** | EFS **sustituye** el UID/GID del cliente por los indicados en todas las operaciones |

Son especialmente útiles cuando:

- Varias aplicaciones comparten un sistema de archivos y deben quedar **aisladas** entre sí.
- Los clientes no tienen usuarios coherentes (contenedores de ECS/EKS, funciones **Lambda**, que exigen montar EFS a través de un punto de acceso).
- Se ha **impedido el acceso *root***: el punto de acceso crea el directorio con el propietario correcto sin necesidad de *root*.

=== "Consola AWS"

    1. **EFS → Puntos de acceso → Crear punto de acceso** (o pestaña **Puntos de acceso** del sistema de archivos).
    2. **Sistema de archivos**: el sistema `fs-0abc123`. **Nombre**: `ap-web`.
    3. **Ruta del directorio raíz**: `/web`.
    4. **Usuario POSIX**: ID de usuario `48`, ID de grupo `48`.
    5. **Permisos de creación del directorio raíz**: ID de usuario propietario `48`, ID de grupo propietario `48`, permisos `0755`.
    6. Pulsar **Crear punto de acceso**. El botón **Asociar** del punto de acceso muestra el comando de montaje exacto.

=== "AWS CLI"

    ```bash
    aws efs create-access-point --file-system-id fs-0abc123 \
      --posix-user Uid=48,Gid=48 \
      --root-directory 'Path=/web,CreationInfo={OwnerUid=48,OwnerGid=48,Permissions=755}' \
      --tags Key=Name,Value=ap-web

    aws efs describe-access-points --file-system-id fs-0abc123 \
      --query "AccessPoints[].[AccessPointId,RootDirectory.Path,LifeCycleState]" --output table
    ```

Montaje en la instancia a través del punto de acceso (requiere `tls`):

```bash
sudo mount -t efs -o tls,accesspoint=fsap-0123456789abcdef0 fs-0abc123:/ /var/www/html/uploads
```

En el ejemplo, el UID/GID 48 corresponde al usuario `apache` en Amazon Linux: todo lo que escriba cualquier proceso del cliente se guardará como `apache`, dentro de `/web`.

Para obligar a que un rol solo pueda acceder **a través** de un punto de acceso concreto, se usa la condición `elasticfilesystem:AccessPointArn` en la política del sistema de archivos.

---

## Modalidad de acceso: solo lectura o lectura/escritura

El criterio de evaluación pide montar el sistema de ficheros «estableciendo la modalidad (solo lectura o bien lectura/escritura)». En EFS esta modalidad puede imponerse en **varios niveles**, de menor a mayor robustez:

| Nivel | Cómo | ¿Quién lo controla? | Robustez |
| --- | --- | --- | --- |
| **Opción de montaje** | `mount -t efs -o tls,ro fs-0abc123:/ /mnt/efs` o `ro` en `/etc/fstab` | El administrador **de la instancia** | Baja: un *root* local puede volver a montar en `rw` |
| **Permisos POSIX** | `chmod 555`, propietario distinto del usuario de la aplicación | El propietario de los ficheros | Media |
| **Punto de acceso** | Usuario POSIX sin permisos de escritura sobre el directorio raíz | El administrador de EFS | Media-alta |
| **Política de IAM / del sistema de archivos** | Conceder solo `ClientMount`, no `ClientWrite`, o un `Deny` explícito de `ClientWrite` si el rol ya lo concede | El administrador **de AWS** | **Alta**: se impone en el servidor, sea cual sea la configuración del cliente |

Un diseño habitual combina varios: por ejemplo, los servidores web montan los contenidos en **lectura/escritura** con un rol que tiene `ClientMount` y `ClientWrite`, mientras que un servidor de informes los monta en **solo lectura** con un rol que solo tiene `ClientMount` y, además, con la opción `ro`.

```bash
# Comprobar la modalidad de un montaje
findmnt /mnt/efs -o TARGET,SOURCE,FSTYPE,OPTIONS
touch /mnt/efs/prueba     # en solo lectura: "Read-only file system"
```

---

## Desmontar

```bash
sudo umount /mnt/efs
# "target is busy": algún proceso tiene ficheros abiertos o su directorio de trabajo dentro
sudo lsof +f -- /mnt/efs
```

Y retirar la entrada de `/etc/fstab` si el desmontaje es definitivo.

---

## Diagnóstico de problemas de montaje

| Síntoma | Causa probable | Solución |
| --- | --- | --- |
| `mount` se queda esperando y falla por tiempo de espera | El grupo de seguridad no permite **TCP 2049** | Añadir la regla de entrada desde el SG del cliente |
| `Failed to resolve "fs-...efs...amazonaws.com"` | Sin destino de montaje en la AZ, o DNS desactivado en la VPC | Crear el destino de montaje; activar DNS en la VPC |
| `access denied by server` | La política no concede `ClientMount` a la identidad (o el cliente es anónimo) | Montar con `-o tls,iam`; revisar la política y el rol |
| `Permission denied` al escribir siendo *root* | *Root squashing* activo | Usar un usuario con permisos o un punto de acceso |
| `Read-only file system` | Montado con `ro` o sin `ClientWrite` | Revisar opciones y política |
| La instancia no arranca tras editar `fstab` | Falta `_netdev` o error de sintaxis | Probar siempre con `mount -a` |

---

## Referencias

- AWS. *Mounting EFS file systems* — [Montaje](https://docs.aws.amazon.com/efs/latest/ug/mounting-fs.html)
- AWS. *Installing the Amazon EFS client* — [amazon-efs-utils](https://docs.aws.amazon.com/efs/latest/ug/installing-amazon-efs-utils.html)
- AWS. *Mounting automatically with /etc/fstab* — [Montaje automático](https://docs.aws.amazon.com/efs/latest/ug/automount-with-efs-mount-helper.html)
- AWS. *Recommended NFS mount settings* — [Opciones NFS](https://docs.aws.amazon.com/efs/latest/ug/mounting-fs-nfs-mount-settings.html)
- AWS. *Working with Amazon EFS access points* — [Puntos de acceso](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html)
- AWS. *Troubleshooting mount issues* — [Solución de problemas](https://docs.aws.amazon.com/efs/latest/ug/troubleshooting-efs-mounting.html)
