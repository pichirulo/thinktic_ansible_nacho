# thinktic_ansible_nacho
curso ansible thinktic nov 2022 - nacho

cero conocimientos previos de git, docker y ansible ;-)
------------------------------------------------------------------------------------
EJER01 - Se prepara el entorno para trabajar, crear maquinas con contenedores lxc, configurarlas. Uso de ansible adhoc con inventarios y modulos.
--
//crear contenedores lxc con imagen de ubuntu y de rocky
#lxc launch images:ubuntu/22.10 nacubu
#lxc launch images:rockylinux/8 nacrocky
#lxc list
![image](https://user-images.githubusercontent.com/90699821/205459032-e8fef946-b293-49fa-90fa-19234b68d5bd.png)
//acceso a la shell del rocky para una primera conf de red, ssh, usuario
#lxc exec nacrocky -- /bin/bash
//adduser alumno, añado alumno al /etc/sudoers
//creo clave ssh para alumno, añado clave al .ssh/authorized_keys 
alumno@ubuntu:~$ ssh alumno@nacrocky
Last login: Sat Dec  3 20:26:55 2022 from 10.218.144.1
alumno@nacrocky ~$ ip a l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:a7:0d:f3 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.218.144.27/24 brd 10.218.144.255 scope global dynamic noprefixroute eth0
       valid_lft 3306sec preferred_lft 3306sec
alumno@nacrocky ~$ 

//se puede especificar el inventory que queremos usar en el fichero ansible.cfg y seria el que usaria siempre que no usemos la opcion "-i nombre_inventario"
[defaults]
inventory = inventory-dns
//fichero inventory por direcciones IP
#cat ansible-nacho/thinktic_ansible_nacho/ejer01/inventory
10.218.144.233
10.218.144.221
10.218.144.100
10.218.144.27
10.218.144.108
//fichero inventory por FQDN
#cat ansible-nacho/thinktic_ansible_nacho/ejer01/inventory-dns 
u1
u2
prueba-docker
nacrocky
nacubu
//comprobamos el inventario, vemos que como no tenemos etiquetas nos devuelven todas las ip en "ungrouped"
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer01$ ansible-inventory -i inventory --list
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "ungrouped"
        ]
    },
    "ungrouped": {
        "hosts": [
            "10.218.144.100",
            "10.218.144.108",
            "10.218.144.221",
            "10.218.144.233",
            "10.218.144.27"
        ]
    }
}
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer01$ ansible-inventory -i inventory-dns --list
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "ungrouped"
        ]
    },
    "ungrouped": {
        "hosts": [
            "nacrocky",
            "nacubu",
            "prueba-docker",
            "u1",
            "u2"
        ]
    }
}
//mirar si están presentes los servidores, que estan definidos en el fichero inventory, con el modulo ping 
//"A trivial test module, this module always returns pong on successful contact. It does not make sense in playbooks, but it is useful from /usr/bin/ansible to verify the ability to login and that a usable Python is configured. This is NOT ICMP ping, this is just a trivial test module that requires Python on the remote-node."
https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html
$ ansible all -i inventory -m ping
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer01$ ansible all -i inventory -m ping
10.218.144.108 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ssh: connect to host 10.218.144.108 port 22: Connection refused",
    "unreachable": true
}
10.218.144.27 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
10.218.144.233 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
10.218.144.100 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
10.218.144.221 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
//vemos que nos dice que la 10.218.144.108 no es accesible por ssh, lo que es correcto porque hemos creado dos contenedores lxc nuevos (nacrocky y nacubu) pero solo hemos configurado la maquina nacrocky
//Ahora agrupamos agrupar los hosts usandolos "[]" para poder aplicar comandos sobre grupos en vez de sobre todos los elementos del inventory
#cat ansible-nacho/thinktic_ansible_nacho/ejer01/inventory-dns
[ubuntu]
u1
u2
prueba-docker
nacubu
[rocky]
nacrocky
//comprobamos que esta ok el inventario-dns con las etiquetas
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer01$ ansible-inventory -i inventory-dns --list
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "rocky",
            "ubuntu",
            "ungrouped"
        ]
    },
    "rocky": {
        "hosts": [
            "nacrocky"
        ]
    },
    "ubuntu": {
        "hosts": [
            "nacubu",
            "prueba-docker",
            "u1",
            "u2"
        ]
    }
}
//ahora ejecutamos el mismo modulo ping pero solo sobre la etiqueta rocky
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer01$ ansible rocky -i inventory-dns -m ping
nacrocky | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
------------------------------------------------------------------------------------
EJER02 - Instalar un servidor web (apache) en rocky.
--
//Los Playbooks nos van a permitir la definición de las tareas que queremos ejecutar
//comprobar la sintaxisde un fichero YAML comando ansible-playbook --syntax-check fichero.yaml

//comprobamos que no lo tenemos instalado previamente en nuestra maquina nacrocky
alumno@nacrocky ~$ yum list installed httpd
Error: No hay paquetes que se correspondan con la lista
//definimos en el ansible.cfg el inventory y el usuario root para hacer la instalacion en nacrocky, y que nos pida la pass al empezar
[defaults]
inventory = inventory-dns
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=True
//creamos nuestro primer playbook  task.yml 
//Le decimos que lo haga solo en la maquina nacrocky usando la etiqueta rocky del inventaary-dns
//usamos los modulos
//ansible.builtin.yum  --> para instalar paquetes en rocky (yum)
//ansible.builtin.copy --> copiar fichero (el index de nuestro dir)
//ansible.builtin.service --> para arrancar los servicios/demonios httpd y firewalld
//ansible.posix.firewalld --> abrimos puertos en el firewall, en este caso servicios http (80) y https (443)
https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html

task.yml
---
- name: Instalación de Apache
  hosts: rocky
  tasks:
    - name: Instalando el paquete httpd
      ansible.builtin.yum:
        name: httpd
        state: latest
    - name: Copiando index.html personalizado
      ansible.builtin.copy:
        src: index.html
        dest: /var/www/html/index.html
    - name: Arrancando el servicio httpd
      ansible.builtin.service:
        name: httpd
        enabled: true
        state: started 
    - name: Instalando el paquete firewall
      ansible.builtin.yum:
        name: firewalld
        state: latest
    - name: Arrancando el servicio firewall
      ansible.builtin.service:
        name: firewalld
        enabled: true
        state: started 
    - name: Abrir el firewall para el http
      ansible.posix.firewalld:
        service: http
        state: enabled
        immediate: true
        permanent: true
    - name: Abrir el firewall para el https
      ansible.posix.firewalld:
        service: https
        permanent: yes
        state: enabled
...

//Comprobamos que el fichero task.yml esta correcto
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer02$ ansible-playbook --syntax-check tasks.yaml 

playbook: tasks.yaml
//ejecutamos el playbook task.yaml para instalar el apache en la maquina nacrocky
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer02$ ansible-playbook  tasks.yaml 
BECOME password: 

PLAY [Instalación de Apache] ***************************************************

TASK [Gathering Facts] *********************************************************
ok: [nacrocky]

TASK [Instalando el paquete httpd] *********************************************
changed: [nacrocky]

TASK [Copiando index.html personalizado] ***************************************
fatal: [nacrocky]: FAILED! => {"changed": false, "checksum": "045ec966d6ba77a86ac371105da5402006d00051", "gid": 0, "group": "root", "mode": "0644", "msg": "chown failed: failed to look up user www-data", "owner": "root", "path": "/var/www/html/index.html", "size": 210, "state": "file", "uid": 0}

PLAY RECAP *********************************************************************
nacrocky                   : ok=2    changed=1    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
//nos da un error porque no existe el user www-data
//quitamos el usuario www-data y dejamos que sea root (aunque no es una buena practica) y lo volvemos a lanzar  

alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer02$ ansible-playbook  tasks.yaml 
BECOME password: 

PLAY [Instalación de Apache] ***************************************************

TASK [Gathering Facts] *********************************************************
ok: [nacrocky]

TASK [Instalando el paquete httpd] *********************************************
ok: [nacrocky]

TASK [Copiando index.html personalizado] ***************************************
ok: [nacrocky]

TASK [Arrancando el servicio httpd] ********************************************
ok: [nacrocky]

TASK [Abrir el firewall para el http] ******************************************
fatal: [nacrocky]: FAILED! => {"changed": false, "msg": "Python Module not found: firewalld and its python module are required for this module,                         version 0.2.11 or newer required (0.3.9 or newer for offline operations)"}

PLAY RECAP *********************************************************************
nacrocky                   : ok=4    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0 
//ahora nos da un error porque no existe el modulo de firewall, Le decimos que lo instale y lo volvemos a probar
    - name: Instalando el paquete firewall
      ansible.builtin.yum:
        name: firewalld
        state: latest
    - name: Arrancando el servicio firewall
      ansible.builtin.service:
        name: firewalld
        enabled: true
        state: started 
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer02$ ansible-playbook  tasks.yaml 
BECOME password: 

PLAY [Instalación de Apache] ***************************************************

TASK [Gathering Facts] *********************************************************
ok: [nacrocky]

TASK [Instalando el paquete httpd] *********************************************
ok: [nacrocky]

TASK [Copiando index.html personalizado] ***************************************
changed: [nacrocky]

TASK [Arrancando el servicio httpd] ********************************************
ok: [nacrocky]

TASK [Instalando el paquete firewall] ******************************************
ok: [nacrocky]

TASK [Arrancando el servicio firewall] *****************************************
changed: [nacrocky]

TASK [Abrir el firewall para el http] ******************************************
changed: [nacrocky]

TASK [Abrir el firewall para el https] *****************************************
changed: [nacrocky]

PLAY RECAP *********************************************************************
nacrocky                   : ok=8    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
//ahora ya le gusta mas ( failed=0 ) ;-)  comprobamos que ha hecho en la maquina rocky y si se ve mi index.html
//ahora si estan instalados los paquetes http y firewall
alumno@nacrocky ~$ yum list installed httpd
Paquetes instalados
httpd.x86_64                                      2.4.37-51.module+el8.7.0+1059+126e9251                                      @appstream
alumno@nacrocky ~$ yum list installed firewalld
Paquetes instalados
firewalld.noarch                           
//y los servicios corriendo httpd y firewalld
alumno@nacrocky ~$ systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/system/httpd.service.d
           └─zzz-lxc-service.conf
   Active: active (running) since Sat 2022-12-03 22:08:15 UTC; 20min ago
     Docs: man:httpd.service(8)
 Main PID: 4417 (httpd)
   Status: "Total requests: 6; Idle/Busy workers 100/0;Requests/sec: 0.00491; Bytes served/sec:   4 B/sec"
    Tasks: 213 (limit: 50131)
   Memory: 19.8M
   CGroup: /system.slice/httpd.service
           ├─4417 /usr/sbin/httpd -DFOREGROUND
           ├─4418 /usr/sbin/httpd -DFOREGROUND
           ├─4419 /usr/sbin/httpd -DFOREGROUND
           ├─4420 /usr/sbin/httpd -DFOREGROUND
           └─4421 /usr/sbin/httpd -DFOREGROUND
alumno@nacrocky ~$ systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
  Drop-In: /run/systemd/system/firewalld.service.d
           └─zzz-lxc-service.conf
   Active: active (running) since Sat 2022-12-03 22:24:28 UTC; 4min 19s ago
     Docs: man:firewalld(1)
 Main PID: 9373 (firewalld)
    Tasks: 2 (limit: 50131)
   Memory: 26.8M
   CGroup: /system.slice/firewalld.service
           └─9373 /usr/libexec/platform-python -s /usr/sbin/firewalld --nofork --nopid
//comprobamos que nos ha copiado el fichero index.html
alumno@nacrocky ~$ ls /var/www/html/index.html -l
-rw-r--r-- 1 root root 186 dic  3 22:24 /var/www/html/index.html
alumno@nacrocky ~$ cat /var/www/html/index.html
<!doctype html>
<html>
<head>
    <title>Ansible</title>
    <meta charset="utf-8">
</head>
<body>
<p>Soy NACHO, con mi primer playbook chispas ;-)</p>
<p>HOLA MUNDO</p>
</body>
</html>
//y al abrir el navegador nos sale nuestro index.html personalizado

![image](https://user-images.githubusercontent.com/90699821/205464618-194656c8-855a-4577-904c-59530dc69647.png)

------------------------------------------------------------------------------------
EJER03 - Instalar un servidor mariaDB en rocky.
--
//Para instalar mariadb usamos el modulo de yum y le pasamos los paquetes por variables definidas en un archivo "vars_rocky8.yaml"
//usamos una variable en local para pasarle la pass de root.

vars_rocky8.yaml
---
mariadb_packages:
  - mariadb
  - perl                                                      
  - perl-IO-Socket-SSL
  - perl-libwww-perl
  - mariadb-server

task.yaml
---
- name: Instalación de MariaDB
  hosts: rocky
  vars:
    - mysql_root_password: "nacho01"   
  vars_files: # importación de variables desde fichero
    - vars_rocky8.yaml
  tasks:
    - name: Instalando el paquete mariadb
      ansible.builtin.yum:
        name: "{{ mariadb_packages }}"
        state: latest
    - name: arrancando mariadb
      ansible.builtin.service:
         name: mariadb
         enabled: true
         state: started
    - name: creando pass de mysql
      community.mysql.mysql_user module:
          login_user: root
          login_password: "{{ mysql_root_password }}"
          name: alumno
...
//No se mucho de BBDD asi que no puedo hacer la parte de configurarla, pero si se puede comprobar que mariadb esta instalado y el demonio corriendo en la maquina nacrocky

alumno@nacrocky ~]$ sudo systemctl status mariadb
[sudo] password for alumno: 
● mariadb.service - MariaDB 10.3 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/system/mariadb.service.d
           └─zzz-lxc-service.conf
   Active: active (running) since Wed 2022-12-07 00:22:27 UTC; 14min ago
     Docs: man:mysqld(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 1040 ExecStartPost=/usr/libexec/mysql-check-upgrade (code=exited, status=0/SUCCESS)
  Process: 756 ExecStartPre=/usr/libexec/mysql-prepare-db-dir mariadb.service (code=exited, status=0/SUCCESS)
  Process: 725 ExecStartPre=/usr/libexec/mysql-check-socket (code=exited, status=0/SUCCESS)
 Main PID: 793 (mysqld)
   Status: "Taking your SQL requests now..."
    Tasks: 30 (limit: 50131)
   Memory: 73.0M
   CGroup: /system.slice/mariadb.service
           └─793 /usr/libexec/mysqld --basedir=/usr
dic 07 00:22:27 nacrocky systemd[1]: Starting MariaDB 10.3 database server...
dic 07 00:22:27 nacrocky mysql-prepare-db-dir[756]: Database MariaDB is probably initialized in /var/lib/mysql already, nothing is done.
dic 07 00:22:27 nacrocky mysql-prepare-db-dir[756]: If this is not the case, make sure the /var/lib/mysql is empty before running mysql-prepare-db-dir.
dic 07 00:22:27 nacrocky mysqld[793]: 2022-12-07  0:22:27 0 [Note] /usr/libexec/mysqld (mysqld 10.3.35-MariaDB) starting as process 793 ...
dic 07 00:22:27 nacrocky systemd[1]: Started MariaDB 10.3 database server.

//La tarea de asignar la pass da un error que no he sabido quitar
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer03$ ansible-playbook tasks.yaml 
BECOME password: 
ERROR! couldn't resolve module/action 'community.mysql.mysql_user module'. This often indicates a misspelling, missing collection, or incorrect module path.

The error appears to be in '/home/alumno/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer03/tasks.yaml': line 18, column 7, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

         state: started
    - name: creando pass de mysql
      ^ here
//
//
alumno@ubuntu:~/Escritorio/ansible-nacho/thinktic_ansible_nacho/ejer03$ ansible-galaxy collection install community.mysql
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Skipping 'community.mysql' as it is already installed


