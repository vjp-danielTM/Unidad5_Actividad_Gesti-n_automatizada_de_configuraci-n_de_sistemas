# Unidad5_Actividad_Gestión_automatizada_de_configuración_de_sistemas

## Preparación del entorno y despliegue de nodos

En primer lugar, comprobé el estado inicial del clúster con el comando `kubectl get nodes`, observando que en ese momento únicamente estaba disponible el nodo principal `ansible`, que actuaba como `control-plane`.

![Comprobación inicial de nodos](images/101.png)

A continuación, verifiqué la versión del cliente de Kubernetes con `kubectl version --client=true` y después inicié el clúster Minikube con varios nodos utilizando el perfil `ansible` mediante el comando `minikube start --nodes 3 -p ansible`. Durante este proceso se levantó el nodo principal y se añadió un nodo worker adicional, quedando configurado el contexto para trabajar con este clúster.

![Inicio del clúster Minikube](images/102.png)

Después, añadí dos nodos worker más al clúster con el comando `minikube node add -p ansible`. De esta forma se incorporaron los nodos `ansible-m03` y `ansible-m04`, ampliando la infraestructura disponible para repartir los distintos componentes de la aplicación.

![Adición de nodos worker](images/103.png)

Una vez creados los nodos, ejecuté de nuevo `kubectl get nodes` para comprobar que el clúster había quedado formado por el nodo principal `ansible` y los workers `ansible-m03` y `ansible-m04`, todos ellos en estado `Ready`.

![Verificación de nodos creados](images/104.png)

Seguidamente, preparé el directorio de trabajo copiando los archivos necesarios de la práctica, entre ellos el `Dockerfile` y el fichero `store-app-k8s.yaml`, dentro de la carpeta `Unidad5/Actividad-Ansible/store-app`. Con ello dejé reunidos en una misma ubicación los recursos necesarios para continuar con la construcción y despliegue de la aplicación.

![Copia de archivos al directorio de trabajo](images/105.png)

Después comprobé el contenido del directorio y descomprimí el paquete `store-app-postgree.zip`, obteniendo así el proyecto `store-app` con su estructura de ficheros y código fuente. Este paso fue necesario para disponer de la aplicación sobre la que se iba a generar la imagen Docker.

![Descompresión del proyecto store-app](images/106.png)

Con el proyecto ya preparado, construí la imagen Docker de la aplicación directamente dentro del entorno de Minikube mediante el comando `minikube image build -t store-app:latest . --all -p ansible`. De esta manera, la imagen quedó disponible en todos los nodos del clúster, evitando tener que subirla a un registro externo.

![Construcción de la imagen en Minikube](images/107.png)

Por último, realicé también la construcción de la imagen con `docker build -t store-app:latest .` para dejar preparada la versión de la aplicación basada en PostgreSQL. En la salida puede verse que la imagen se generó correctamente y quedó etiquetada como `store-app:latest`.

![Construcción final de la imagen Docker](images/108.png)

Con esto aplicamos el *manifest* y con `kubectl apply -f store-app-k8s.yaml` le dice a **Kubernetes** que lea el fichero store-app-k8s.yaml y cree o actualice en el clúster los recursos definidos ahí dentro

![aplicar manifest](images/109.png)

Ahora con el sleep lo que hacemos es darle tiempo a que se levante todo, y con el get all vemos lo qe tenemos levantado.

![comandos](images/110.png)

`kubectl get pods -o wide` muestra la lista de pods, pero con más detalle que el comando

Además del nombre y estado, suele enseñar datos como:

- IP del pod

- Nodo donde está corriendo

- Tiempo de vida

- Más columnas de información

Y el siguiente comando `minikube service store-app --url -p ansible` te devuelve la **URL** para acceder al servicio *store-app* dentro del perfil de *Minikube* llamado **ansible**

![comandos](images/111.png)

Y si vamos a la *url* vemos que esta la **aplicacion**.

![comandos](images/112.png)

Este conjunto de comandos instala Ansible en el sistema y comprueba que se ha instalado correctamente.

- `sudo apt update`: actualiza la lista de paquetes.
- `sudo apt install software-properties-common`: instala utilidades necesarias para añadir repositorios.
- `sudo add-apt-repository --yes --update ppa:ansible/ansible`: añade el repositorio de Ansible.
- `sudo apt update`: recarga los paquetes disponibles con el nuevo repositorio.
- `sudo apt install ansible`: instala Ansible.
- `ansible --version`: verifica la instalación mostrando la versión.

![comandos](images/113.png)

Este comando abre una sesión SSH dentro del nodo worker `m03` del perfil `ansible` de Minikube.

![comandos](images/114.png)

Este comando abre una sesión SSH dentro del nodo worker `m04` del perfil `ansible` de Minikube.

![comandos](images/115.png)

Este comando abre una sesión SSH dentro del nodo `principal` del perfil `ansible` de Minikube

![comandos](images/116.png)

Creamos una clave *ssh* en mi maquina

![comandos](images/117.png)

Este primer conjunto de comandos copia la clave pública de Minikube a los workers para permitir acceso SSH sin contraseña.

- `minikube ssh --profile=ansible 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/minikube_key.pub`: crea la carpeta `.ssh` en el nodo y añade la clave pública al fichero `authorized_keys`.
- `minikube ssh --node m02 --profile=ansible ...`: repite el proceso en el worker `m02`.
- `minikube ssh --node m03 --profile=ansible ...`: repite el proceso en el worker `m03`.
- `Ctrl + C`: se usa para cortar el comando cuando queda esperando entrada.

![comandos](images/118.png)

Y con este otro conjunto de comandos lo que hace es entrar a una sesion de ssh.

- `ssh -i ~/.ssh/minikube_key docker@$(minikube ip --profile=ansible)`
- `ssh -i ~/.ssh/minikube_key docker@$(minikube ip --node m02 --profile=ansible)`
- `ssh -i ~/.ssh/minikube_key docker@$(minikube ip --node m03 --profile=ansible)`

![comandos](images/119.png)
![comandos](images/120.png)

Recordamos que un ***inventory*** es un archivo donde tenemos el nombre e ip de un conjunto de nodos relacionados. Vamos a crear el Inventario de hosts de nuestro despliegue: **inventory.ini** de manera automático con un script

```bash
#!/bin/bash
set -euo pipefail

CONTROL_IP=$(minikube -p ansible ip)
WORKER_APP_IP=$(minikube -p ansible ip -n ansible-m03)
WORKER_DB_IP=$(minikube -p ansible ip -n ansible-m04)

CONTROL_KEY=$(minikube -p ansible ssh-key)
WORKER_APP_KEY=$(minikube -p ansible ssh-key -n ansible-m03)
WORKER_DB_KEY=$(minikube -p ansible ssh-key -n ansible-m04)

cat > inventory.ini << EOF
[control_plane]
ansible ansible_host=$CONTROL_IP ansible_user=docker ansible_ssh_private_key_file=$CONTROL_KEY

[app_nodes]
ansible-m03 ansible_host=$WORKER_APP_IP ansible_user=docker ansible_ssh_private_key_file=$WORKER_APP_KEY

[db_nodes]
ansible-m04 ansible_host=$WORKER_DB_IP ansible_user=docker ansible_ssh_private_key_file=$WORKER_DB_KEY

[all_workers:children]
app_nodes
db_nodes

[all:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
EOF

```

![comandos](images/121.png)

Y cuando lo ejecutamos se crea el **inventory.ini**

![comandos](images/122.png)

Esto es lo que hay dentro del inventory donde vemos que estan las ips de los nodos

![comandos](images/123.png)

Al ejecutar el `ansible all -i inventory.ini -m ping` probamos si hacen ping

![comandos](images/124.png)

Con esto lo que hacemos es probar *test* de **ansible**

![comandos](images/125.png)

Este playbook usa los módulos `group` y `user` para crear un grupo y un usuario llamados `maint` en todos los nodos workers.

Primero crea el grupo `maint` si no existe. Después crea el usuario `maint`, lo asigna a ese grupo, le configura `/bin/bash` como shell y le genera su carpeta personal.

Con esto dejas preparado el usuario de mantenimiento para usarlo en el resto de tareas del clúster.

![comandos](images/126.png)

Ejecutamos el *playbook*

![comandos](images/127.png)

Y *comprobamos* que se ha ejecutado **concretamente**

![comandos](images/128.png)

Este paso consiste en crear un playbook de Ansible llamado `manage_maint_files.yml`. En él se define que las tareas se ejecutarán sobre el grupo `all_workers` con privilegios de administrador, y se establece como variable el usuario `maint`

Luego, el playbook crea directorios distintos según el tipo de nodo:

- en los nodos de aplicación crea `/opt/appmaint/logs`
- en los nodos de base de datos crea `/opt/dbmaint/backups`

Además, asigna como propietario y grupo al usuario `maint` y define los permisos correspondientes en cada caso

![comandos](images/129.png)

Ejecutamos el *playbook*

![comandos](images/130.png)

Y lo comprobamos

![comandos](images/131.png)

hemos creaado el siguiente *playbook* para añadir **clave pública** de usuario **maint** para que pueda conectarse a los *nodos*:

![comandos](images/132.png)

Y ejecutamos el *playbook*

![comandos](images/133.png)

Ejecutar directamente **módulo** file desde *Ansible*

![comandos](images/134.png)

Y con estoo ejecutamos directamente módulo **file**:

![comandos](images/135.png)

Vamos a ver cómo podemos instalar software y también desinstalar. Tan sólo tenemos que modificar el state que puede tomar los valores: **absent**, **build-dep**, **fixed**, **latest**, **present**.

![comandos](images/136.png)

Ahora ejecutamos el *playbook*

![comandos](images/137.png)

Instalar directamente desde Ansible

> ansible all -i inventory.ini -m apt -a "name=curl state=present"

![comandos](images/138.png)

Creamos el siguiente *playbook* para instalar **nginx**, levantar el servicio y comprobar que está ejecutándose:

![comandos](images/141.png)

Ejecutamos el *playbook*

![comandos](images/142.png)



### ORGANIZACIÓN DE PROYECTOS EN ANSIBLE

inventory: directorio donde se separan los hosts en inventarios

> store_app.ini

```bash
[web]
web01
web02

[db]
db01
```

**Variables**: *host_vars* y *group_vars*: contienen archivos que contienen la declaración de variables que usaremos en los **playbooks**

> vars_store_app.yml

```bash
vars:
  paquete: nginx
```

![comandos](images/143.png)

### Roles

*Ansible* tiene comando **automático** para crear la estructura de ***rol***

> site.yml

```bash
---
- hosts: app_nodes
  become: yes

  roles:
    - nginx
```

![comandos](images/144.png)

> roles/nginx/tasks/main.yml

```bash
---
- name: Instalar nginx
  apt:
    name: nginx
    state: present

- name: Copiar index.html
  copy:
    src: index.html
    dest: /var/www/html/index.html

- name: Arrancar nginx
  service:
    name: nginx
    state: started
    enabled: yes
```

![comandos](images/145.png)

**ansible.cfg** es el archivo donde se definen los parámetros de funcionamiento de *Ansible*

> ansible.cfg

```bash
[defaults]
inventory = inventory/inventory.ini
roles_path = roles
# silenciar las notificaciones de python
interpreter_python = auto_silent
```

Y luego lo ejecutamos con `ansible-playbook site.yml`

![comandos](images/146.png)

#### Autor

> ***Daniel Trujillo Martin***
