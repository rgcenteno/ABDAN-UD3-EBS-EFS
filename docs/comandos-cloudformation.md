# Comandos básicos CloudFormation

Podemos realizar las operaciones que necesitemos con una plantilla CloudFormation mediante CloudShell.

1. Inicia sesión en la consola de AWS.

2. En la esquina inferior izquierda pulsa sobre CloudShell

3. Una vez desplegada la consola, puedes subir el fichero yaml pulsando en esquina superior derecha el botón Acciones > Cargar archivo

El yaml estará en la carpeta actual. Puedes usar un comando `ls` para comprobar que efectivamente se ha subido.


!!! info
    
    Antes de revisar los comandos recuerda que debes cambiar los siguientes valores:

    - `vpc-abdan.yaml` por el nombre de tu fichero

    - `us-east-1` por la región de despliegue

    - `$stackname` por el stack creado o por el nombre que quieras dar a tu stack.

## 1. Validar la plantilla

```sh
aws cloudformation validate-template \
    --template-body file://vpc-abdan.yaml \
    --region us-east-1
```

## 2. Desplegar (crea el stack o lo actualiza si ya existe)

```sh
aws cloudformation deploy \
    --template-file vpc-abdan.yaml \
    --stack-name $stackname \
    --region us-east-1
```

## 2b. Desplegar cambiando algún parámetro

```sh
aws cloudformation deploy \
    --template-file vpc-abdan.yaml \
    --stack-name $stackname \
    --parameter-overrides VpcName=vpc-prueba VpcCidr=10.1.0.0/16 \
    --region us-east-1
```

## 3. Ver las salidas (IDs de los recursos)

```sh
aws cloudformation describe-stacks \
    --stack-name $stackname \
    --query 'Stacks[0].Outputs' \
    --output table \
    --region us-east-1
```

## 4. Si algo falla, ver los eventos

```sh
aws cloudformation describe-stack-events \
    --stack-name $stackname \
    --query 'StackEvents[?contains(ResourceStatus,`FAILED`)].[LogicalResourceId,ResourceStatusReason]' \
    --output table \
    --region us-east-1
```

## 5. Borrar todo (NAT y EIP incluidos)

```sh
aws cloudformation delete-stack --stack-name $stackname --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name $stackname --region us-east-1
```
