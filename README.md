# PRUEBA LÍDER CENTRO DE EXCELENCIA

## Índice
* [Enlaces y Credenciales](#enlaces-y-credenciales)
* [Repositorios](#repositorios)
* [Arquitectura de la Solución](#arquitectura-de-la-solución)
* [Servicios Desplegados](#servicios-desplegados)
* [Pipelines](#pipelines)
* [Procedimiento Lanzar Infraestructura y Servicios](#procedimiento-lanzar-infraestructura-y-servicios)
* [Pruebas Consumo Servicio](#pruebas-consumo-servicio)
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



## Servicios Desplegados

## 1. Microservicio Python Usuarios

Servicio Web de python construido con el framework FastAPI que pertmite definir los Endpoints, el cual interactua con una tabla en DynamoDB del cual consulta e inserta información, para este servicio se crearon 4 operaciones:

- /healt (get) - Permite identificar si el servicio se encuentra operativo.
- /user (get) - Permite obtener la información de todos los usuarios
- /users/{user_id} (get) - Permite obtener la información de un usuario por medio delk id.
- /users (post) - Permite la creación de un nuevo usuario.


### 2. BD Dynamo DB

Servicio de base datos que contiene toda la informacón gestionada por medio del servicio usuarios, el cual tiene la siguiente definición:

| Campo    | Tipo     |
|----------|----------|
| user_id  | str      |
| email    | str      |
| Row 3    | str      |




## Pipelines

### 1. Despliegue Servicios

El pipeline de despliegue de servicios realiza las siguientes tareas:

1. Checkout code
Clona el repositorio en el entorno de ejecución de Github Actions
2. Set up AWS credentials
Configura las credenciales de AWS para permitir la interacción con los servicios de AWS.
3. Install dependencies
Instala las dependencias requeridas para realizar las pruebas unitarias
4. Run unit tests
Ejecuta las pruebas unitarias y genera un reporte de cobertura de código.
5. SonarCloud Scan
Realiza un análisis de calidad de código y cobertura utilizando SonarCloud
6. Set up Docker Buildx
Configura Docker Buildx para construir imágenes de Docker
7. Build Docker image
Construye la imagen Docker de la aplicación de acuerdo con el Dockefile
8. Login to Amazon ECR
Se autentica en AWS ECR
9. Tag Docker image
Etiqueta la imagen Docker para su despliegue
10. Push Docker image to ECR
Sube la imagen Docker etiquetada a AWS ECR
11. Update ECS service
Realiza el despliegue o actualización del servicio ECS para desplegar la nueva versión de la imagen Docker en AWS ECS
12. Verify ECS Service Status
Espera a que el servicio se encuentre operativo.


### 2. Aprovisionamiento Infraestructrua

El pipeline de despliegue de infraestructura realiza las siguientes tareas:

1. Check out code
clona el repositorio en entorno de ejecución Github Actions
2. WS Credentials
Configura las credenciales de AWS para permitir la interacción con los servicios de AWS
3. Set up Terraform
Configura Terraform en el entorno de ejecución Github Actions
4. Terraform Init
Inicializa Terraform, descargando los módulos y proveedores necesarios
5. Terraform Validate
Valida la configuración de Terraform para asegurarse de que no contiene errores de sintaxis ni de configuración.
6. Terraform Plan
Genera y muestra un plan de ejecución de Terraform
7. Terraform Apply
Aplica los cambios planificados a la infraestructura de AWS



## Procedimiento Lanzar Infraestructura y Servicios

### 1. Aprovisionamiento Infraestructura
1. Ingresar al enlace https://github.com/Lider-Bancolomiba/bancolombia-prueba-infra/actions/runs/13022758955/workflow
2. Dar Clic en el boton Re-run all jobs

### 2. Despliegue Servicios
1. Ingresar al enlace https://github.com/Lider-Bancolomiba/bancolombia-prueba-ms-usuarios/actions/runs/13032248250/workflow
2. Dar Clic en el boton Re-run all jobs


## Pruebas Consumo Servicio

El Enspoint donde se ha despledo el servicio es http://ecs-alb-387103903.us-east-1.elb.amazonaws.com/ y ser realizan pruebas por medio de https://reqbin.com/post-online

1. Identificar Salud del Servicio

![SaludServicios](/images/SaludServicio.png)

2. Obtener la Información de Usuarios

![ObtenerUsuarios](/images/ObtenerUsuarios.png)

3. Crear Usuarios
![CrearUsuarios](/images/CrearUsuarios.png)

## Autor

* **Christian Camilo Gil López** 
