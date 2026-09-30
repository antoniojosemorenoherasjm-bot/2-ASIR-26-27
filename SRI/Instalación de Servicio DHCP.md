# Montar un servidor DHCP en Ubuntu con VirtualBox

Apuntes de mi práctica de Servicios de Red e Internet. Dos máquinas Ubuntu en VirtualBox: `UbuntuSRI` (servidor) y `UbuntuSRI Cliente`. El profesor pide que el servidor tenga **una sola tarjeta de red**, así que en vez de usar dos adaptadores cambio el adaptador entre **Puente** (internet y SSH) y **Red interna** (la práctica de DHCP).

El servidor mantiene siempre la IP `172.16.5.210/24`, tanto en modo Puente como en modo laboratorio.

---

## Cómo funciona DHCP en un par de líneas

El cliente no tiene IP, así que grita a toda la red (`0.0.0.0` → `255.255.255.255`, puertos UDP 68 → 67). El proceso tiene cuatro pasos, conocido como DORA:

1. **Discover:** el cliente pregunta si hay algún servidor DHCP.
2. **Offer:** el servidor ofrece una IP.
3. **Request:** el cliente dice que la acepta (también en broadcast, para avisar a otros servidores).
4. **Acknowledge:** el servidor confirma y fija el tiempo de concesión.

Un detalle que vi en la captura: si el cliente ya tiene una concesión vigente y reinicia la conexión, se salta Discover y Offer y solo hace Request y ACK. Por eso al principio solo me salieron dos paquetes.

---

## 1. Preparar todo con internet (modo Puente)

Todo esto se hace con el adaptador en Puente, porque luego en Red interna no habrá internet.

```bash
sudo apt update && sudo apt install isc-dhcp-server openssh-server tcpdump
```

Al terminar da un error de arranque del servicio. Es normal, aún no está configurado.

Miro cómo se llama la interfaz (`ip a`; en mi caso `enp0s3`) y se la indico al servicio:

```bash
sudo sed -i 's/^INTERFACESv4=.*/INTERFACESv4="enp0s3"/' /etc/default/isc-dhcp-server
grep INTERFACESv4 /etc/default/isc-dhcp-server
```

Ahora la configuración. Uso `tee` porque con `sudo cat > fichero` la redirección la haría mi shell sin permisos y fallaría:

```bash
sudo tee /etc/dhcp/dhcpd.conf > /dev/null << 'EOF'
authoritative;
default-lease-time 600;
max-lease-time 7200;

subnet 172.16.5.0 netmask 255.255.255.0 {
    range 172.16.5.50 172.16.5.100;
    option domain-name-servers 8.8.8.8;
    option broadcast-address 172.16.5.255;
}
EOF
```

Tres cosas que importan aquí:

- El `subnet` lleva la dirección de **red** (`.0`), no la del servidor.
- El `range` no puede incluir `.210`, que es la IP del servidor.
- No pongo `option routers` porque en la Red interna no hay router.

Compruebo que el fichero se guardó de verdad (me falló una vez) y lo valido:

```bash
grep -v '^\s*#' /etc/dhcp/dhcpd.conf | grep -v '^\s*$'
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Ojo: `dhcpd -t` solo comprueba la sintaxis. Un fichero de ejemplo sin bloque `subnet` también pasa, y luego el servicio no arranca.

Por último, dejo el servicio deshabilitado para que nunca arranque solo:

```bash
sudo systemctl disable --now isc-dhcp-server
```

---

## 2. Pasar al modo laboratorio

Apago el servidor y en VirtualBox pongo el Adaptador 1 en **Red interna** con el nombre `intnet`, elegido del desplegable para no equivocarme al escribirlo. El Adaptador 2 lo dejo desactivado.

Antes de arrancar el DHCP compruebo que de verdad estoy en la red interna. Desde PowerShell en mi PC:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" showvminfo "UbuntuSRI" | Select-String "NIC "
```

Tiene que decir `Internal Network 'intnet'`. Si dice `Bridged Interface`, no arranco nada.

Entonces sí:

```bash
sudo systemctl start isc-dhcp-server
systemctl status isc-dhcp-server
sudo journalctl -u isc-dhcp-server -b | grep -i listening
```

Debe estar `active (running)` y salir una línea como `Listening on LPF/enp0s3/.../172.16.5.0/24`.

---

## 3. El cliente

El cliente tiene que estar con la interfaz en automático y sin ninguna IP fija. En `/etc/netplan/99-config.yaml`:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: true
```

```bash
sudo chmod 600 /etc/netplan/99-config.yaml
sudo netplan apply
```

Su adaptador también va en Red interna, con el mismo nombre `intnet`. Para pedir IP:

```bash
sudo nmcli con down netplan-enp0s3 && sudo nmcli con up netplan-enp0s3
ip a show enp0s3
```

`dhclient` ya no viene en Ubuntu reciente, así que uso `nmcli`.

---

## 4. Comprobar que funciona

En el servidor miro las concesiones:

```bash
cat /var/lib/dhcp/dhcpd.leases
```

Tiene que aparecer la IP con la MAC del cliente. Las horas salen en UTC, por eso van dos horas por detrás de la hora local. Con `default-lease-time 600`, el `ends` queda 10 minutos después del `starts`.

Para ver el intercambio en directo, en el servidor:

```bash
sudo tcpdump -i enp0s3 -n -e -v port 67 or port 68
```

y en el cliente vuelvo a hacer el `down` y `up` de arriba. Sin `-v`, tcpdump solo pone `Request` y `Reply` (el código de BOOTP); con `-v` aparece el tipo real de mensaje DHCP en la opción 53.

---

## 5. Volver al modo Puente

El orden importa. Primero paro el servicio y luego apago:

```bash
sudo systemctl stop isc-dhcp-server
sudo poweroff
```

Después pongo el Adaptador 1 en **Puente** y arranco. Nunca arranco el DHCP con el adaptador en Puente: como la subred es la misma que la de la red real, mi servidor repartiría IPs a otros equipos de la clase. Por eso el servicio queda deshabilitado y lo arranco a mano solo cuando verifico que estoy en Red interna.

---

## 6. IP fija en el cliente para poder usar SSH

En clase la red sí tiene DHCP, pero al cliente le dio `172.16.5.152` con una concesión corta y sin gateway ni DNS. Con tantas máquinas virtuales prefiero fijar una IP.

Guardo la configuración de DHCP para la próxima práctica:

```bash
sudo cp /etc/netplan/99-config.yaml ~/99-dhcp.yaml.bak
```

Compruebo desde el servidor (que tiene internet) que la IP elegida está libre:

```bash
sudo apt install arping
sudo arping -I enp0s3 -c 3 172.16.5.211
```

Si nadie responde, la uso. Si responde una MAC que no es la de mi cliente, elijo otra. Y la fijo:

```bash
sudo tee /etc/netplan/99-config.yaml > /dev/null << 'EOF'
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.16.5.211/24
      routes:
        - to: default
          via: 172.16.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
EOF

sudo chmod 600 /etc/netplan/99-config.yaml
sudo netplan apply
ping -c 2 8.8.8.8
```

Para volver a DHCP en el laboratorio: `sudo cp ~/99-dhcp.yaml.bak /etc/netplan/99-config.yaml && sudo netplan apply`.

Con internet ya puedo instalar el SSH en el cliente (`sudo apt install openssh-server`) y conectarme desde mi PC con `ssh antonio@172.16.5.211`.

---

## Lo que me falló y por qué

**El servicio no arrancaba con "No subnet declaration".** El `dhcpd.conf` seguía siendo el de ejemplo y mi bloque `subnet` no se había guardado. `dhcpd -t` no lo detectó porque solo mira la sintaxis.

**El cliente no cogía IP.** Era un clon del servidor y heredó su `99-config.yaml` con IP fija y `dhcp4: no`. Un equipo con IP fija nunca pide DHCP, esté en la red que esté. Además, en netplan el último fichero por orden alfabético es el que gana, así que el `99` pisaba a los demás.

**`ping` al gateway sin respuesta.** El router del colegio bloquea el ICMP hacia sí mismo, pero deja pasar el que va hacia internet. Para comprobar la salida uso `ping 8.8.8.8`, y para ver si el gateway está ahí, `ip neigh show 172.16.0.1` (ARP).

**`nmcli` mostraba el gateway vacío y había internet.** El gateway está definido como ruta por defecto, no en el campo `gateway` del perfil. Se ve con `ip route`.

**El gateway `172.16.0.1` queda fuera de `172.16.5.0/24`.** Por eso en `ip route` aparece una línea extra `172.16.0.1 dev enp0s3 scope link`. Sospecho que la red real es más ancha que `/24`, pero no lo he comprobado.

**`ss -ulpn` enseña `0.0.0.0:67`.** Es normal en dhcpd. La interfaz real de escucha se ve en el `journalctl` con `grep listening`.

**Sospechas equivocadas.** Dos veces pensé que la causa era otra (un DHCP falso en la red y un fallo de VirtualBox). Los logs y el `dhcpd.leases` lo desmintieron. Primero se comprueba, luego se supone.

---

## Desinstalar y empezar de cero

```bash
sudo systemctl stop isc-dhcp-server
sudo apt purge isc-dhcp-server
sudo apt autoremove
sudo rm -f /var/lib/dhcp/dhcpd.leases*
```

`purge` borra también `dhcpd.conf` y `/etc/default/isc-dhcp-server`; con `remove` se quedarían. No borro la carpeta `/etc/dhcp` entera.

---

## Pendiente de probar

**Reserva por MAC**, para que el cliente reciba siempre la misma IP (fuera del `range`):

```
host cliente1 {
    hardware ethernet 08:00:27:7f:40:6f;
    fixed-address 172.16.5.60;
}
```

Esto todavía no lo he ejecutado. Después habría que validar con `dhcpd -t` y reiniciar el servicio.

**Seguridad:** DHCP no autentica, el cliente se fía del primero que responde. De ahí vienen los ataques de *rogue DHCP* (un servidor falso que reparte el gateway del atacante) y *starvation* (agotar el pool con MACs falsas), que se combaten con DHCP snooping y port security en el switch.
