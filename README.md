# PRUEBA LÍDER CENTRO DE EXCELENCIA

## Índice
* [Enlaces y Credenciales](#enlaces-y-credenciales)
* [Repositorios](#repositorios)
* [Arquitectura de la Solución](#requisitos)
* [Servicios Desplegados](#procedimiento)
* [Pipelines](#pipelines)
* [Procedimiento Lanzar Infraestructura y Servicios](#procedimiento)
* [Pruebas Consumo Servicio](#procedimiento)
* [Autor](#autor)


## Enlaces y Credenciales

### 1. Github

https://github.com/Lider-Bancolomiba
```
Usuario:liderbancolombia85@gmail.com
Contraseña:Bancolombia85$
```

### 2. Consola AWS

https://us-east-1.console.aws.amazon.com/console/home?region=us-east-1
```
Usuario Raiz:liderbancolombia85@gmail.com
Contraseña:Bancolombia85$
```

### 3. SonarCloud

https://sonarcloud.io/login
```
Por medio de SSO con Github
```


## Repositorios
Los repositorios se encuentran alojados en Github en la Organización Lider-Bancolomiba https://github.com/orgs/Lider-Bancolomiba/repositories, para descargar de manera local ejecutar los siguientes comandos:


### 1. Documentación [bancolombia-prueba-doc-solucion]
```
git clone https://github.com/Lider-Bancolomiba/bancolombia-prueba-doc-solucion.git
```

### 2. Infraestructura [bancolombia-prueba-infra]
```
git clone https://github.com/Lider-Bancolomiba/bancolombia-prueba-infra.git
```

### 3. Servicios [bancolombia-prueba-ms-usuarios]
```
git clone https://github.com/Lider-Bancolomiba/bancolombia-prueba-ms-usuarios.git
```

## Arquitectura de la Solución
Con base en el reto planteado se ha diseñado la siguiente arquitectura:
![Arquitectura](/images/ArquitecturaMicroserviciosAWS.png)

Los componentes y flujo de trabajo se describe a continuación:

#### 1. Versionamiento Código Fuente 
El flujo de trabajo inicia con el push realizado al repositorio remoto por parte del desarrollador, en este caso se usa para alojar el código fuente el servicio de repositorio que nos brinda Github.

#### 2. Ejecución de Pipelines
Una vez se realice el push a los repositorios, dependiendo al cual se realice la actualización, se ejecutaran los pipelines desarrollados en GitHub Actions, desplegando la infraestructura o servicios de la solución, quienes se van a comunicar con los demás servicios del stack usados y realizar todas las tareas requeridas para hacer el build, pruebas y despliegue de los servicios.

#### 3. Análisis de Código Estático
Posterior a realizar el checkout del código fuente por medio del servicio SonarQube Cloud, se ejecuta el análisis estático de código y la identificación de la coberura de pruebas generado en la pruebas unitarias.

#### 4. Despliegue Infraestructura
En caso tal de ejecutar el pipeline de aprovisionamiento de infraestructura, se usa para fin Terraform, quien va ejecutar las tareas necesarias para inicializar, validar, generar un plan y aplicar los cambios de Terraform a la infraestructura de AWS.

#### 5. Pruebas de Seguridad
Este es un componente propuesto, que no se tiene en el actual despliegue, se añade como componente propuesto "Aikido" con el objetivo de poder integrar prácticas DevSecOps, permitiendo adoptar prácticas como:

- Análisis Dinámico de Código
- Detección de Secretos
- Análisis de Dependencias
- Análisis de Imagenes
- Análisis de IaC

#### 6. Almacenamiento Estado Infraestructura
El pipeline de infraestructura con el objetivo de tener un despliegue de infraestructrura idempotente aloja el archivo del estado de en un bucket de s3. Que para el presente caso es el archivo terraform-core.tfstate.

#### 7. Asignación de Permisos
Para asegurarse una correcta ejecución del despliegue de los servicios sobre AWS ECS y se tengan los permisos necesarios, el flujo de infraestructura define tres políticas de AWS IAM que otorgan permisos específicos a un rol:

- Interactuar con ECR para obtener imágenes de contenedores
- Enviar logs a CloudWatch Logs
- Realizar operaciones en una tabla de DynamoDB

#### 8. Montaje de la Imagen
Para almacenamuiento, gestión y despliegue de las imagenes generadas en el paso de build de acuerdo al Dockerfile, se usa el servicios AWS ECR, quien nos sirve como contenedor y versionador de la imagen cada vez que es ejecutado el pipéline

#### 9. Comunicación de los servicios
Con le objetivo de permitir comunicaciones de los servicios y tener una adecuada configuración de red, se usa el servicio AWS VPC, el uso de este servicio nos va permitir la creación de un red privada que permitar tener un aislamiento de red, creación de subred para identificar el segmento que va a ser asignado a los diferentes servicios y una tablas de enrutamiento que permitan dirigir el trafico dentro de la VPC.

#### 10. Creación de Contenedores
Con el objetivo de permitir la orquestación de contenedores, gestionar y escalar aplicaciones contenedorizadas para el montaje de los microservicios se usa en la presente arquitectura AWS ECS. El mismo despliega servicios que alojan contendores permitiendo el despliegue de nuestros definidos, una vez se tiene la imagen versionada se realiza la creación de los servicios sobre ECS quien cuenta con un Cluster main-ecs-cluster, quien monta el servicio usuarios-service, con el contenedor que despliega la imagen de Docker definida.

#### 11. Almacenamiento de Datos
Como complemento al servicio que hemos desplegado en ECS, usamos el servicio de AWS DynamoDB, este servicio de base de datos NoSQL que nos brinda una gran versatilidad de funcionles y nos va a permitir obtener y almacenar toda la data requerida para el correcto funcionamiento de los microservicios.

#### 12. Monitoreo de Servicios
Para gestionar la data del funcionamiento de los sevicios y crear alertas que permitan en tiempo real monitorear el correcto funcionamiento de los servicios se ha usado el servicio AWS Cloud Watch, para el caso de uso actual se han creado 3 alarmas que permiten validar la disponibilidad y rendimiento de los servicios montados

- Respuesta del Servicio, envía una notificación cuando el servicio tarda más de 3 segundos en dar respuesta.
- Porcentaje de Uso de CPU, envía una notificación cuando el servicio tiene un porcentaje de uso de CPU superior a 80.
- Porcentaje de Uso de Memoría, envía una notificación cuando el servicio tiene un porcentaje de uso de Memoria superior a 80.

Sin embargo como recomendación se menciona el uso Datadog o Dynatrace, ya que estas aplicaciones son muy robustas con grandes prestaciones para el monitoreo y la observabilidad, adicionalmente nos va permitir isntrumentar los servicios dentro de los contenedores,  entregando metricas especificas del comportamiento de los contenedores y entregaqndo información a nivel de incidenetes sobre del código fuente.

#### 13. Balanceo de CArga
Amazon Elastic Load Balancer (ALB) es un servicio de balanceo de carga que distribuye el tráfico entrante a múltiples destinos, en nuestro caso, los contenedores en ECS que hemos desplegado, en este caso usamos ALC para acceder y consumir los servicios desplegados en el AWS ECS.


## Autor

* **Christian Camilo Gil López** 