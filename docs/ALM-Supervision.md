# Seguridad, monitorización y auditoría

Este apartado reúne, para EBS y EFS, los mecanismos que responden a los términos del criterio de evaluación: **visibilidad, acceso, seguridad, monitorización, observabilidad y auditoría**. La mayoría ya se han visto en los apartados anteriores; aquí se ordenan y se completan con CloudWatch y CloudTrail.

| Término | Pregunta | EBS | EFS |
| --- | --- | --- | --- |
| **Visibilidad** | ¿Quién puede alcanzar los datos? | Solo la instancia a la que está adjuntado. Cuidado con las **instantáneas públicas** | Solo dentro de la VPC: subredes privadas y **grupo de seguridad** (TCP 2049) |
| **Acceso** | ¿Quién administra y quién usa los datos? | IAM (quién crea, adjunta o borra) y permisos POSIX | IAM, **política del sistema de archivos**, puntos de acceso y POSIX |
| **Seguridad** | ¿Están protegidos los datos? | Cifrado por defecto | Cifrado en reposo y TLS, *root squashing* |
| **Monitorización** | ¿Funciona bien ahora? | Métricas y alarmas de **CloudWatch** | Métricas y alarmas de **CloudWatch** |
| **Auditoría** | ¿Quién hizo qué y cuándo? | **CloudTrail** | **CloudTrail** (incluidas las conexiones de clientes) |

---

## Monitorización con CloudWatch

Las métricas se publican automáticamente cada minuto y sin coste. Las más útiles:

| Servicio | Métrica | Qué indica |
| --- | --- | --- |
| EBS | `VolumeReadOps`, `VolumeWriteOps` | Operaciones realizadas: permiten calcular las IOPS reales |
| EBS | `VolumeQueueLength` | Peticiones en espera. Alta y sostenida: el volumen no da abasto |
| EBS | `VolumeIOPSExceededCheck` | 1 si la aplicación intenta superar las IOPS del volumen: está **infradimensionado** |
| EBS | `BurstBalance` | Créditos de ráfaga de `gp2`. Si llega a 0, el rendimiento cae: migrar a `gp3` |
| EFS | `StorageBytes` (por clase) | Espacio ocupado en Standard, IA y Archive: control de costes y del ciclo de vida |
| EFS | `ClientConnections` | Clientes conectados |
| EFS | `MeteredIOBytes` | E/S facturable; en modo Elastic, relacionada directamente con el coste |

!!! note "Espacio libre en un volumen EBS"
    EBS no sabe cuánto espacio ocupa el sistema de ficheros. Para vigilar el `df` de una instancia hay que instalar el **agente de CloudWatch**.

### Crear una alarma

=== "Consola AWS"

    1. **CloudWatch → Alarmas → Todas las alarmas → Crear alarma → Seleccionar métrica**.
    2. **EBS → Métricas por volumen**, buscar el ID del volumen y marcar `VolumeIOPSExceededCheck`. **Seleccionar métrica**.
    3. **Estadística**: *Máximo*. **Periodo**: *1 minuto*. **Condición**: *Mayor o igual que* `1`.
    4. **Notificación**: *En alarma* → un tema de SNS con un correo de destino.
    5. **Nombre**: `ebs-datos-iops-excedidas` → **Crear alarma**.

=== "AWS CLI"

    ```bash
    aws cloudwatch put-metric-alarm --alarm-name ebs-datos-iops-excedidas \
      --namespace AWS/EBS --metric-name VolumeIOPSExceededCheck \
      --dimensions Name=VolumeId,Value=vol-0abc... \
      --statistic Maximum --period 60 --evaluation-periods 5 \
      --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
      --alarm-actions arn:aws:sns:us-east-1:111122223333:alertas-almacenamiento
    ```

La pestaña **Supervisión** de cada volumen o sistema de archivos muestra sus gráficas sin salir de la consola del servicio.

---

## Auditoría con CloudTrail

CloudTrail registra **todas las llamadas a la API**: quién creó, adjuntó, modificó o borró un volumen, una instantánea o un sistema de archivos, y cuándo. En EFS, cuando los clientes montan con `-o tls,iam`, registra además cada conexión (`NewClientConnection`) con la identidad del cliente. Los intentos de montaje anónimo rechazados aparecen como `ANONYMOUS_PRINCIPAL` con `AccessDenied`.

=== "Consola AWS"

    1. **CloudTrail → Historial de eventos**.
    2. Filtro **Nombre del evento**: `DeleteVolume` (o `NewClientConnection`, `PutFileSystemPolicy`…).
    3. Cada fila muestra la hora, el usuario y el recurso; al abrirla se ve el registro completo.

=== "AWS CLI"

    ```bash
    aws cloudtrail lookup-events \
      --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteVolume \
      --max-results 10 --query "Events[].[EventTime,Username]" --output table
    ```

!!! info "Para saber más"
    La [ampliación de monitorización](AMP-Supervision.md) incluye el resto de métricas, la observabilidad (cómo correlacionarlas para diagnosticar), el bloqueo del acceso público a instantáneas y las reglas de cumplimiento de AWS Config.

---

## Referencias

- AWS. *Amazon CloudWatch metrics for Amazon EBS* — [Métricas de EBS](https://docs.aws.amazon.com/ebs/latest/userguide/using_cloudwatch_ebs.html)
- AWS. *Amazon CloudWatch metrics for Amazon EFS* — [Métricas de EFS](https://docs.aws.amazon.com/efs/latest/ug/efs-metrics.html)
- AWS. *Logging Amazon EFS API calls with AWS CloudTrail* — [CloudTrail y EFS](https://docs.aws.amazon.com/efs/latest/ug/logging-using-cloudtrail.html)
