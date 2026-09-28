# Servicios en Red e Internet
## 1 - Instalación del Servicio SSH
En esta primera semana de curso hemos empezado con un servicio básico llamado SSH que se usa para conectarse a la terminal de un servidor mediante un equipo cliente. 
El protocolo SSH viene como sustituto de Telnet el cual se usaba para la misma función de SSH pero cualquiera que estuviera en la red podía ver el tráfico en texto plano, por lo que 
se creo SSH que realiza la misma función pero de manera que el tráfico que se genera aparece cifrado.
### Modelo Cliente - Servidor
Para entender el servicio SSH primero hay que entender el modelo **Cliente - Servidor** ya que el servicio SSH se basa en este modelo.
Para establecer una conexión SSH necesitamos un equipo servidor el cual va a ser al que nos vamos a conectar y equipo cliente que es el que va a conectarse al servidor para ejecutar comandos
en su consola. \
En nuestro caso vamos a usar una máquina virtual para establecer la conexión SSH, para realizar esta practica necesitaremos:
* Máquina Física -> En nuestro caso va a ser nuestro Windows 11 que va a ser el cliente que se conecte al servidor.
* Máquina Virtual -> En este caso Ubuntu con el servicio SSH actuando de servidor para realizar la conexión.
Una vez entendemos todo esto, podemos empezar a instalar el servicio SSH para realizar la conexión entre nuestra máquina física y la máquina virtual.
***
### Configuración del Cliente
En nuestro caso, la máquina cliente va a ser Windows 11, por defecto Windows ya trae el servicio SSH instalado, pero en el caso de que no viniera instalado por defecto deberíamos ejecutar
el siguiente comando en PowerShell dependiendo de lo que necesitemos:
#### Instalar el Cliente SSH
``` powershell
dism /online /add-capability /capabilityname:OpenSSH.Client~~~~0.0.1.0
```
#### Instalar el Servidor SSH
``` powershell
dism /online /add-capability /capabilityname:OpenSSH.Server~~~~0.0.1.0
```
En este caso yo no he tenido que ejecutar estos comandos porque mi Windows ya lo traía el servicio cliente por defecto así que en esta máquina principalmente no hay 
nada más que configurar por ahora (después tendremos que configurar las claves para no tener que identificarnos cada vez que nos conectemos a un servidor).

### Configuración del Servidor
La máquina que va a hacer de servidor en este caso concreto va a ser nuestra máquina virtual con sistema operativo Ubuntu, para configurar la parte del servidor debemos de instalar el servicio SSH
en modo servidor mediante la consola de Ubuntu.

Para comenzar con la instalación del servicio de SSH en modo Servidor desde la consola de Ubuntu tenemos que ejecutar los siguientes comandos:
``` bash
sudo apt update
sudo apt install openssh-server -y
```

