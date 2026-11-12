# Seguridad, monitorización y auditoría (ampliación)

!!! info "Contenido de ampliación"
    Esta página es **opcional**. Recoge la versión completa del apartado del núcleo y amplía sus contenidos. No es necesaria para realizar las prácticas.

El criterio de evaluación pide configurar «los parámetros del sistema relativos a **visibilidad, acceso, seguridad digital, monitorización, observabilidad y auditoría**». Este apartado reúne, para EBS y EFS, los mecanismos que responden a cada uno de esos términos:

| Término | Pregunta | Mecanismos |
| --- | --- | --- |
| **Visibilidad** | ¿Quién puede ver o alcanzar el recurso? | Subredes privadas, grupos de seguridad, bloqueo del acceso público a instantáneas |
| **Acceso** | ¿Quién puede administrarlo y quién puede usar los datos? | IAM, política del sistema de archivos, puntos de acceso, POSIX |
| **Seguridad digital** | ¿Están protegidos los datos? | Cifrado en reposo y en tránsito, *root squashing*, copias inmutables |
| **Monitorización** | ¿Funciona bien ahora? | Métricas y alarmas de CloudWatch, comprobaciones de estado |
| **Observabilidad** | ¿Por qué se comporta así? | Correlación de métricas, paneles, registros |
| **Auditoría** | ¿Quién hizo qué y cuándo? ¿Se cumplen las normas? | CloudTrail, AWS Config, etiquetas |

---

## Visibilidad

### EBS

Un volumen EBS **no tiene dirección de red**: solo es visible para la instancia a la que está adjuntado. El riesgo de exposición está en las **instantáneas**, que pueden compartirse con otras cuentas o **hacerse públicas**. Una instantánea pública permite a cualquier cuenta de AWS crear un volumen con **todo el contenido del disco**.

=== "Consola AWS"

    **Bloquear el acceso público a las instantáneas** (por región)

    1. **EC2 → Panel → Atributos de la cuenta → Protección y seguridad de datos**.
    2. En **Bloquear el acceso público a las instantáneas de EBS**, pulsar **Administrar**.
    3. Marcar **Bloquear el acceso público** y elegir **Bloquear todo el uso compartido público**.
    4. **Actualizar**.

    **Ver con quién está compartida una instantánea**

    1. **EC2 → Instantáneas**, seleccionar la instantánea.
    2. **Acciones → Configuración de instantáneas → Modificar permisos**: muestra si es **Privada** o **Pública** y las cuentas con las que se comparte. Desde aquí también se retiran cuentas.

=== "AWS CLI"

    ```bash
    # Bloquear a nivel de región que las instantáneas se compartan públicamente
    aws ec2 enable-snapshot-block-public-access --state block-all-sharing
    aws ec2 get-snapshot-block-public-access-state

    # Ver con quién está compartida una instantánea
    aws ec2 describe-snapshot-attribute --snapshot-id snap-0abc... --attribute createVolumePermission
    ```

### EFS

Un sistema de archivos EFS solo es visible **dentro de la VPC** a través de sus destinos de montaje. La visibilidad se controla con:

- Destinos de montaje en **subredes privadas**.
- **Grupo de seguridad** con TCP 2049 solo desde los grupos de seguridad de los clientes.
- **Política del sistema de archivos** que no conceda acceso a principales externos. EFS **bloquea por defecto** las políticas que concederían acceso público.

---

## Acceso: plano de control y plano de datos

Igual que en las bases de datos, conviene distinguir:

- **Plano de control**: quién puede **crear, modificar, adjuntar, copiar o eliminar** volúmenes, instantáneas, sistemas de archivos o destinos de montaje. Lo gobierna **IAM**.
- **Plano de datos**: quién puede **leer y escribir los datos**. En EBS, quien tenga acceso a la instancia; en EFS, la combinación de red, política del sistema de archivos, punto de acceso y permisos POSIX.

Ejemplo de política de IAM que permite a un operador gestionar instantáneas pero **no borrar volúmenes**, y solo sobre recursos etiquetados con su proyecto. En la consola se crea en **IAM → Políticas → Crear política → JSON**; con la CLI, con `aws iam create-policy --policy-document file://politica.json`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GestionInstantaneasProyecto",
      "Effect": "Allow",
      "Action": ["ec2:CreateSnapshot", "ec2:CreateTags", "ec2:DescribeVolumes", "ec2:DescribeSnapshots"],
      "Resource": "*",
      "Condition": { "StringEquals": { "aws:ResourceTag/proyecto": "web" } }
    },
    {
      "Sid": "SinBorradoDeVolumenes",
      "Effect": "Deny",
      "Action": ["ec2:DeleteVolume", "elasticfilesystem:DeleteFileSystem"],
      "Resource": "*"
    }
  ]
}
```

---

## Seguridad digital

| Medida | EBS | EFS |
| --- | --- | --- |
| Cifrado en reposo | Cifrado por defecto en la región | Al crear el sistema de archivos |
| Cifrado en tránsito | Automático en volúmenes cifrados | `-o tls` y política con `aws:SecureTransport` |
| Privilegios mínimos en el cliente | Permisos POSIX | *Root squashing*, puntos de acceso, POSIX |
| Protección frente a borrado | `DeleteOnTermination=false`, Papelera de reciclaje para instantáneas | Política de IAM que deniegue `DeleteFileSystem` |
| Protección frente a *ransomware* | **Bloqueo de instantáneas** | **AWS Backup Vault Lock** (copias inmutables) |

---

## Monitorización con Amazon CloudWatch

### Métricas de EBS

EBS publica métricas en el espacio de nombres **`AWS/EBS`**, con una resolución de **1 minuto** y sin coste, mientras el volumen está adjuntado:

| Métrica | Qué indica | Uso |
| --- | --- | --- |
| `VolumeReadOps` / `VolumeWriteOps` | Operaciones de lectura y escritura en el periodo | Calcular las IOPS reales (dividiendo por los segundos del periodo) |
| `VolumeReadBytes` / `VolumeWriteBytes` | Bytes leídos y escritos | Calcular el rendimiento real |
| `VolumeAvgIOPS` / `VolumeAvgThroughput` | IOPS y rendimiento medios del último minuto | Comparar con lo aprovisionado |
| `VolumeAvgReadLatency` / `VolumeAvgWriteLatency` | Latencia media (ms) | Detectar degradación del almacenamiento |
| `VolumeQueueLength` | Peticiones pendientes | Una cola alta y sostenida indica que el volumen no da abasto |
| `VolumeIOPSExceededCheck` | 1 si la aplicación intenta superar las IOPS aprovisionadas | **Volumen infradimensionado**: subir IOPS o cambiar de tipo |
| `VolumeThroughputExceededCheck` | 1 si intenta superar el rendimiento aprovisionado | Subir MiB/s |
| `VolumeStalledIOCheck` | 1 si el volumen no completa E/S (instancias Nitro) | Problema de disponibilidad del volumen |
| `VolumeIdleTime` | Segundos sin actividad | Volúmenes sobredimensionados o sin uso |
| `BurstBalance` | % de créditos de ráfaga (`gp2`, `st1`, `sc1`) | Si cae a 0, el rendimiento baja a la base: migrar a `gp3` |

Además, cada volumen tiene **comprobaciones de estado** (*volume status checks*): `ok`, `warning`, `impaired` o `insufficient-data`.

=== "Consola AWS"

    **EC2 → Volúmenes**, seleccionar el volumen y abrir la pestaña **Comprobaciones de estado**. La columna **Comprobaciones de estado** de la lista también lo resume.

=== "AWS CLI"

    ```bash
    aws ec2 describe-volume-status --volume-ids vol-0abc...
    ```

!!! note "Espacio libre: una métrica del sistema operativo"
    EBS no sabe qué parte del volumen está ocupada: eso lo sabe el **sistema de ficheros**. Para vigilar el espacio libre (`df`) hay que instalar el **agente de CloudWatch** en la instancia, que publica métricas como `disk_used_percent` en el espacio de nombres `CWAgent`.

### Métricas de EFS

EFS publica en **`AWS/EFS`**, por sistema de archivos:

| Métrica | Qué indica |
| --- | --- |
| `StorageBytes` (dimensión `StorageClass`: `Total`, `Standard`, `IA`, `Archive`) | Espacio ocupado en cada clase: base para el control de costes y para comprobar el ciclo de vida |
| `ClientConnections` | Número de clientes conectados |
| `DataReadIOBytes`, `DataWriteIOBytes`, `MetadataIOBytes`, `TotalIOBytes` | Volumen de E/S de datos y metadatos |
| `MeteredIOBytes` | E/S facturable; en modo Elastic, relacionada directamente con el coste |
| `PermittedThroughput` | Rendimiento máximo permitido en cada momento |
| `BurstCreditBalance` | Créditos de ráfaga (modo Bursting) |
| `PercentIOLimit` | Cercanía al límite de E/S (modo General Purpose) |
| `TimeSinceLastSync` | Retraso de la replicación (en el sistema de destino) |

### Alarmas

Una métrica solo es útil si alguien reacciona a ella. Ejemplos de alarmas:

=== "Consola AWS"

    **Alarma de IOPS excedidas en un volumen EBS**

    1. **CloudWatch → Alarmas → Todas las alarmas → Crear alarma → Seleccionar métrica**.
    2. **EBS → Métricas por volumen**, buscar el ID del volumen y marcar `VolumeIOPSExceededCheck`. **Seleccionar métrica**.
    3. **Estadística**: *Máximo*. **Periodo**: *1 minuto*.
    4. **Condiciones**: *Estática*, **Mayor o igual que** `1`. En **Configuración adicional**, **Puntos de datos para la alarma**: `5` de `5`.
    5. **Notificación**: *En alarma* → tema de SNS `alertas-almacenamiento` (o crear uno nuevo con un correo).
    6. **Nombre**: `ebs-datos-iops-excedidas`. **Crear alarma**.

    **Alarma de tamaño de un sistema EFS**

    Igual que la anterior, eligiendo **EFS → Métricas de almacenamiento** (dimensiones *FileSystemId* y *StorageClass*), la métrica `StorageBytes` con `StorageClass = Total`, estadística *Promedio*, periodo *1 hora* y condición **Mayor que** `500000000000`.

    Desde la pestaña **Supervisión** de un volumen o de un sistema de archivos también se accede a sus gráficas y, desde cada gráfica, a la creación de alarmas.

=== "AWS CLI"

    ```bash
    # Volumen EBS que intenta superar sus IOPS durante 5 minutos seguidos
    aws cloudwatch put-metric-alarm --alarm-name ebs-datos-iops-excedidas \
      --namespace AWS/EBS --metric-name VolumeIOPSExceededCheck \
      --dimensions Name=VolumeId,Value=vol-0abc... \
      --statistic Maximum --period 60 --evaluation-periods 5 \
      --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
      --alarm-actions arn:aws:sns:us-east-1:111122223333:alertas-almacenamiento

    # Sistema EFS que supera 500 GB (control de costes)
    aws cloudwatch put-metric-alarm --alarm-name efs-web-mas-de-500GB \
      --namespace AWS/EFS --metric-name StorageBytes \
      --dimensions Name=FileSystemId,Value=fs-0abc123 Name=StorageClass,Value=Total \
      --statistic Average --period 3600 --evaluation-periods 1 \
      --threshold 500000000000 --comparison-operator GreaterThanThreshold \
      --alarm-actions arn:aws:sns:us-east-1:111122223333:alertas-almacenamiento
    ```


---

## Observabilidad

La monitorización dice **que** algo va mal; la observabilidad permite entender **por qué**. En almacenamiento consiste en **correlacionar** métricas:

| Síntoma | Correlación | Diagnóstico |
| --- | --- | --- |
| La aplicación responde lento | `VolumeQueueLength` alta + `VolumeIOPSExceededCheck` = 1 | El volumen EBS está limitado por IOPS |
| La aplicación responde lento | Latencia alta pero IOPS por debajo de lo aprovisionado | Límite de la **instancia** hacia EBS, o patrón de E/S ineficiente |
| Lentitud repentina en `gp2` | `BurstBalance` cae a 0 | Créditos agotados: migrar a `gp3` |
| Lentitud en EFS Bursting | `BurstCreditBalance` cae a 0 | Cambiar a Elastic |
| Factura de EFS creciente | `StorageBytes` de `Standard` no baja | El ciclo de vida no está configurado o los datos no son fríos |

Un **panel de CloudWatch** (*dashboard*) que reúna estas métricas junto a las de la instancia (CPU, red) y las de la aplicación es la herramienta habitual. **Compute Optimizer** analiza el historial de métricas de los volúmenes y recomienda ajustes.

---

## Auditoría

### AWS CloudTrail

CloudTrail registra **todas las llamadas a la API** de EBS y EFS, con la identidad, la hora, la IP de origen y los parámetros: `CreateVolume`, `AttachVolume`, `ModifyVolume`, `DeleteVolume`, `CreateSnapshot`, `ModifySnapshotAttribute`, `CreateFileSystem`, `PutFileSystemPolicy`, `DeleteFileSystem`…

En EFS, cuando los clientes montan con **autorización IAM**, CloudTrail registra además los eventos **`NewClientConnection`** y **`UpdateClientConnection`**, con la identidad del cliente y los permisos concedidos (montaje, escritura, *root*). Un intento de montaje anónimo rechazado aparece con la identidad `ANONYMOUS_PRINCIPAL` y el error `AccessDenied`.

=== "Consola AWS"

    1. **CloudTrail → Historial de eventos**.
    2. En el filtro, elegir **Nombre del evento** y escribir `DeleteVolume` (o `NewClientConnection`, `PutFileSystemPolicy`…).
    3. Ajustar el intervalo de fechas. Cada fila muestra la hora, el usuario y el recurso; al abrirla se ve el registro JSON completo.

=== "AWS CLI"

    ```bash
    aws cloudtrail lookup-events \
      --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteVolume \
      --max-results 10 --query "Events[].[EventTime,Username]" --output table
    ```

### AWS Config

AWS Config evalúa de forma continua si los recursos cumplen reglas. En la consola se añaden en **AWS Config → Reglas → Agregar regla → Agregar regla administrada por AWS**; con la CLI, con `aws configservice put-config-rule`. Reglas administradas útiles:

| Regla | Comprueba |
| --- | --- |
| `ec2-ebs-encryption-by-default` | Que el cifrado por defecto de EBS esté activo |
| `encrypted-volumes` | Que los volúmenes adjuntados estén cifrados |
| `ebs-snapshot-public-restorable-check` | Que no haya instantáneas públicas |
| `efs-encrypted-check` | Que los sistemas EFS estén cifrados |
| `ebs-in-backup-plan` / `efs-in-backup-plan` | Que los recursos estén incluidos en un plan de AWS Backup |

### Etiquetas

Las etiquetas (`proyecto`, `entorno`, `propietario`, `backup`) no son solo organización: permiten **asignar costes**, **seleccionar recursos** en planes de copia y políticas de DLM, y **restringir permisos** de IAM por recurso, como en el ejemplo anterior.

---

## Referencias

- AWS. *Amazon CloudWatch metrics for Amazon EBS* — [Métricas de EBS](https://docs.aws.amazon.com/ebs/latest/userguide/using_cloudwatch_ebs.html)
- AWS Storage Blog. *Understanding and monitoring latency for Amazon EBS volumes* — [Latencia de EBS](https://aws.amazon.com/blogs/storage/understanding-and-monitoring-latency-for-amazon-ebs-volumes-using-amazon-cloudwatch)
- AWS. *Amazon CloudWatch metrics for Amazon EFS* — [Métricas de EFS](https://docs.aws.amazon.com/efs/latest/ug/efs-metrics.html)
- AWS. *Logging Amazon EFS API calls with AWS CloudTrail* — [CloudTrail y EFS](https://docs.aws.amazon.com/efs/latest/ug/logging-using-cloudtrail.html)
- AWS. *Block public access for Amazon EBS snapshots* — [Bloqueo del acceso público](https://docs.aws.amazon.com/ebs/latest/userguide/block-public-access-snapshots.html)
- AWS Config. *List of AWS Config managed rules* — [Reglas administradas](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
