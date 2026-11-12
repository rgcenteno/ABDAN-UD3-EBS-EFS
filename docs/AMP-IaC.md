# Cifrado e infraestructura como código (ampliación)

!!! info "Contenido de ampliación"
    Esta página es **opcional**. Recoge la versión completa del apartado del núcleo y amplía sus contenidos. No es necesaria para realizar las prácticas.

El criterio de evaluación pide configurar «los parámetros del sistema relativos a cifrado, mediante herramientas proporcionadas por la entidad proveedora de nube **vía IaC**». Este apartado repasa cómo funciona el cifrado en EBS y EFS, cómo se gestionan las claves en **AWS KMS** y cómo se fija todo ello en una **plantilla de infraestructura como código**, de modo que el cifrado no dependa de que alguien recuerde marcar una casilla.

!!! warning "Alcance de este apartado"
    Como en la unidad de bases de datos, la infraestructura como código se presenta **a nivel aplicado**: una plantilla completa y comentada para desplegar almacenamiento cifrado. El estudio en profundidad de CloudFormation y Terraform se aborda en el módulo **Despliegue de servicios administrados en la nube**.

---

## Cifrado en EBS

| Aspecto | Detalle |
| --- | --- |
| **Algoritmo** | AES-256, con claves de AWS KMS |
| **Qué protege** | Datos en reposo del volumen, datos en tránsito entre la instancia y el volumen, instantáneas y volúmenes creados a partir de ellas |
| **Rendimiento** | Sin impacto apreciable: el cifrado se realiza en el hardware del servidor anfitrión (Nitro) |
| **Momento** | Al **crear** el volumen. Un volumen existente **no puede cifrarse ni descifrarse** directamente |
| **Claves** | `aws/ebs` (administrada por AWS) o una clave administrada por el cliente |

### Cifrado por defecto

El cifrado por defecto es un ajuste **por región y cuenta**: una vez activado, **todos los volúmenes nuevos** (y las copias de instantáneas sin cifrar) se cifran automáticamente, aunque quien los cree no lo indique.

=== "Consola AWS"

    1. **EC2 → Panel**; en el lateral derecho, **Atributos de la cuenta → Protección y seguridad de datos**.
    2. En **Cifrado de EBS**, pulsar **Administrar**.
    3. Marcar **Habilitar** en *Cifrado predeterminado* y elegir la **Clave de cifrado predeterminada** (`aws/ebs` o `alias/ud3-almacenamiento`).
    4. **Actualizar cifrado de EBS**. El ajuste solo afecta a la **región seleccionada**: hay que repetirlo en cada región que se utilice.

=== "AWS CLI"

    ```bash
    aws ec2 enable-ebs-encryption-by-default
    aws ec2 get-ebs-encryption-by-default

    # Usar una clave propia como clave por defecto
    aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/ud3-almacenamiento
    aws ec2 get-ebs-default-kms-key-id
    ```

### Cifrar un volumen existente sin cifrar

Como el cifrado no se puede activar sobre un volumen existente, se pasa por una instantánea:

1. Crear una **instantánea** del volumen sin cifrar.
2. **Copiar** la instantánea indicando cifrado (o crear directamente un volumen desde ella con `--encrypted`).
3. Crear un **volumen nuevo** a partir de la instantánea cifrada.
4. Desmontar y desadjuntar el volumen antiguo, adjuntar y montar el nuevo.

=== "Consola AWS"

    1. **Volúmenes** → seleccionar el volumen sin cifrar → **Acciones → Crear instantánea**.
    2. **Instantáneas** → seleccionar la nueva instantánea (estado *Completada*) → **Acciones → Copiar instantánea**: misma región, marcar **Cifrar esta instantánea** y elegir `alias/ud3-almacenamiento` → **Copiar instantánea**.
    3. Seleccionar la copia cifrada → **Acciones → Crear volumen a partir de una instantánea**, en la AZ de la instancia.
    4. Sustituir el volumen: desmontar en el sistema operativo, **Desasociar volumen** el antiguo, **Asociar volumen** el nuevo y montarlo.

    Atajo: en el paso 3 también puede usarse directamente la instantánea **sin cifrar** marcando **Cifrar este volumen** al crear el volumen.

=== "AWS CLI"

    ```bash
    aws ec2 create-snapshot --volume-id vol-sincifrar --description "Para cifrar"
    aws ec2 copy-snapshot --source-region us-east-1 --source-snapshot-id snap-sincifrar \
      --encrypted --kms-key-id alias/ud3-almacenamiento --description "Cifrada"
    aws ec2 create-volume --snapshot-id snap-cifrada --availability-zone us-east-1a --volume-type gp3
    ```

---

## Cifrado en EFS

| Aspecto | En reposo | En tránsito |
| --- | --- | --- |
| **Mecanismo** | AES-256 con claves de AWS KMS | TLS 1.2 entre el cliente y el destino de montaje |
| **Cuándo se decide** | Al **crear** el sistema de archivos. **No se puede cambiar** | Al **montar** (`-o tls`) |
| **Cómo se exige** | Política de IAM con la condición `elasticfilesystem:Encrypted`, regla de AWS Config | Política del sistema de archivos con `aws:SecureTransport` |
| **Claves** | `aws/elasticfilesystem` o clave del cliente | Certificados gestionados por AWS |

Para impedir que nadie cree sistemas de archivos sin cifrar, puede añadirse a las políticas de IAM de los administradores (o a una SCP de la organización) una sentencia que deniegue `elasticfilesystem:CreateFileSystem` cuando `elasticfilesystem:Encrypted` sea `false`.

---

## Claves administradas por AWS o por el cliente

| Aspecto | Clave administrada por AWS (`aws/ebs`, `aws/elasticfilesystem`) | Clave administrada por el cliente |
| --- | --- | --- |
| Creación | Automática | El cliente la crea en KMS |
| Política de la clave | Fija, no editable | Editable: quién la usa y quién la administra |
| Rotación | Automática (anual) | Configurable |
| Compartir instantáneas con otra cuenta | **No** | **Sí** (concediendo uso de la clave) |
| Revocación | No | Deshabilitarla deja **inaccesibles** los datos cifrados con ella |
| Auditoría | CloudTrail | CloudTrail, con control detallado |
| Coste | Sin coste mensual | Coste mensual por clave + peticiones |

!!! danger "Perder la clave es perder los datos"
    Si se deshabilita o se elimina una clave administrada por el cliente, **todos los volúmenes, instantáneas y sistemas de archivos cifrados con ella quedan inaccesibles**. KMS no borra las claves inmediatamente: se programa la eliminación con un periodo de espera de **7 a 30 días**, durante el cual puede cancelarse.

---

## Plantilla de ejemplo: EBS y EFS cifrados

La siguiente plantilla de **AWS CloudFormation** crea en una sola pila:

1. Una **clave de KMS** propia con rotación automática y su alias.
2. Un **volumen EBS** `gp3` cifrado con esa clave, con `DeletionPolicy: Snapshot` y, opcionalmente, adjuntado a una instancia.
3. Un **grupo de seguridad** que solo admite NFS desde el grupo de seguridad de los clientes.
4. Un **sistema de archivos EFS** cifrado con la misma clave, Regional, Elastic, con **ciclo de vida** (IA a 30 días, Archive a 90, vuelta al primer acceso), **copias automáticas** y una **política** que concede lectura/escritura solo al rol de los clientes, impide el acceso *root* y exige TLS.
5. Dos **destinos de montaje** en subredes de zonas distintas.
6. Un **punto de acceso** para la aplicación web, con directorio `/web` propiedad de `apache` (UID/GID 48).

Descarga: [almacenamiento-cifrado.yaml](files/almacenamiento-cifrado.yaml)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: >
  UD3 - Volumen EBS y sistema de archivos EFS cifrados con una clave propia de KMS,
  con ciclo de vida, copias automaticas, politica de acceso y punto de acceso.

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC de las instancias cliente
  SubredA:
    Type: AWS::EC2::Subnet::Id
    Description: Subred (preferiblemente privada) en la primera AZ
  SubredB:
    Type: AWS::EC2::Subnet::Id
    Description: Subred (preferiblemente privada) en la segunda AZ
  GrupoSeguridadClientes:
    Type: AWS::EC2::SecurityGroup::Id
    Description: Grupo de seguridad de las instancias que montaran EFS
  ZonaVolumen:
    Type: AWS::EC2::AvailabilityZone::Name
    Description: AZ del volumen EBS (debe coincidir con la de la instancia)
  InstanciaId:
    Type: String
    Default: ""
    Description: (Opcional) instancia a la que adjuntar el volumen EBS
  RolClientes:
    Type: String
    Default: LabRole
    Description: Rol IAM de las instancias cliente (LabRole en el Learner Lab)

Conditions:
  AdjuntarVolumen: !Not [!Equals [!Ref InstanciaId, ""]]

Resources:
  # ---------------------------------------------------------------- Cifrado
  ClaveAlmacenamiento:
    Type: AWS::KMS::Key
    Properties:
      Description: Clave para EBS y EFS de la UD3
      EnableKeyRotation: true
      PendingWindowInDays: 7
      KeyPolicy:
        Version: "2012-10-17"
        Statement:
          - Sid: AdministracionPorLaCuenta
            Effect: Allow
            Principal:
              AWS: !Sub arn:aws:iam::${AWS::AccountId}:root
            Action: kms:*
            Resource: "*"

  AliasClaveAlmacenamiento:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: alias/ud3-almacenamiento
      TargetKeyId: !Ref ClaveAlmacenamiento

  # ---------------------------------------------------------------- EBS
  VolumenDatos:
    Type: AWS::EC2::Volume
    DeletionPolicy: Snapshot          # al borrar la pila se crea una instantanea final
    UpdateReplacePolicy: Snapshot     # si un cambio sustituye el volumen, tambien
    Properties:
      AvailabilityZone: !Ref ZonaVolumen
      Size: 10
      VolumeType: gp3
      Iops: 3000
      Throughput: 125
      Encrypted: true
      KmsKeyId: !GetAtt ClaveAlmacenamiento.Arn
      Tags:
        - { Key: Name, Value: ud3-datos }
        - { Key: backup, Value: diario }

  AdjuntoVolumen:
    Type: AWS::EC2::VolumeAttachment
    Condition: AdjuntarVolumen
    Properties:
      InstanceId: !Ref InstanciaId
      VolumeId: !Ref VolumenDatos
      Device: /dev/sdf

  # ---------------------------------------------------------------- EFS: red
  GrupoSeguridadEfs:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: NFS solo desde las instancias cliente
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 2049
          ToPort: 2049
          SourceSecurityGroupId: !Ref GrupoSeguridadClientes
      Tags:
        - { Key: Name, Value: efs-sg }

  # ---------------------------------------------------------------- EFS: sistema de archivos
  SistemaArchivos:
    Type: AWS::EFS::FileSystem
    DeletionPolicy: Retain            # EFS no admite Snapshot: se conserva al borrar la pila
    UpdateReplacePolicy: Retain
    Properties:
      Encrypted: true                 # no se puede cambiar despues
      KmsKeyId: !GetAtt ClaveAlmacenamiento.Arn
      PerformanceMode: generalPurpose
      ThroughputMode: elastic
      LifecyclePolicies:
        - TransitionToIA: AFTER_30_DAYS
        - TransitionToArchive: AFTER_90_DAYS
        - TransitionToPrimaryStorageClass: AFTER_1_ACCESS
      BackupPolicy:
        Status: ENABLED
      FileSystemPolicy:
        Version: "2012-10-17"
        Statement:
          - Sid: LecturaEscrituraClientes
            Effect: Allow
            Principal:
              AWS: !Sub arn:aws:iam::${AWS::AccountId}:role/${RolClientes}
            Action:
              - elasticfilesystem:ClientMount
              - elasticfilesystem:ClientWrite
            Condition:
              Bool:
                elasticfilesystem:AccessedViaMountTarget: "true"
          - Sid: ForzarRootSquash             # Deny explicito: LabRole concede ClientRootAccess por IAM
            Effect: Deny
            Principal:
              AWS: "*"
            Action: elasticfilesystem:ClientRootAccess
          - Sid: ExigirTLS
            Effect: Deny
            Principal:
              AWS: "*"
            Action: "*"
            Condition:
              Bool:
                aws:SecureTransport: "false"
      FileSystemTags:
        - { Key: Name, Value: ud3-efs-web }
        - { Key: backup, Value: diario }

  DestinoMontajeA:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref SistemaArchivos
      SubnetId: !Ref SubredA
      SecurityGroups: [!Ref GrupoSeguridadEfs]

  DestinoMontajeB:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref SistemaArchivos
      SubnetId: !Ref SubredB
      SecurityGroups: [!Ref GrupoSeguridadEfs]

  PuntoAccesoWeb:
    Type: AWS::EFS::AccessPoint
    Properties:
      FileSystemId: !Ref SistemaArchivos
      PosixUser:
        Uid: "48"                     # usuario apache en Amazon Linux
        Gid: "48"
      RootDirectory:
        Path: /web
        CreationInfo:
          OwnerUid: "48"
          OwnerGid: "48"
          Permissions: "0755"
      AccessPointTags:
        - { Key: Name, Value: ap-web }

Outputs:
  VolumenId:
    Description: Volumen EBS cifrado
    Value: !Ref VolumenDatos
  SistemaArchivosId:
    Description: Sistema de archivos EFS
    Value: !Ref SistemaArchivos
  PuntoAccesoId:
    Description: Punto de acceso de la aplicacion web
    Value: !Ref PuntoAccesoWeb
  ComandoMontaje:
    Description: Montaje desde una instancia cliente
    Value: !Sub "sudo mount -t efs -o tls,iam,accesspoint=${PuntoAccesoWeb} ${SistemaArchivos}:/ /var/www/html/uploads"
  ClaveArn:
    Description: Clave KMS utilizada
    Value: !GetAtt ClaveAlmacenamiento.Arn
```

Algunos detalles de la plantilla:

- **`DeletionPolicy`**: el volumen EBS usa `Snapshot`, de modo que al borrar la pila se crea una instantánea final. EFS no admite `Snapshot`, por eso usa `Retain`: el sistema de archivos se conserva y debe borrarse a mano.
- **Propiedades que sustituyen el recurso**: cambiar en la plantilla `Encrypted`, `KmsKeyId` o `PerformanceMode` de EFS, o la AZ del volumen, obliga a CloudFormation a **crear un recurso nuevo y eliminar el antiguo**. Antes de aplicar un cambio conviene revisar el **conjunto de cambios**.
- **`Deny` de `ClientRootAccess`**: el *root* de los clientes no tiene privilegios en EFS. No basta con no concederlo, porque la política de IAM de `LabRole` ya lo concede. La aplicación escribe a través del **punto de acceso**, que crea `/web` con el propietario correcto.
- **Cambios del ciclo de vida**: modificar el tamaño del volumen, su tipo o la política de ciclo de vida de EFS en la plantilla y volver a desplegarla aplica el cambio **sin sustituir** los recursos.

### Despliegue de la pila

=== "Consola AWS"

    **Crear la pila**

    1. **CloudFormation → Pilas → Crear pila → Con recursos nuevos (estándar)**.
    2. **Elegir una plantilla existente → Cargar un archivo de plantilla** y seleccionar `almacenamiento-cifrado.yaml`. La consola valida la sintaxis al cargarla. **Siguiente**.
    3. **Nombre de la pila**: `ud3-almacenamiento`. Rellenar los **parámetros** (VPC, subredes, grupo de seguridad de los clientes, AZ del volumen, instancia opcional). **Siguiente**.
    4. **Opciones de la pila**: etiquetas y, en el Learner Lab, el rol `LabRole` si se solicita. **Siguiente**.
    5. Revisar y pulsar **Enviar**. La pestaña **Eventos** muestra el progreso hasta `CREATE_COMPLETE`.
    6. La pestaña **Salidas** muestra los ID y el comando de montaje.

    **Modificar con un conjunto de cambios**

    1. Seleccionar la pila → **Conjuntos de cambios → Crear conjunto de cambios** (o **Actualizar** → **Reemplazar la plantilla existente**).
    2. Cargar la plantilla modificada, revisar los parámetros y crear el conjunto de cambios.
    3. Revisar la columna **Sustitución** (*Replacement*): si un recurso indica **True**, se creará uno nuevo y se eliminará el antiguo.
    4. **Ejecutar conjunto de cambios** si el resultado es el esperado.

    **Eliminar la pila**: seleccionarla → **Eliminar**. El volumen deja una instantánea final y el sistema EFS se conserva (`DeletionPolicy`).

=== "AWS CLI"

    ```bash
    aws cloudformation validate-template --template-body file://almacenamiento-cifrado.yaml

    aws cloudformation deploy \
      --stack-name ud3-almacenamiento \
      --template-file almacenamiento-cifrado.yaml \
      --parameter-overrides \
          VpcId=vpc-0abc... SubredA=subnet-0aaa... SubredB=subnet-0bbb... \
          GrupoSeguridadClientes=sg-0web... ZonaVolumen=us-east-1a \
          InstanciaId=i-0123456789abcdef0

    aws cloudformation describe-stacks --stack-name ud3-almacenamiento \
      --query "Stacks[0].Outputs" --output table

    # Ver qué cambiaría una modificación antes de aplicarla
    aws cloudformation deploy --stack-name ud3-almacenamiento \
      --template-file almacenamiento-cifrado.yaml --no-execute-changeset

    aws cloudformation delete-stack --stack-name ud3-almacenamiento
    ```


!!! note "Learner Lab"
    En el AWS Academy Learner Lab el rol de las instancias es `LabRole` (perfil `LabInstanceProfile`), que es el valor por defecto del parámetro `RolClientes`. Si alguna operación devuelve `AccessDenied` por las restricciones del laboratorio, se puede eliminar de la plantilla el recurso afectado (por ejemplo, la clave propia, usando entonces las claves administradas por AWS).

---

## Equivalente en Terraform

```hcl
resource "aws_kms_key" "almacenamiento" {
  description             = "Clave para EBS y EFS de la UD3"
  enable_key_rotation     = true
  deletion_window_in_days = 7
}

resource "aws_ebs_volume" "datos" {
  availability_zone = "us-east-1a"
  size              = 10
  type              = "gp3"
  encrypted         = true
  kms_key_id        = aws_kms_key.almacenamiento.arn
  tags              = { Name = "ud3-datos", backup = "diario" }
}

resource "aws_efs_file_system" "web" {
  encrypted        = true
  kms_key_id       = aws_kms_key.almacenamiento.arn
  performance_mode = "generalPurpose"
  throughput_mode  = "elastic"

  lifecycle_policy { transition_to_ia                    = "AFTER_30_DAYS" }
  lifecycle_policy { transition_to_archive               = "AFTER_90_DAYS" }
  lifecycle_policy { transition_to_primary_storage_class = "AFTER_1_ACCESS" }

  tags = { Name = "ud3-efs-web" }
}

resource "aws_efs_backup_policy" "web" {
  file_system_id = aws_efs_file_system.web.id
  backup_policy { status = "ENABLED" }
}

resource "aws_efs_mount_target" "a" {
  file_system_id  = aws_efs_file_system.web.id
  subnet_id       = var.subred_a
  security_groups = [aws_security_group.efs.id]
}
# ... aws_efs_mount_target "b", aws_efs_file_system_policy, aws_efs_access_point
```

---

## Referencias

- AWS. *Amazon EBS encryption* — [Cifrado de EBS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html)
- AWS. *Enable Amazon EBS encryption by default* — [Cifrado por defecto](https://docs.aws.amazon.com/ebs/latest/userguide/encryption-by-default.html)
- AWS. *Data encryption in Amazon EFS* — [Cifrado de EFS](https://docs.aws.amazon.com/efs/latest/ug/encryption.html)
- AWS. *Enforcing encryption at rest with IAM* — [Exigir el cifrado al crear](https://docs.aws.amazon.com/efs/latest/ug/using-iam-to-enforce-encryption-at-rest.html)
- AWS CloudFormation. *AWS::EC2::Volume* — [Referencia](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-volume.html)
- AWS CloudFormation. *AWS::EFS::FileSystem* — [Referencia](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-efs-filesystem.html)
- AWS CloudFormation. *AWS::EFS::AccessPoint* — [Referencia](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-efs-accesspoint.html)
- HashiCorp. *aws_efs_file_system* — [Proveedor AWS de Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_file_system)
