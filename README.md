# Minecraft-Server
Minecraft es un sandbox de aventura donde los jugadores se sumergen en un mundo abierto tridimensional generado proceduralmente. El juego se basa en la construcción y la creatividad. Su modo multijugador es bastante utilizado por la comunidad. Y si bien existen muchas guias y empresas que permiten generar un servidor de minecraft de una forma bastante sencilla. En este proyecto buscamos ampliar la visión del mismo videjuego, buscando generar un entorno cloud donde se puedan generar diversos mundos de minecraft dentro de un mismo servidor permitiendo de esta manera entender como funcionan los servicios de alquiler de servidores y que tan escalables y customizables pueden ser estos.

## Herramientas
- DigitalOcean
- Kubernetes
- Docker

# Servicio Cloud
Para este proyecto estamos haciendo uso de IaaS utilizando los servicios de [DigitalOcean](https://www.digitalocean.com/). Utilizamos un droplet que es una maquina virtual con el sistema operativo Ubuntu en la versión 18.04. Para ejecutar el proyecto correctamente es necesario tener como minimo 4 GB de memoria.

Las caracteristicas del Droplet Alquilado en la región de San Francisco:
- 4GB de memoria
- 80 GB de SSD
- 4 TB de transferencia

## Orquestador / Kubernetes
Para el orquestador de kubernetes fue manejado usando [Rancher](https://www.rancher.com/).  

### Docker
Para instalar Rancher primero debemos instalar Docker. Para ello simplemente se hace uso de la siguiente linea de comando en la versión especificada.

```bash
curl https://releases.rancher.com/install-docker/18.09.sh | sh
```

### Rancher
Para instalar Rancher es bastante simple ya que trabaja sobre una imagen de docker. Por lo cual solo se debe correr este comando de Docker.

```bash
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /opt/rancher:/var/lib/rancher \
  --privileged \
  rancher/rancher:v2.4.17
```

Una vez ejecutado solo se debe acceder a la IP del droplet y en nuestro caso pedimos a DigitalOcean una IP reservada para el droplet la cúal era estatica y ganaba acceso publico. Una vez ingresado suele pedirte que configures las contraseña de Rancher.

### Creación del cluster
Para esto en el apartado Global debemos crear un cluster con el boton de Add Cluster

![Global](/images/global.png)

Luego lo creamos desde un nodo existente.

![Add Cluster](/images/add-cluster.png)

Tras ello dejamos la configuración por defecto de ranger y le damos a siguiente.

![Add Cluster 2](/images/add-cluster-2.png)


Finalmente activamos el etcd y control plane para tener un almacen de respaldo de los datos del cluster y el plano de control para manetener el desire state en el cluster.

![Add Cluster 2](/images/add-cluster-3.png)

Luego se copia el comando a usar que en mi caso es el siguiente:

```bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.4.17 --server https://24.199.73.35 --token pqrp96tcmxh5ff6jq2h6m4wfjcz5tcvfl927pj4kllm65grc6m4bts --ca-checksum 610f96f6617ce41e65b92d91e47ae61203ba30511d6380b6b1dbea7b8fca70db --etcd --controlplane --worker
```
Ingresamos el comando en el droplet y luego se espera unos minutos para dejar activo el cluster.

![Cluster Activo](/images/active.png)

### Creación del nodo
Para crear el nodo debemos nos dirijimos al namespace default y la parte de workloads le damos a deploy.

![Workloads](/images/workload.png)

De allí configuramos el nodo con los siguientes valores:
- Docker image: [itzg/minecraft-server](https://hub.docker.com/r/itzg/minecraft-server/)
- Puerto: 25565
- Variables de entorno:
  - EULA=TRUE
- Volumen bind-mount
  - Mount Point: mc
  - Path: /data

![Configuración del nodo](/images/workload-2.png)

![Configuración del nodo](/images/workload-3.png)

Con eso configurado ya estaría listo el nodo.

![Nodo activado](/images/workload-active.png)

### Verificación de que el mundo se creo correctamente
Para ello primero debemos fijarnos de que mundo se creo correctamente en el pod y para ello debemos ingresar a los logs del pod y revisar que aparezca en **Done**.

![Nodo activado](/images/done.png)

Tras ello se puede verificar que los usuarios se puedan conectar al juego entrando a su apartado multiplayer e ingresando la IP reservada mencionada al inicio.

![Nodo activado](/images/game-1.png)
![Nodo activado](/images/game-2.png)
![Nodo activado](/images/game-3.png)
![Nodo activado](/images/game-4.png)

# Escalabilidad

Para esta escalabilidad recomendamos principalmente recurrir a la escalabilidad vertical, ya que las principales operaciones de los servidores de minecraft se hacen en memoria. Por lo cual se dificulta particionar esa tarea en dos pods distintos. Sin embargo, minecraft es un juego con una comunidad bastante grande, por lo cual ya se han creado distintas modificaciones para el juego que pueden brindarnos un escalamiento horizontal pero atribuyendo eventos propios del juego o de las caracteristicas del servidor.

![Nodo activado](/images/server-game-modes.jpg)

Para ello primero hay que entender como son utilizados actualmente los servidores de minecraft. Uno de los servidores bastante conocido dentro de la comunidad es hypixel y este brinda los siguientes modos de juego:
- Bed Wars
- Sky Wars
- Murder Mystery
- Arcade Games
- UHC Champions
- Areana Brawl
- Build Battle
- Cops and Crims
- Duels
- Mega Walls
- Paintball Warfare
- Quackcraft
- Blitz Survival Games
- Smash Heroes
- The TNT Games
- Turbo Kart Racers
- VampireZ
- The Walls
- Warlords
Como vemos son 19 modos de juegos distintos. Si al servidor se conectan cerca de 1000 usuarios y cada usuarios juega uno de los modos mencionados anteriormente se puede generar una saturación en los recursos utilizados para ejecutar el servidor. Sin embargo, si dividimos estos modos de juego en servidores más pequeños ejecutando unicamente lo necesario, podemos optimizar mejor los recursos del cluster. Para ello la comunidad actualmente usa una modificación llamada [bungeecord](https://www.spigotmc.org/wiki/sobre-bungeecord/) que funciona como un proxy que recibe a todos los jugadores que quieran entrar al servidor y los distribuyen en los distintos servidores que tenga el cluster.


![Nodo activado](/images/escalabilidad.png)

Como vemos en la imagen hemos asignado distintos nodos que responden a un modo de juego distinto y poseen sus caractaristicas distintas como la cantidad de usuarios. Sin embargo, en el último nodo decidimos asignarles más pods. Esto debido a que algunos modos de juego no generan cambios en disco y unicamente sirven para responder a operaciones exclusivas usuarios. Por ende podemos crear 2 pods para recibir a más usuarios y cada uno tenga un pod exclusivo donde realizar su juego tras finalizar liberar el recurso para otros jugadores.

  

