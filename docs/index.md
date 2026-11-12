Bienvenid@ a la documentación de la Unidad Didáctica 3 del módulo: Administración de bases de datos y almacenamiento en la nube

# Administración de almacenamiento de bloques y ficheros

En esta unidad se estudian los dos servicios de AWS que proporcionan almacenamiento **montable** a las máquinas virtuales:

- **Amazon Elastic Block Store (EBS)**: almacenamiento en **bloques**, el equivalente a un disco conectado a una instancia EC2.
- **Amazon Elastic File System (EFS)**: almacenamiento de **ficheros** compartido mediante NFS, el equivalente a un servidor de ficheros o una cabina NAS.

En la unidad anterior se trabajó con el almacenamiento de **objetos** (Amazon S3). Con esta unidad se completan los tres modelos de almacenamiento que ofrece la nube y se adquieren los criterios para elegir entre ellos.

---

## Resultado de aprendizaje

**RA3.** Administra sistemas de almacenamiento de ficheros en nube, tanto en dispositivos de bloque como en sistemas de almacenamiento en red, utilizando tanto herramientas gráficas como de línea de comandos, API y/o IaC, garantizando el almacenamiento, la ciberseguridad y los patrones de uso que mejor se ajusten a los requisitos.

### Criterios de evaluación y dónde se trabajan

| Criterio | Contenido | Apartados |
| --- | --- | --- |
| **a)** Clase de almacenamiento según requisitos funcionales, patrones de acceso, durabilidad, costes de almacenamiento y de recuperación | Tipos de volúmenes EBS, clases de EFS, comparativa EBS/EFS/S3 | [Bloques, ficheros y objetos](Almacenamiento-Introduccion.md), [Tipos de volúmenes](EBS-02.md), [Clases de almacenamiento](EFS-04.md), [Costes](ALM-Costes.md) |
| **b)** Región geográfica y replicación entre zonas o regiones | Ámbito de AZ de EBS, instantáneas regionales, EFS Regional y One Zone, replicación | [Instantáneas](EBS-03.md), [Arquitectura de EFS](EFS-02.md), [Replicación EFS](EFS-08.md) |
| **c)** Cifrado mediante IaC | AWS KMS, cifrado por defecto, plantillas de CloudFormation y Terraform | [Cifrado e IaC](ALM-IaC.md) |
| **d)** Visibilidad, acceso, seguridad, monitorización, observabilidad y auditoría | Grupos de seguridad, IAM, políticas de EFS, CloudWatch, CloudTrail, AWS Config | [Políticas de EFS](EFS-06.md), [Seguridad y monitorización](ALM-Supervision.md) |
| **e)** Montaje garantizando el acceso desde los orígenes y la modalidad (solo lectura o lectura/escritura) | Montaje de volúmenes EBS y de EFS, `/etc/fstab`, puntos de acceso | [Ciclo de vida EBS](EBS-05.md), [Montaje de EFS](EFS-07.md) |
| **f)** Cambios durante el ciclo de vida: tamaño, clase, configuración, desmontaje y borrado | Elastic Volumes, ampliación del sistema de ficheros, ciclo de vida de EFS, eliminación | [Ciclo de vida EBS](EBS-05.md), [Ciclo de vida EFS](EFS-04.md), [Creación de EFS](EFS-05.md) |
| **g)** Replicación y copia de seguridad recuperables en la forma y el tiempo especificados | Instantáneas, Data Lifecycle Manager, AWS Backup, EFS Replication, RPO y RTO | [Instantáneas](EBS-03.md), [Copias y replicación EFS](EFS-08.md), [Copias y recuperación](ALM-Recuperacion.md) |

---

## Material audiovisual

Los apartados siguen el mismo orden que las sesiones grabadas:

| Sesión | Apartados relacionados |
| --- | --- |
| Amazon EBS e instantáneas | Bloque **Amazon EBS** |
| Amazon EFS: características, casos de uso y ciclo de vida | Apartados 1 a 4 del bloque **Amazon EFS** y copias de seguridad |
| Creación de un sistema de archivos EFS (consola) | [Creación de un sistema de archivos](EFS-05.md) |
| Políticas del sistema de archivos EFS | [Políticas del sistema de archivos y permisos](EFS-06.md) |

!!! tip "Consola AWS y AWS CLI"
    Las operaciones sobre los recursos de AWS se muestran en dos pestañas: **Consola AWS**, con los pasos en la consola web, y **AWS CLI**, con los comandos equivalentes. Al elegir una pestaña, todas las del sitio cambian a la misma opción. Los nombres de menús y botones corresponden a la consola en **español** (entre paréntesis se indica a veces el nombre en inglés) y pueden variar ligeramente cuando AWS actualiza la interfaz.

    Los comandos que se ejecutan **dentro de la instancia** (`lsblk`, `mkfs`, `mount`, `/etc/fstab`…) no tienen equivalente en la consola y se muestran sin pestañas. La [chuleta de comandos](Chuleta.md) reúne únicamente la versión de línea de comandos.

!!! note "Región de los ejemplos"
    Los diagramas de las presentaciones utilizan las regiones de **España** (`eu-south-2`) e **Irlanda** (`eu-west-1`) para ilustrar la elección de región y la recuperación ante desastres. Los comandos de los laboratorios utilizan **`us-east-1`**, la región del AWS Academy Learner Lab.
