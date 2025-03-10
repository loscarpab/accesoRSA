# SSH seguro

Vamos a configurar un acceso seguro por SSH a un sistema Debian dentro de un contenedor Docker, utilizando claves RSA para la autenticación.

## Generación de claves RSA

Para comenzar, abrimos una terminal y ejecutamos el siguiente comando para generar un par de claves SSH:

```
sudo ssh-keygen -t rsa -b 4096
```

- `-t rsa`: Especifica que se generará una clave RSA, uno de los algoritmos de cifrado más utilizados.
- `-b 4096`: Define el tamaño de la clave en bits, lo que proporciona mayor seguridad.

Este comando generará dos archivos:
- **id_rsa**: La clave privada, que nunca debe compartirse.
- **id_rsa.pub**: La clave pública, que se copiará en el servidor para permitir la autenticación.

## Creación de la imagen Docker

Ahora construiremos la imagen Docker que contendrá el sistema Debian con el servidor SSH habilitado. Para ello, creamos un archivo llamado `Dockerfile` con el siguiente contenido:

```
FROM debian:latest

RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo "root:root" | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

Explicación:
- Se utiliza la imagen más reciente de Debian.
- Se instala el servidor SSH.
- Se crea el directorio necesario para el daemon SSH.
- Se establece la contraseña de root como `root`.
- Se permite el acceso SSH como root y se habilita la autenticación por contraseña.
- Se expone el puerto 22 para conexiones SSH.

Para construir la imagen con estos parámetros, ejecutamos:

```
docker build -t ssh-debian .
```

## Configuración con Docker Compose

Creamos un archivo `docker-compose.yml` con la siguiente configuración:

```
version: '3.8'

services:
  ssh.server:
    build: .
    container_name: ssh-container
    ports:
      - "2222:22"
    restart: always
    volumes:
      - /ssh_keys:/home/ubuntu/.ssh

volumes:
  ssh_keys:
```

Este archivo especifica:
- La construcción del contenedor desde el `Dockerfile`.
- La asignación de un nombre al contenedor.
- La exposición del puerto 22 en el contenedor como el puerto 2222 en el host.
- La persistencia de claves SSH a través de un volumen.

## Iniciar el contenedor Docker

Para levantar el contenedor con la configuración definida, ejecutamos:

```
docker compose up
```

## Copiar la clave pública al contenedor

Para poder autenticarnos sin contraseña, copiamos nuestra clave pública al contenedor:

```
docker cp ~/.ssh/id_rsa.pub <container_id>:/root/.ssh/authorized_keys
```

> La ID del contenedor se puede obtener con `docker ps`.

## Conexión por SSH

Finalmente, accedemos al contenedor usando nuestra clave privada:

```
ssh -p 2222 -i id_rsa root@localhost
```

Con esto, hemos configurado un acceso seguro por SSH a nuestro sistema Debian dentro de Docker, utilizando claves RSA para la autenticación.

