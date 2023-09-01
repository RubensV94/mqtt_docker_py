# Cómo configurar e instalar Mosquitto MQTT Broker utilizando docker 
Para no Debian distros, los comandos pueden variar
_Como configuración por defecto solo se permite el uso de conexiones locales por temas de seguridad.No obstante, más adelante se habilita la autenticación por lo que se evita lo anterior._

## 1. Instalar docker

Instalar Docker Engine en caso de tratarse de Linux o  Docker Desktop si se trata de Windows.

## 2. Crear el directorio base para la configuración de mqtt.
```bash
mkdir mqtt5
cd mqtt5

# Para guardar los archivos mosquitto.conf y pwfile (para la contraseña):
mkdir config
```

## 3. Crear el archivo de configuración de mosquitto - mosquitto.conf
En Linux:
```bash
nano config/mosquitto.conf
```
En Windows:
```bash
cd config
type mosquitto.conf
```

Añadir al archivo mosquitto.conf lo siguiente para tener una configuración básica incluyendo la configuración del websocket:
```
allow_anonymous false
listener 1883
listener 9001
protocol websockets
persistence true
password_file /mosquitto/config/pwfile
persistence_file mosquitto.db
persistence_location /mosquitto/data/
```

## 4. Crear el archivo de la contraseña llamado pwfile:
En Linux:
```bash
touch config/pwfile
```
En Windows: (dentro de la carpeta config)
```bash
type pwfile
```
## 5. Crear el archivo docker-compose con el nombre exacto 'docker-compose.yml'

```yml

version: "3.7"
services:
  # servicio mqtt5 eclipse-mosquitto
  mqtt5:
    image: eclipse-mosquitto #utiliza la imagen de este nombre de dockerhub
    container_name: mqtt5
    ports:
      - "1883:1883" #puerto mqtt por defecto
      - "9001:9001" #puerto mqtt por defecto para websockets
    volumes:
      - ./config:/mosquitto/config:rw
      - ./data:/mosquitto/data:rw
      - ./log:/mosquitto/log:rw

# Volúmenes creados para mapping data,config and log. Permite modificar los archivos desde fuera del docker y tener una permanencia de ellos aunque se reinicie.
volumes:
  config:
  data:
  log:

networks:
  default:
    name: mqtt5-network

```

## 6. Crear y ejecutar el docker container for MQTT
En Linux:
```bash
sudo docker-compose -p mqtt5 up -d
```
En Windows:
```bash
docker-compose -p mqtt5 up -d #-d ejecuta el docker en modo background.
```

### Comprueba si el contenedor esta funcionando.
```bash
sudo docker ps
```

## 7. Crea el usuario y contraseña en el archivo pwfile de dentro del docker: 
```bash
# login modo interactivo (-it) en el contenedor mqtt de id correspondiente
sudo docker exec -it <container-id> sh

#Se abrirá la shell del docker que se encuentra funcionando

# Añadir el usuario y a continuación pide la contraseña:
mosquitto_passwd -c /mosquitto/config/pwfile user
Password: >> 123456789
#Solo se permite configurar un usuario, con el que se pueden identificar múltiples publishers y suscribers.
# (Opcional) Si se quiere eliminar el usuario creado:
mosquitto_passwd -D /mosquitto/config/pwfile <user-name-to-delete>

# Escribe 'exit' para salir del prompt del contenedor.

```
Resetear el contenedor
```bash
sudo docker restart <container-id>
```

# Pruebas de funcionamiento. Broker, publisher y suscriber mqtt contenerizados.

Si quieres comprobar su funcionamiento sigue además los pasos que se detallan a continuación.
Esto va a crear un multi-contenedor docker con un brocker de mqtt, un publisher y un suscriber.

## 1. Añade una red interna en el archivo docker-compose.yaml
-En el mismo nivel que "services:" se define la red.
```yml
networks:
  default:
    name: mqtt5-network
```
-Dentro de cada uno de los servicios, se indica qué red van a utilizar:
```yml
services:
    mqtt5:
      networks:
        - default
```

## 2. Exponer los puertos 1883 de los docker.
Para ello se utilizará el comando:
```yml
ports:
  - "1883"
```

## 3. El archivo docker-compose.yaml final:

```yml
version: '3.8'
services:

  mqtt5:
    image: eclipse-mosquitto
    container_name: mqtt5
    expose:
      - "1883" #puerto mqtt por defecto
      - "9001" #puerto mqtt por defecto para websockets
    ports:
      - "1883:1883"
    networks:
      - default
    volumes:
      - ./config:/mosquitto/config:rw
      - ./data:/mosquitto/data:rw
      - ./log:/mosquitto/log:rw
  
  mqtt5_sub:
    image: "python:3.9.18-slim-bullseye"
    container_name: mqtt5_user_sub
    expose:
      - "1883"
    command:
	-apt update 
	-apt install mosquitto-clients -y
    tty: true #mantiene vivo el contenedor, de otro modo, en cuanto ejecuta el command se apaga automáticamente.
    networks:
      - default
  mqtt5_pub:
    image: "python:3.9.18-slim-bullseye"
    container_name: mqtt5_user_pub
    expose:
      - "1883"
    command: 
      - apt update
      - apt install mosquitto-clients -y
    tty: true
    networks:
      - default
## volumes for mapping data,config and log
volumes:
  config:
  data:
  log:

networks:
  default:
    name: mqtt5-network
```
## 4. Iniciar el docker compose directamente (sin utilizar build):
Iniciar en el bash con:
```bash
docker-compose up
```

## 5. Comprobar la IP del broker mqtt, con el comando docker inspect <id_docker>:
Con el comando siguiente, buscar el id del docker mqtt5:
```bash
docker ps
```
A continuación si estás en windows:
```bash
docker inspect <id_docker_mqtt5> | findstr "IPAddress"
```
Obtendremos la IP del broker mqtt que se necesita para el paso 7.


## 6. Abrir un terminal para publicar y otro para suscribirse:
En un terminal introducimos:
```bash
docker exec -it <id_mqtt_sub> sh
```
Y en el otro:
```bash
docker exec -it <id_mqtt_pub> sh
```

## 7. En los terminales introduce:
-En el de suscripción:
```bash
mosquitto_sub -v -t "test_mqtt" -h <IP_docker_broker> -u user -P 123456789
```
-En el de publicación:
```bash
mosquitto_pub -v -t "test_mqtt" -h <IP_docker_broker> -u user -P 123456789
```
*-t es el topic al que se quiere suscribir. -v mostrar los mensajes verbalmente. -h IP del host.


## 8. Comprobar la recepción:
Se puede ver si todo ha funcionado correctamente como el suscriptor recibe todo lo que se envíe por ese canal.

*Repositorio original github.com: [sukesh-ak / setup-mosquitto-with-docker](https://github.com/sukesh-ak/setup-mosquitto-with-docker)
