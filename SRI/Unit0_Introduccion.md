# Introducción a los Servicios en Red e Internet

## ¿Qué es un servicio en red?

Un **servicio en red** es una funcionalidad que un equipo (servidor) ofrece a otros equipos (clientes) a través de una red, siguiendo el modelo **cliente-servidor**. El cliente solicita el servicio y el servidor responde a esa petición, generalmente usando un protocolo específico y un puerto determinado.

Ejemplo básico: cuando escribes una URL en el navegador, tu equipo (cliente) solicita una página web a un servidor mediante el protocolo **HTTP**, y el servidor responde enviando el contenido.

## El modelo cliente-servidor

- **Cliente**: equipo que inicia la solicitud (navegador, cliente de correo, terminal SSH...)
- **Servidor**: equipo que escucha peticiones en un puerto concreto y responde
- **Protocolo**: conjunto de reglas que ambos deben seguir para entenderse (HTTP, FTP, SMTP, DNS...)

Un mismo servidor físico puede ofrecer varios servicios simultáneamente, cada uno escuchando en un puerto distinto (ej. HTTP en el 80, HTTPS en el 443, SSH en el 22).

## Principales servicios de red

| Servicio | Protocolo | Puerto | Función |
|---|---|---|---|
| Navegación web | HTTP / HTTPS | 80 / 443 | Servir páginas y aplicaciones web |
| Correo electrónico | SMTP / IMAP / POP3 | 25 / 143 / 110 | Envío y recepción de correo |
| Resolución de nombres | DNS | 53 | Traducir nombres de dominio a IPs |
| Configuración automática de red | DHCP | 67 / 68 | Asignar IP, máscara, puerta de enlace... |
| Transferencia de ficheros | FTP / SFTP | 21 / 22 | Subir y descargar archivos |
| Acceso remoto seguro | SSH | 22 | Terminal remota cifrada |
| Compartición de recursos | SMB / NFS | 445 / 2049 | Compartir carpetas y archivos en red local |

## ¿Qué es Internet?

**Internet** es una red global de redes interconectadas que utiliza el conjunto de protocolos **TCP/IP** para comunicar dispositivos en todo el mundo. No es un único servicio, sino la infraestructura sobre la que funcionan todos los servicios anteriores a escala global.

Elementos clave:
- **IP (Internet Protocol)**: identifica de forma única cada dispositivo en la red
- **TCP**: garantiza que los datos lleguen completos y en orden
- **Routers**: dirigen el tráfico entre redes distintas
- **ISP (proveedor de servicios de Internet)**: conecta las redes locales con la red global

## Por qué importa esto en ASIR

Como administrador de sistemas, no basta con saber *usar* estos servicios: tienes que saber **instalarlos, configurarlos y asegurarlos** en un servidor. Este bloque introductorio es la base sobre la que se construyen los módulos siguientes: DHCP y DNS en profundidad, servidores web (Apache/Nginx), correo, y seguridad perimetral (firewalls, VPN).
