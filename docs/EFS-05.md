# Creación de un sistema de archivos EFS

La consola de EFS ofrece dos caminos: **Creación rápida**, que crea un sistema de archivos con los valores recomendados en la VPC elegida, y **Personalizar**, un asistente de cuatro pasos en el que se revisa cada opción. Este apartado recorre el asistente personalizado explicando cada opción y la recomendación más habitual para entornos generales.

| Paso | Contenido |
| --- | --- |
| 1 | Configuración del sistema de archivos: nombre, tipo, copias, ciclo de vida, cifrado, rendimiento |
| 2 | Acceso a la red: VPC, destinos de montaje, subredes y grupos de seguridad |
| 3 | Política del sistema de archivos (opcional), que se estudia en el apartado siguiente |
| 4 | Revisar y crear |

---

## Paso 1. Configuración del sistema de archivos

### Nombre

Es una **etiqueta** (`Name`) para identificar fácilmente el sistema de archivos. No afecta al funcionamiento ni al rendimiento. Se recomienda una nomenclatura clara y consistente, por ejemplo `efs-produccion`, `efs-desarrollo` o `efs-web-contenidos`.

### Tipo de sistema de archivos

| Opción | Descripción | Recomendación |
| --- | --- | --- |
| **Regional** | Datos redundantes en varias AZ de la región; mayor disponibilidad y resiliencia | **Producción** |
| **Única zona** (*One Zone*) | Datos en una sola AZ, que se elige aquí; más económica, menos protección | Desarrollo y pruebas |

Para la mayoría de los entornos de producción, la recomendación es **Regional**. **No se puede cambiar después.**

### Copias de seguridad automáticas

Integra el sistema de archivos con **AWS Backup** mediante un plan predeterminado: **una copia diaria con retención de 35 días**. Salvo que exista una política de copias corporativa específica, se recomienda **mantenerla activada**, especialmente si el sistema almacena información importante. Puede desactivarse o sustituirse por un plan propio más adelante.

### Administración del ciclo de vida

| Opción | Valor recomendado | Comentario |
| --- | --- | --- |
| **Transición a IA** | **30 días** desde el último acceso | En entornos con poca actividad también se usan 60 o 90 días |
| **Transición a Archive** | **90 días** desde el último acceso | O *Ninguna* si se requiere acceso rápido a todos los datos |
| **Transición a Standard** | **Al primer acceso** | Recupera el rendimiento de forma transparente cuando se vuelven a usar los datos |

### Cifrado

La opción **Habilitar el cifrado de los datos en reposo** protege los datos mediante AWS KMS. La recomendación general es **habilitarlo siempre**: mejora la seguridad, apenas tiene impacto operativo y **no puede activarse después**.

Para la **clave KMS**, la administrada por AWS (`aws/elasticfilesystem`) es suficiente en la mayoría de los casos; una clave propia se usa cuando hay requisitos de cumplimiento o de control de accesos más estrictos.

### Configuración del rendimiento

| Opción | Recomendación |
| --- | --- |
| **Modo de rendimiento** (*throughput*) | **Elástico**: se ajusta solo a la actividad y se paga lo que se lee y escribe |
| **Modo de desempeño** (*performance*) | **Uso general**: la menor latencia, válido en la práctica totalidad de los casos |

Los modos Aprovisionado, Ráfagas y E/S máxima se describen en la [ampliación sobre rendimiento](EFS-03.md).

---

## Paso 2. Acceso a la red

### VPC

Se selecciona la VPC donde se crearán los destinos de montaje. Lo habitual es elegir **la misma VPC** en la que están las instancias EC2, los contenedores ECS o los servicios que van a usar EFS.

### Destinos de montaje

AWS recomienda crear un destino de montaje **en cada zona de disponibilidad** donde existan servidores que vayan a utilizar EFS. En entornos sencillos puede bastar con uno, pero en producción se despliega uno por cada AZ para mejorar la disponibilidad y reducir la latencia. El asistente propone uno por AZ y permite eliminar los que no se necesiten.

Para cada destino de montaje se indica:

| Opción | Recomendación |
| --- | --- |
| **Zona de disponibilidad** | Las AZ en las que haya clientes |
| **Subred** | Una subred de esa AZ, preferiblemente **privada** cuando el acceso es desde servidores internos |
| **Tipo de dirección IP** | **IPv4** salvo que la red ya trabaje con IPv6 (también IPv6 o doble pila) |
| **Dirección IP** | Automática, o una IP concreta de la subred |
| **Grupos de seguridad** | Un grupo específico que solo admita **TCP 2049** desde los grupos de seguridad de los clientes |

!!! warning "Grupos de seguridad"
    Es uno de los aspectos más importantes. El asistente asigna por defecto el **grupo de seguridad predeterminado de la VPC**, que normalmente **no** admite tráfico desde las instancias. Se debe sustituir por un grupo propio (por ejemplo, `efs-sg`) que permita únicamente el acceso desde los grupos de seguridad de los servicios que realmente necesitan el sistema de archivos, evitando abrir el acceso a rangos amplios de direcciones IP.

---

## Paso 3. Política del sistema de archivos

Paso opcional en el que se definen permisos a nivel de AWS: impedir el acceso *root*, forzar solo lectura, evitar el acceso anónimo y exigir cifrado en tránsito. Se estudia en el apartado de **políticas del sistema de archivos y permisos**.

## Paso 4. Revisar y crear

Se revisa el resumen y se pulsa **Crear**. El sistema de archivos pasa a estado **Disponible** en pocos segundos, pero los **destinos de montaje** tardan algo más (estado *Creando* → *Disponible*). Hasta que estén disponibles no se puede montar.

---

## Resumen: un EFS «estándar»

Para una aplicación corporativa habitual, una configuración segura, altamente disponible y escalable sería:

- ✅ Sistema de archivos **Regional**
- ✅ Copias de seguridad automáticas **activadas**
- ✅ Transición a IA tras **30 días**
- ✅ Transición a Archive tras **90 días** (o desactivada si los datos se consultan con frecuencia)
- ✅ Vuelta a Standard **al primer acceso**
- ✅ Cifrado en reposo **activado**, con la clave administrada por AWS (o propia si hay requisitos de cumplimiento)
- ✅ Modo de rendimiento **Elástico** y modo de desempeño **Uso general**
- ✅ Destino de montaje en **cada AZ** utilizada, en **subredes privadas**
- ✅ Acceso restringido mediante **grupos de seguridad específicos**

---

## Resumen del procedimiento

=== "Consola AWS"

    1. **EFS → Sistemas de archivos → Crear sistema de archivos → Personalizar**.
    2. **Paso 1**: nombre `efs-web-contenidos`, tipo **Regional**, copias automáticas activadas, ciclo de vida 30 / 90 / al primer acceso, cifrado activado, modo **Elástico** y **Uso general**, etiqueta `entorno = produccion`. **Siguiente**.
    3. **Paso 2**: VPC de las instancias; un destino de montaje por AZ, en subredes privadas, con el grupo de seguridad `efs-sg`. **Siguiente**.
    4. **Paso 3**: política del sistema de archivos (opcional). **Siguiente**.
    5. **Paso 4**: revisar y **Crear**.
    6. Comprobar en la pestaña **Red** del sistema que los destinos de montaje pasan a **Disponible**.

=== "AWS CLI"

    ```bash
    # 1. Sistema de archivos (Regional; para One Zone añadir --availability-zone-name us-east-1a)
    aws efs create-file-system \
      --creation-token efs-web-contenidos \
      --performance-mode generalPurpose \
      --throughput-mode elastic \
      --encrypted \
      --backup \
      --tags Key=Name,Value=efs-web-contenidos Key=entorno,Value=produccion

    aws efs describe-file-systems --file-system-id fs-0abc... \
      --query "FileSystems[0].LifeCycleState"          # esperar a "available"

    # 2. Política de ciclo de vida
    aws efs put-lifecycle-configuration --file-system-id fs-0abc... \
      --lifecycle-policies '[{"TransitionToIA":"AFTER_30_DAYS"},{"TransitionToArchive":"AFTER_90_DAYS"},{"TransitionToPrimaryStorageClass":"AFTER_1_ACCESS"}]'

    # 3. Un destino de montaje por AZ
    aws efs create-mount-target --file-system-id fs-0abc... \
      --subnet-id subnet-privada-a --security-groups sg-efs...
    aws efs create-mount-target --file-system-id fs-0abc... \
      --subnet-id subnet-privada-b --security-groups sg-efs...

    aws efs describe-mount-targets --file-system-id fs-0abc... \
      --query "MountTargets[].[AvailabilityZoneName,IpAddress,LifeCycleState]" --output table
    ```


---

## Modificaciones posteriores

| Parámetro | ¿Se puede cambiar? | Consola AWS | AWS CLI |
| --- | --- | --- | --- |
| Nombre y etiquetas | Sí | Pestaña **Etiquetas → Administrar etiquetas** | `aws efs tag-resource` |
| Tipo (Regional / One Zone) | **No** | Crear otro y copiar los datos (o restaurar una copia) | — |
| Cifrado en reposo y clave KMS | **No** | Crear otro cifrado y copiar los datos | — |
| Modo de desempeño | **No** | Crear otro | — |
| Modo de rendimiento (*throughput*) | Sí (espera de 24 h tras aumentar el aprovisionado) | Sección **General → Editar** | `update-file-system` |
| Política de ciclo de vida | Sí | Sección **General → Editar** | `put-lifecycle-configuration` |
| Copias automáticas | Sí | Sección **General → Editar → Copias de seguridad automáticas** | `put-backup-policy` con `Status=ENABLED` o `Status=DISABLED` |
| Destinos de montaje | Añadir o eliminar; cambiar sus grupos de seguridad | Pestaña **Red → Administrar** | `create-mount-target`, `delete-mount-target`, `modify-mount-target-security-groups` |
| Política del sistema de archivos | Sí | Pestaña **Política del sistema de archivos → Editar** | `put-file-system-policy`, `delete-file-system-policy` |

---

## Eliminar un sistema de archivos

La eliminación es **irreversible** y destruye todos los datos. El orden es:

1. **Desmontar** el sistema de archivos en todos los clientes y retirar sus líneas de `/etc/fstab`.
2. Si está configurada, **eliminar la replicación**.
3. **Eliminar los destinos de montaje** (y, si se desea, los puntos de acceso).
4. **Eliminar el sistema de archivos**.

=== "Consola AWS"

    1. Comprobar en la pestaña **Replicación** que no hay replicación configurada (si la hay, **Eliminar replicación**).
    2. **EFS → Sistemas de archivos**, seleccionar el sistema y pulsar **Eliminar**.
    3. Escribir el **ID del sistema de archivos** para confirmar y pulsar **Confirmar**.

    La consola elimina también los **destinos de montaje** asociados antes de borrar el sistema de archivos.

=== "AWS CLI"

    ```bash
    aws efs describe-mount-targets --file-system-id fs-0abc... \
      --query "MountTargets[].MountTargetId" --output text

    aws efs delete-mount-target --mount-target-id fsmt-0aaa...
    aws efs delete-mount-target --mount-target-id fsmt-0bbb...
    aws efs delete-file-system --file-system-id fs-0abc...
    ```

!!! warning "Las copias de AWS Backup sobreviven al sistema de archivos"
    Los puntos de recuperación guardados en el almacén de AWS Backup **no se borran** al eliminar el sistema de archivos: siguen permitiendo restaurarlo y **siguen generando coste** hasta que caduca su retención o se eliminan.

---

## Referencias

- AWS. *Creating EFS file systems* — [Creación de sistemas de archivos](https://docs.aws.amazon.com/efs/latest/ug/creating-using-create-fs.html)
- AWS. *CreateFileSystem* — [Referencia de la API](https://docs.aws.amazon.com/efs/latest/ug/API_CreateFileSystem.html)
- AWS. *Creating and managing mount targets* — [Destinos de montaje](https://docs.aws.amazon.com/efs/latest/ug/manage-fs-access.html)
- AWS. *Deleting an EFS file system* — [Eliminación](https://docs.aws.amazon.com/efs/latest/ug/delete-efs-fs.html)
