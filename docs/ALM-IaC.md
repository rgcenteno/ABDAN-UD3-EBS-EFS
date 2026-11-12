# Cifrado e infraestructura como código

El criterio de evaluación pide configurar el **cifrado** mediante **infraestructura como código (IaC)**: describir los recursos en un fichero de texto, de modo que el cifrado quede fijado en la plantilla y no dependa de que alguien recuerde marcar una casilla.

---

## Cifrado en EBS y EFS

| Aspecto | Amazon EBS | Amazon EFS |
| --- | --- | --- |
| **En reposo** | AES-256 con AWS KMS. Protege el volumen, el tráfico con la instancia y las instantáneas | AES-256 con AWS KMS |
| **Cuándo se decide** | Al **crear** el volumen. Un volumen existente no se puede cifrar: se hace una instantánea, se copia cifrada y se crea un volumen nuevo | Al **crear** el sistema de archivos. **No se puede cambiar** después |
| **En tránsito** | Automático en los volúmenes cifrados | Opción `tls` al montar; se exige con la política del sistema de archivos |
| **Clave** | `aws/ebs` o una clave propia | `aws/elasticfilesystem` o una clave propia |

Con una **clave administrada por el cliente** se controla quién la usa, se configura su rotación y se pueden compartir instantáneas cifradas con otras cuentas. Si se deshabilita o se elimina, **los datos cifrados con ella quedan inaccesibles**.

### Cifrado por defecto en EBS

Es un ajuste **por región**: una vez activado, todos los volúmenes nuevos se cifran aunque nadie lo indique.

=== "Consola AWS"

    1. **EC2 → Panel**; en el lateral derecho, **Atributos de la cuenta → Protección y seguridad de datos**.
    2. En **Cifrado de EBS**, pulsar **Administrar**.
    3. Marcar **Habilitar** y elegir la **clave de cifrado predeterminada**.
    4. **Actualizar cifrado de EBS**. Hay que repetirlo en cada región que se utilice.

=== "AWS CLI"

    ```bash
    aws ec2 enable-ebs-encryption-by-default
    aws ec2 get-ebs-encryption-by-default
    ```

---

## Plantilla de CloudFormation

**AWS CloudFormation** es el servicio de IaC nativo de AWS. Una **plantilla** (YAML) describe los recursos y CloudFormation los crea juntos en una **pila**. La plantilla siguiente crea una clave de KMS propia y, cifrados con ella, un volumen EBS y un sistema de archivos EFS.

Descarga: [cifrado-basico.yaml](files/cifrado-basico.yaml)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: UD3 - Volumen EBS y sistema de archivos EFS cifrados con la misma clave de KMS

Parameters:
  ZonaVolumen:
    Type: AWS::EC2::AvailabilityZone::Name
    Description: AZ del volumen EBS (la misma que la de la instancia)

Resources:
  ClaveAlmacenamiento:              # clave propia, con rotacion automatica
    Type: AWS::KMS::Key
    Properties:
      Description: Clave para EBS y EFS de la UD3
      EnableKeyRotation: true
      KeyPolicy:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub arn:aws:iam::${AWS::AccountId}:root
            Action: kms:*
            Resource: "*"

  VolumenDatos:                     # volumen EBS cifrado
    Type: AWS::EC2::Volume
    DeletionPolicy: Snapshot        # al borrar la pila deja una instantanea
    Properties:
      AvailabilityZone: !Ref ZonaVolumen
      Size: 10
      VolumeType: gp3
      Encrypted: true
      KmsKeyId: !GetAtt ClaveAlmacenamiento.Arn
      Tags:
        - { Key: Name, Value: ud3-datos }

  SistemaArchivos:                  # EFS cifrado (no se puede cambiar despues)
    Type: AWS::EFS::FileSystem
    Properties:
      Encrypted: true
      KmsKeyId: !GetAtt ClaveAlmacenamiento.Arn
      ThroughputMode: elastic
      FileSystemTags:
        - { Key: Name, Value: ud3-efs }

Outputs:
  VolumenId:
    Value: !Ref VolumenDatos
  SistemaArchivosId:
    Value: !Ref SistemaArchivos
```

Las propiedades que fijan el cifrado son `Encrypted: true` y `KmsKeyId`. Cambiarlas después en la plantilla obliga a CloudFormation a **sustituir** el recurso (crear uno nuevo y borrar el antiguo), con la consiguiente pérdida de datos si no se ha previsto.

### Desplegar y eliminar la pila

=== "Consola AWS"

    1. **CloudFormation → Pilas → Crear pila → Con recursos nuevos (estándar)**.
    2. **Cargar un archivo de plantilla** y seleccionar `cifrado-basico.yaml`. **Siguiente**.
    3. **Nombre de la pila**: `ud3-cifrado`. Parámetro **ZonaVolumen**: la AZ de la instancia. **Siguiente** dos veces.
    4. **Enviar**. La pestaña **Eventos** muestra el progreso hasta `CREATE_COMPLETE`; la pestaña **Salidas**, los ID creados.
    5. Comprobar en **EC2 → Volúmenes** y **EFS → Sistemas de archivos** que ambos aparecen como **Cifrados** con la clave nueva.
    6. Para borrarlo todo: seleccionar la pila → **Eliminar**.

=== "AWS CLI"

    ```bash
    aws cloudformation deploy --stack-name ud3-cifrado \
      --template-file cifrado-basico.yaml \
      --parameter-overrides ZonaVolumen=us-east-1a

    aws cloudformation describe-stacks --stack-name ud3-cifrado \
      --query "Stacks[0].Outputs" --output table

    aws cloudformation delete-stack --stack-name ud3-cifrado
    ```

!!! info "Para saber más"
    La [ampliación de IaC](AMP-IaC.md) incluye una plantilla completa (destinos de montaje, grupo de seguridad, política de acceso, ciclo de vida, copias y punto de acceso) y su equivalente en Terraform.

---

## Referencias

- AWS. *Amazon EBS encryption* — [Cifrado de EBS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html)
- AWS. *Data encryption in Amazon EFS* — [Cifrado de EFS](https://docs.aws.amazon.com/efs/latest/ug/encryption.html)
- AWS CloudFormation. *AWS::EC2::Volume* — [Referencia](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-volume.html)
- AWS CloudFormation. *AWS::EFS::FileSystem* — [Referencia](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-efs-filesystem.html)
