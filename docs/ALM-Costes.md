# Selección del almacenamiento y costes

!!! info "Contenido de ampliación"
    Esta página es **opcional**. Desarrolla contenidos que en el núcleo de la unidad solo se mencionan. No es necesaria para realizar las prácticas.

En los apartados anteriores se han estudiado por separado los tipos de volúmenes EBS y las clases de EFS. Este apartado los reúne para aplicar el primer criterio de evaluación de la unidad: **escoger una clase de almacenamiento teniendo en cuenta requisitos funcionales, patrones de acceso, durabilidad, costes de almacenamiento y de recuperación y cualquier otra limitación**.

---

## Cómo factura cada servicio

| Concepto | Amazon EBS | Amazon EFS |
| --- | --- | --- |
| **Capacidad** | GiB **aprovisionados** al mes, se usen o no | GB **ocupados** al mes en cada clase |
| **Rendimiento** | IOPS y MiB/s por encima de la base (gp3) o IOPS aprovisionadas (io1/io2) | Elastic: GB leídos y escritos · Provisioned: MiB/s aprovisionados · Bursting: incluido |
| **Acceso a los datos** | Sin cargo | Cargo por GB leído en **IA** y **Archive** (coste de recuperación) |
| **Cambios de clase** | — | Cargo por las transiciones del ciclo de vida |
| **Copias** | Instantáneas: GB-mes de bloques almacenados (nivel estándar o archive) | AWS Backup: GB-mes de copias (cálido o frío) |
| **Transferencia** | Copias de instantáneas entre regiones | Tráfico entre AZ si se monta desde otra zona; replicación entre regiones |

### Precios orientativos

Los precios varían por región y cambian con el tiempo. La tabla recoge **órdenes de magnitud** en `us-east-1` para comparar opciones; para un presupuesto real debe utilizarse la [calculadora de precios de AWS](https://calculator.aws/).

| Recurso | Precio aproximado (GB-mes) |
| --- | --- |
| EBS `gp3` | 0,08 USD (+ IOPS > 3.000 y MiB/s > 125) |
| EBS `gp2` | 0,10 USD |
| EBS `io2` | 0,125 USD + cada IOPS aprovisionada |
| EBS `st1` | 0,045 USD |
| EBS `sc1` | 0,015 USD |
| Instantánea EBS (estándar / archive) | 0,05 USD / 0,0125 USD |
| EFS Standard (Regional) | 0,30 USD |
| EFS Infrequent Access | 0,016 USD (+ lectura por GB) |
| EFS Archive | 0,008 USD (+ lectura por GB) |
| EFS One Zone | Aproximadamente la mitad que la clase Regional equivalente |
| S3 Standard (referencia, UD2) | 0,023 USD |

!!! warning "EFS Standard no es «caro» ni EBS «barato» sin contexto"
    Por GB, EFS Standard cuesta casi cuatro veces más que `gp3`. Pero EBS cobra el **tamaño reservado**: un volumen de 100 GiB con 20 GiB ocupados cuesta lo mismo que si estuviera lleno. Y EBS no se comparte: cuatro servidores web con su propio volumen de contenidos pagan **cuatro** volúmenes. La comparación correcta se hace **para la carga concreta**.

---

## Ejemplos de cálculo

### Ejemplo 1: el efecto del ciclo de vida en EFS

Un sistema de archivos con **1.000 GB** en el que, tras unos meses, solo el **20 %** de los ficheros se consulta cada mes.

| Configuración | Cálculo | Coste mensual aproximado |
| --- | --- | --- |
| Todo en Standard (sin ciclo de vida) | 1.000 × 0,30 | **300 USD** |
| Ciclo de vida a IA | 200 × 0,30 + 800 × 0,016 | 60 + 12,8 = **72,8 USD** + lecturas de IA |
| Ciclo de vida a IA y Archive (500 GB fríos en Archive) | 200 × 0,30 + 300 × 0,016 + 500 × 0,008 | 60 + 4,8 + 4 = **68,8 USD** + lecturas |

Activar el ciclo de vida **reduce el coste de almacenamiento en más de un 75 %**. La única condición es que los datos fríos sean realmente fríos: si se leyeran con frecuencia, los cargos de lectura podrían anular el ahorro, y por eso existe la vuelta a Standard al primer acceso.

### Ejemplo 2: sobredimensionar gp2 frente a gp3

Una base de datos que ocupa **200 GiB** y necesita **6.000 IOPS** sostenidas.

| Opción | Configuración | Coste mensual aproximado |
| --- | --- | --- |
| `gp2` | 2.000 GiB (3 IOPS/GiB × 2.000 = 6.000) × 0,10 | **200 USD** |
| `gp3` | 200 GiB × 0,08 + 3.000 IOPS adicionales × 0,005 | 16 + 15 = **31 USD** |

`gp3` permite pagar **la capacidad que se usa y el rendimiento que se necesita** por separado.

### Ejemplo 3: contenidos compartidos de una web

Cuatro servidores web detrás de un balanceador comparten **50 GB** de imágenes.

| Opción | Problema / coste |
| --- | --- |
| Un volumen EBS por servidor, 100 GiB cada uno | 4 × 100 × 0,08 = 32 USD/mes, **pero los contenidos no se comparten**: hay que sincronizarlos |
| EBS Multi-Attach | Solo `io1`/`io2`, misma AZ y requiere un sistema de ficheros de clúster: **no aplicable** |
| **EFS Regional con ciclo de vida** | ~50 GB entre Standard e IA: menos de 15 USD/mes, **compartido, multi-AZ y elástico** |
| S3 + CloudFront | La opción más barata para servir contenido estático, pero exige adaptar la aplicación para usar la API |

---

## Parámetros que afectan al coste

| Servicio | Parámetro | Ajuste para reducir costes |
| --- | --- | --- |
| EBS | **Tipo de volumen** | Migrar `gp2` → `gp3`; usar `st1`/`sc1` para datos secuenciales o fríos |
| EBS | **Tamaño** | No sobredimensionar: se puede ampliar en caliente cuando haga falta |
| EBS | **IOPS y rendimiento** | Aprovisionar solo lo que demuestren las métricas (`VolumeIOPSExceededCheck`) |
| EBS | **Volúmenes huérfanos** | Eliminar los volúmenes `available` que no se utilicen (con instantánea final) |
| EBS | **Retención de instantáneas** | Políticas de DLM o AWS Backup que borren las antiguas; nivel archive para las de larga duración |
| EFS | **Tipo** | One Zone para desarrollo, pruebas y datos regenerables |
| EFS | **Ciclo de vida** | Transición a IA y Archive; vuelta al primer acceso si el patrón es incierto |
| EFS | **Modo de rendimiento** | Elastic para cargas variables; Provisioned solo si la carga es alta y constante |
| EFS | **Destinos de montaje** | Uno en cada AZ con clientes, para evitar tráfico entre zonas |
| Ambos | **Etiquetas** | Etiquetas de asignación de costes (`proyecto`, `entorno`) para saber quién gasta qué |

!!! tip "Herramientas de ayuda"
    - **AWS Compute Optimizer** analiza las métricas de los volúmenes EBS y recomienda el tipo, tamaño, IOPS y rendimiento adecuados.
    - **AWS Cost Explorer** muestra el gasto por servicio y por etiqueta.
    - **AWS Trusted Advisor** detecta, entre otros, volúmenes EBS infrautilizados.

---

## Árbol de decisión

1. **¿Necesita un único servidor un disco (SO, base de datos)?** → **EBS**.
    - ¿Carga transaccional estándar? → `gp3`.
    - ¿Base de datos crítica con latencia garantizada? → `io2`.
    - ¿Lectura secuencial masiva? → `st1`.
    - ¿Datos fríos con acceso secuencial ocasional? → `sc1`.
    - ¿Datos temporales que pueden perderse? → *instance store*.
2. **¿Necesitan varios servidores Linux los mismos ficheros?** → **EFS**.
    - ¿Producción? → Regional. ¿Desarrollo o datos regenerables? → One Zone.
    - ¿Muchos ficheros que dejan de usarse? → Ciclo de vida a IA y Archive.
    - ¿Carga desconocida o variable? → Elastic.
3. **¿Servidores Windows con carpetas compartidas?** → **FSx for Windows File Server**.
4. **¿Objetos accesibles por API, copias de seguridad, web estática, archivo a largo plazo?** → **S3** (UD2).

---

## Referencias

- AWS. *Amazon EBS pricing* — [Precios de EBS](https://aws.amazon.com/ebs/pricing/)
- AWS. *Amazon EFS pricing* — [Precios de EFS](https://aws.amazon.com/efs/pricing/)
- AWS. *AWS Pricing Calculator* — [Calculadora de precios](https://calculator.aws/)
- AWS. *Migrate your Amazon EBS volumes from gp2 to gp3 and save up to 20% on costs* — [Artículo](https://aws.amazon.com/blogs/storage/migrate-your-amazon-ebs-volumes-from-gp2-to-gp3-and-save-up-to-20-on-costs/)
- AWS. *AWS Compute Optimizer: EBS volume recommendations* — [Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-ebs-recommendations.html)
