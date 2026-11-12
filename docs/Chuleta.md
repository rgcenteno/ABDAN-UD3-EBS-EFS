# Chuleta de comandos

Recopilación de los comandos más habituales de la unidad, como orientación para los laboratorios. Los valores entre `< >` deben sustituirse por los propios. Todos los comandos de la AWS CLI se ejecutan en la región configurada (`aws configure set region us-east-1`).

!!! tip "Antes de empezar"
    - `aws sts get-caller-identity` indica con qué identidad se está trabajando.
    - Añadir `--query` (sintaxis JMESPath) y `--output table` hace la salida mucho más legible.
    - Los comandos de **sistema operativo** (`lsblk`, `mkfs`, `mount`…) se ejecutan **dentro de la instancia**; los de `aws` pueden ejecutarse desde cualquier equipo con credenciales (CloudShell, la propia instancia…).

---

## Amazon EBS

### Volúmenes

```bash
# AZ de una instancia
aws ec2 describe-instances --instance-ids <i-id> \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text

# Crear un volumen gp3 cifrado
aws ec2 create-volume --availability-zone us-east-1a --size 10 --volume-type gp3 \
  --encrypted --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=datos},{Key=backup,Value=diario}]'
aws ec2 wait volume-available --volume-ids <vol-id>

# Listar volúmenes
aws ec2 describe-volumes \
  --query "Volumes[].[VolumeId,Size,VolumeType,State,AvailabilityZone,Encrypted]" --output table

# Volúmenes huérfanos (se pagan sin usarse)
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query "Volumes[].[VolumeId,Size,CreateTime]" --output table

# Adjuntar / desadjuntar
aws ec2 attach-volume --volume-id <vol-id> --instance-id <i-id> --device /dev/sdf
aws ec2 detach-volume --volume-id <vol-id>

# Modificar en caliente (tamaño, tipo, IOPS, rendimiento)
aws ec2 modify-volume --volume-id <vol-id> --size 20
aws ec2 modify-volume --volume-id <vol-id> --volume-type gp3 --iops 6000 --throughput 250
aws ec2 describe-volumes-modifications --volume-ids <vol-id> \
  --query "VolumesModifications[0].[ModificationState,Progress]"

# No borrar el volumen raíz al terminar la instancia
aws ec2 modify-instance-attribute --instance-id <i-id> \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"DeleteOnTermination":false}}]'

# Eliminar
aws ec2 delete-volume --volume-id <vol-id>

# Estado y salud
aws ec2 describe-volume-status --volume-ids <vol-id>
```

### Dentro de la instancia

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,SERIAL     # SERIAL = ID del volumen
sudo file -s /dev/nvme1n1                            # "data" = vacío
sudo mkfs -t xfs /dev/nvme1n1                        # ¡solo si está vacío!
sudo mkdir -p /datos && sudo mount /dev/nvme1n1 /datos
sudo mount -o ro /dev/nvme1n1 /datos                 # solo lectura
sudo mount -o remount,rw /datos                      # pasar a lectura/escritura

# Montaje permanente
sudo blkid /dev/nvme1n1
echo 'UUID=<uuid>  /datos  xfs  defaults,nofail  0  2' | sudo tee -a /etc/fstab
sudo umount /datos && sudo mount -a && findmnt /datos

# Ampliar el sistema de ficheros tras modify-volume
sudo xfs_growfs -d /datos                 # xfs sin particiones
sudo resize2fs /dev/nvme1n1               # ext4 sin particiones
sudo growpart /dev/nvme0n1 1 && sudo xfs_growfs -d /     # volumen raíz con partición

# Desmontar
sudo lsof +f -- /datos                    # quién lo está usando
sudo umount /datos
```

### Instantáneas

```bash
aws ec2 create-snapshot --volume-id <vol-id> --description "Antes del cambio" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=pre-cambio}]'
aws ec2 wait snapshot-completed --snapshot-ids <snap-id>

# Instantáneas coherentes de todos los volúmenes de una instancia
aws ec2 create-snapshots --instance-specification InstanceId=<i-id> --copy-tags-from-source volume

aws ec2 describe-snapshots --owner-ids self \
  --query "sort_by(Snapshots,&StartTime)[].[SnapshotId,VolumeId,StartTime,State,Encrypted]" --output table

# Restaurar: volumen nuevo (en cualquier AZ de la región)
aws ec2 create-volume --snapshot-id <snap-id> --availability-zone us-east-1b --volume-type gp3

# Copiar a otra región (se ejecuta en la región de destino) y cifrar
aws ec2 copy-snapshot --region eu-south-2 --source-region us-east-1 \
  --source-snapshot-id <snap-id> --encrypted --kms-key-id alias/<clave>

aws ec2 delete-snapshot --snapshot-id <snap-id>
```

### Cifrado y protección de la cuenta

```bash
aws ec2 enable-ebs-encryption-by-default
aws ec2 get-ebs-encryption-by-default
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/<clave>

aws ec2 enable-snapshot-block-public-access --state block-all-sharing
aws ec2 get-snapshot-block-public-access-state
```

---

## Amazon EFS

### Sistema de archivos

```bash
aws efs create-file-system --creation-token <token-unico> \
  --performance-mode generalPurpose --throughput-mode elastic \
  --encrypted --backup --tags Key=Name,Value=efs-web
# One Zone: añadir --availability-zone-name us-east-1a

aws efs describe-file-systems \
  --query "FileSystems[].[FileSystemId,Name,LifeCycleState,ThroughputMode,Encrypted,SizeInBytes.Value]" --output table

# Ciclo de vida
aws efs put-lifecycle-configuration --file-system-id <fs-id> --lifecycle-policies \
  '[{"TransitionToIA":"AFTER_30_DAYS"},{"TransitionToArchive":"AFTER_90_DAYS"},{"TransitionToPrimaryStorageClass":"AFTER_1_ACCESS"}]'
aws efs describe-lifecycle-configuration --file-system-id <fs-id>
```

### Red: grupo de seguridad y destinos de montaje

```bash
aws ec2 create-security-group --group-name efs-sg --description "NFS desde web" --vpc-id <vpc-id>
aws ec2 authorize-security-group-ingress --group-id <sg-efs> \
  --protocol tcp --port 2049 --source-group <sg-web>

aws efs create-mount-target --file-system-id <fs-id> --subnet-id <subnet-a> --security-groups <sg-efs>
aws efs create-mount-target --file-system-id <fs-id> --subnet-id <subnet-b> --security-groups <sg-efs>
aws efs describe-mount-targets --file-system-id <fs-id> \
  --query "MountTargets[].[MountTargetId,AvailabilityZoneName,IpAddress,LifeCycleState]" --output table
aws efs modify-mount-target-security-groups --mount-target-id <fsmt-id> --security-groups <sg-efs>
```

### Política, puntos de acceso y copias

```bash
aws efs put-file-system-policy --file-system-id <fs-id> --policy file://politica-efs.json
aws efs describe-file-system-policy --file-system-id <fs-id>
aws efs delete-file-system-policy --file-system-id <fs-id>

aws efs create-access-point --file-system-id <fs-id> --posix-user Uid=48,Gid=48 \
  --root-directory 'Path=/web,CreationInfo={OwnerUid=48,OwnerGid=48,Permissions=755}'
aws efs describe-access-points --file-system-id <fs-id>

aws efs put-backup-policy --file-system-id <fs-id> --backup-policy Status=ENABLED
aws efs describe-backup-policy --file-system-id <fs-id>
```

### Montaje en la instancia

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls <fs-id>:/ /mnt/efs                       # TLS
sudo mount -t efs -o tls,iam <fs-id>:/ /mnt/efs                   # TLS + identidad IAM
sudo mount -t efs -o tls,accesspoint=<fsap-id> <fs-id>:/ /mnt/efs # punto de acceso
sudo mount -t efs -o tls,ro <fs-id>:/ /mnt/efs                    # solo lectura

# Cliente NFS estándar
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  <fs-id>.efs.us-east-1.amazonaws.com:/ /mnt/efs

# /etc/fstab
<fs-id>:/  /mnt/efs  efs  _netdev,noresvport,tls,iam  0  0

findmnt /mnt/efs -o TARGET,SOURCE,FSTYPE,OPTIONS
sudo umount /mnt/efs
```

### Replicación

```bash
aws efs create-replication-configuration --source-file-system-id <fs-id> --destinations Region=eu-south-2
aws efs describe-replication-configurations --file-system-id <fs-id>
aws efs delete-replication-configuration --source-file-system-id <fs-id>      # failover
```

### Eliminar

```bash
aws efs delete-access-point --access-point-id <fsap-id>
aws efs delete-mount-target --mount-target-id <fsmt-id>        # todos los destinos
aws efs delete-file-system --file-system-id <fs-id>
```

---

## Copias de seguridad

```bash
# Data Lifecycle Manager
aws dlm create-lifecycle-policy --description "<descripcion>" --state ENABLED \
  --execution-role-arn arn:aws:iam::<cuenta>:role/LabRole --policy-details file://politica-dlm.json
aws dlm get-lifecycle-policies --query "Policies[].[PolicyId,Description,State]" --output table
aws dlm delete-lifecycle-policy --policy-id <policy-id>

# Instantáneas creadas por una política de DLM
aws ec2 describe-snapshots --owner-ids self \
  --filters Name=tag:aws:dlm:lifecycle-policy-id,Values=<policy-id> \
  --query "Snapshots[].[SnapshotId,StartTime,State]" --output table

# AWS Backup: puntos de recuperación de un recurso
aws backup list-recovery-points-by-resource --resource-arn <arn-recurso>
```

---

## Monitorización

```bash
# Métricas disponibles de un volumen o de un sistema de archivos
aws cloudwatch list-metrics --namespace AWS/EBS --dimensions Name=VolumeId,Value=<vol-id>
aws cloudwatch list-metrics --namespace AWS/EFS --dimensions Name=FileSystemId,Value=<fs-id>

# Espacio ocupado en EFS por clase en la última hora
aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name StorageBytes \
  --dimensions Name=FileSystemId,Value=<fs-id> Name=StorageClass,Value=Total \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ) \
  --period 3600 --statistics Average

# Auditoría: quién borró volúmenes
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteVolume \
  --query "Events[].[EventTime,Username]" --output table
```

---

## Infraestructura como código

```bash
aws cloudformation validate-template --template-body file://cifrado-basico.yaml
aws cloudformation deploy --stack-name ud3-cifrado --template-file cifrado-basico.yaml \
  --parameter-overrides ZonaVolumen=us-east-1a
aws cloudformation describe-stacks --stack-name ud3-cifrado --query "Stacks[0].Outputs" --output table
aws cloudformation delete-stack --stack-name ud3-cifrado
```
