# Seguridad y Alta Disponibilidad — Fundamentos de la seguridad informática

Apuntes del temario de Moodle, reorganizados en un único documento. La numeración respeta la del profesor para poder cruzar los apuntes con el aula virtual. Al final hay un apartado con matices y correcciones que he detectado en el material original; lo que aparece marcado como *(añadido)* no está en el temario y lo he puesto para completar.

**Índice**

1. Fiabilidad, confidencialidad, integridad y disponibilidad
2. Elementos vulnerables: hardware, software y datos
3. Análisis de las principales vulnerabilidades
4. Amenazas y sus tipos
5. Seguridad física y ambiental
6. Hilo conductor del tema
7. Matices y correcciones sobre el material original

---

## 1. Fiabilidad, confidencialidad, integridad y disponibilidad

Toda la asignatura gira alrededor de cuatro propiedades que queremos garantizar en un sistema de información (SI). Conviene tenerlas muy claras porque cada vulnerabilidad, amenaza o medida de protección que veamos después se explica diciendo cuál de ellas afecta o protege.

La **fiabilidad** es la capacidad de conseguir que un SI ofrezca la información sin pausas entre peticiones, es decir, que funcione de forma correcta y constante. La **confidencialidad** es la capacidad de conseguir que la información se muestre únicamente a las personas autorizadas. La **integridad** es la capacidad de conseguir que la información no se altere. El temario la define pensando en alteraciones por causas involuntarias, pero en la práctica también abarca las modificaciones no autorizadas o maliciosas; de hecho, más adelante el propio temario habla de modificación «no autorizada o accidental». La **disponibilidad** es la capacidad de respuesta a las peticiones con las mínimas pausas posibles.

### Medir la disponibilidad: los «nueves»

La disponibilidad se mide en «nueves», que expresan el porcentaje de tiempo que el sistema está operativo. Un SI con «2 nueves» está disponible el 99 % del tiempo, con «3 nueves» el 99,9 %, con «4 nueves» el 99,99 % y con «5 nueves» el 99,999 %. Cada nueve adicional reduce drásticamente el tiempo de parada tolerable, y también encarece de forma notable la infraestructura necesaria.

*(añadido)* Para hacerse una idea real de lo que significa cada nivel, estas son las paradas máximas anuales aproximadas:

| Disponibilidad | Nombre | Parada máxima al año |
|---|---|---|
| 99 % | 2 nueves | unos 3,65 días |
| 99,9 % | 3 nueves | unas 8 h 45 min |
| 99,99 % | 4 nueves | unos 52 min |
| 99,999 % | 5 nueves | unos 5 min |

### Los cuatro niveles TIER

Este nivel de tolerancia se relaciona con los cuatro niveles TIER por los que se clasifican los centros de datos (CPD) según su capacidad para tolerar fallos e interrupciones.

**TIER I — Infraestructura básica.** Es el nivel más sencillo. El centro de datos dispone de lo necesario para mantener los sistemas funcionando, pero sin una infraestructura redundante importante: una única vía de alimentación eléctrica, una única vía de distribución de refrigeración y pocos o ningún componente redundante. El mantenimiento de ciertos elementos puede obligar a detener los sistemas, y el centro queda muy expuesto a interrupciones por averías o por trabajos programados. Un ejemplo sería el pequeño CPD de una empresa con varios servidores, un SAI y climatización, pero nada duplicado. La idea es: *«tengo lo necesario para funcionar, pero si algo importante falla, puedo tener que parar»*.

**TIER II — Componentes redundantes.** Aquí empiezan a aparecer elementos de reserva: SAI redundantes, generadores, equipos de refrigeración adicionales y componentes eléctricos y mecánicos duplicados. Sin embargo, normalmente sigue existiendo una única vía de distribución. Esto es importante: tener un componente duplicado no significa que todo el sistema aguante cualquier fallo. La idea es: *«tengo componentes de reserva, pero no todo está duplicado»*.

**TIER III — Mantenimiento simultáneo.** Es un salto importante. Un centro TIER III está diseñado para poder hacer mantenimiento de determinados componentes sin apagar los sistemas TI, gracias a que dispone de varias vías de distribución para los sistemas críticos y de componentes redundantes. Si hay que intervenir en una de las ramas, la otra sigue dando servicio:

```
                DATA CENTER TIER III

              ┌──────────────────┐
              │    SERVIDORES    │
              └────────┬─────────┘
                       │
                ┌──────┴──────┐
                │             │
             RED A          RED B
                │             │
          ┌─────┴─────┐ ┌─────┴─────┐
          │   UPS A   │ │   UPS B   │
          └───────────┘ └───────────┘
```

La idea es: *«puedo mantener parte de la infraestructura sin apagar el CPD»*. Este nivel es especialmente útil para explicar la alta disponibilidad.

**TIER IV — Tolerancia a fallos.** Es el nivel más exigente. El centro se diseña con múltiples sistemas independientes y redundantes, de modo que el fallo de determinados componentes o sistemas no debería interrumpir los servicios críticos. Combina alimentación eléctrica y refrigeración redundantes, varias vías de distribución, mayor separación e independencia entre sistemas, capacidad de mantenimiento sin interrupción y tolerancia frente a ciertos fallos individuales. La idea es: *«si falla un componente crítico, el sistema debe poder continuar funcionando»*. Se utiliza cuando una interrupción del servicio puede tener consecuencias muy graves.

*(añadido)* Los valores de disponibilidad que suelen asociarse a cada nivel según el Uptime Institute son aproximadamente 99,671 % (I), 99,741 % (II), 99,982 % (III) y 99,995 % (IV). Son cifras orientativas, útiles para relacionar los TIER con los «nueves».

---

## 2. Elementos vulnerables en el sistema informático: hardware, software y datos

### 2.1. Concepto de vulnerabilidad

Una **vulnerabilidad** es una debilidad o defecto de un sistema que puede ser aprovechado para provocar un fallo, comprometer la seguridad o permitir que una amenaza produzca un daño. Dicho de forma corta, es un punto débil que puede ser aprovechado por una amenaza.

Hay cinco conceptos que se confunden con facilidad y que hay que distinguir bien:

| Concepto | Definición | Ejemplo |
|---|---|---|
| Vulnerabilidad | Debilidad del sistema | Sistema operativo sin actualizar |
| Amenaza | Circunstancia o agente que puede provocar un daño | Malware |
| Ataque | Acción que aprovecha una vulnerabilidad | Explotarla para instalar malware |
| Riesgo | Posibilidad de que una amenaza aproveche una vulnerabilidad y provoque un daño | Pérdida de información |
| Impacto | Consecuencia producida por el incidente | Pérdida económica o interrupción del servicio |

Un ejemplo completo: un servidor tiene un servicio vulnerable sin parchear (vulnerabilidad); un atacante se interesa por él (amenaza); explota el fallo (ataque); el servidor queda comprometido (incidente); y como consecuencia se pierden o modifican datos (impacto).

```
Vulnerabilidad: servicio sin parchear
        ↓
Amenaza: atacante
        ↓
Ataque: explotación de la vulnerabilidad
        ↓
Incidente: servidor comprometido
        ↓
Impacto: pérdida o modificación de datos
```

Lo importante de esta cadena es que una vulnerabilidad por sí sola no implica que vaya a producirse un incidente, pero sí aumenta el riesgo de que ocurra.

### 2.2. Vulnerabilidades del hardware

El hardware son los componentes físicos del sistema. Solemos asociar la seguridad al software, pero un fallo físico también puede causar pérdida de disponibilidad e incluso de datos.

**Alimentación eléctrica.** Los equipos necesitan una alimentación estable. Las alteraciones eléctricas pueden provocar apagados inesperados, pérdida de información, corrupción del sistema de archivos, daños en componentes e interrupción del servicio. Las medidas de protección incluyen SAI, fuentes de alimentación redundantes, grupos electrógenos, protección contra sobretensiones y líneas eléctricas redundantes. Esto enlaza directamente con la alta disponibilidad vista en el apartado 1.

**Placa base.** Una avería en la placa puede dejar un servidor completamente inoperativo, y algunos fallos son intermitentes y difíciles de diagnosticar. Se mitiga con mantenimiento preventivo, monitorización del hardware, componentes de calidad y equipos o piezas de sustitución.

**Memoria RAM.** Los errores de memoria producen bloqueos, reinicios, errores del sistema operativo, corrupción de datos y comportamientos aparentemente aleatorios. Son especialmente molestos porque pueden ser intermitentes y difíciles de reproducir. En servidores existe la tecnología **ECC** (*Error-Correcting Code*), capaz de detectar y corregir determinados errores de memoria.

**Discos y almacenamiento.** Son especialmente importantes porque contienen los datos. Pueden aparecer sectores defectuosos, fallos mecánicos, desgaste en las unidades SSD, corrupción de datos o la pérdida completa de la unidad. Aquí aparece un concepto fundamental de ASIR: **RAID**, que permite combinar varias unidades para aumentar el rendimiento, proporcionar redundancia, mejorar la disponibilidad y reducir el impacto de ciertos fallos de disco. Por ejemplo, en RAID 1 los datos se copian en dos discos, de forma que si uno falla el otro conserva una copia completa.

```
        DATOS
          │
       ┌──┴──┐
       ▼     ▼
     DISCO  DISCO
       A     B
       │     │
       └COPIA┘
```

> ⚠️ **RAID no sustituye a las copias de seguridad.** Protege sobre todo frente a fallos de almacenamiento, mientras que una copia de seguridad permite recuperar información borrada, modificada, cifrada por ransomware, etc.

**Tarjetas de expansión.** Tarjetas de red, controladoras de almacenamiento, tarjetas gráficas y similares también pueden fallar. En servidores críticos se recurre a interfaces de red redundantes, varias controladoras, fuentes redundantes y componentes intercambiables en caliente (*hot swap*).

**Interconexiones.** Cables, conectores y soldaduras también son puntos de fallo. Si un servidor se conecta a un switch mediante un único cable y ese cable falla, se pierde el servicio. Esa situación es un **SPOF** (*Single Point of Failure*, punto único de fallo), y se reduce utilizando varias interfaces y caminos de red.

### 2.3. Vulnerabilidades del software

El software es otro de los grandes elementos vulnerables. Puede haber vulnerabilidades en sistemas operativos, aplicaciones, servicios de red, controladores, firmware, bibliotecas y componentes de terceros, y también en configuraciones incorrectas.

**Sistema operativo.** Puede contener fallos que permitan ejecutar código no autorizado, obtener privilegios, acceder a información o provocar una denegación de servicio. Por eso es fundamental mantenerlo actualizado con parches de seguridad. Pero en sistemas críticos hay que tener cuidado, porque una actualización puede corregir una vulnerabilidad y a la vez introducir incompatibilidades o problemas con ciertas aplicaciones. En entornos profesionales lo habitual es seguir el circuito *entorno de pruebas → validación → producción* en lugar de instalar el parche de golpe en todos los sistemas.

### 2.4. Aplicaciones

Las aplicaciones también pueden ser vulnerables por errores de programación, validación incorrecta de los datos de entrada, contraseñas almacenadas de forma insegura, permisos incorrectos, desbordamientos de memoria, configuraciones inseguras o uso de componentes vulnerables. Un atacante puede aprovechar estos problemas para acceder a información o ejecutar acciones no autorizadas.

Por ejemplo, si una aplicación web recibe un campo `usuario = "admin"` y no valida correctamente lo que introduce el usuario, podría ser susceptible a determinados ataques. Por eso son importantes las actualizaciones, el desarrollo seguro, las auditorías, las pruebas de seguridad, la configuración segura y el **principio de mínimo privilegio**.

### 2.5. Firmware

El firmware es el software integrado en dispositivos hardware que controla su funcionamiento. Lo encontramos en routers, switches, tarjetas de red, discos, placas base, dispositivos de almacenamiento y dispositivos IoT, y también puede contener vulnerabilidades. La consecuencia práctica es que no basta con actualizar el sistema operativo: cuando corresponda, hay que mantener actualizados también los dispositivos y su firmware.

### 2.6. La configuración como vulnerabilidad

Un sistema puede tener todo el software perfectamente actualizado y ser vulnerable igualmente por una mala configuración: servicios innecesarios activados, puertos abiertos sin necesidad, contraseñas débiles, permisos excesivos, cuentas que ya no se usan, recursos compartidos sin protección, configuraciones por defecto o ausencia de cifrado. De aquí sale un principio clave: **reducir la superficie de ataque**, es decir, mantener activos únicamente los servicios y accesos necesarios.

### 2.7. Vulnerabilidades de los datos

Los datos son uno de los activos más importantes de una organización. Un sistema puede estar perfectamente operativo y sufrir un incidente grave si los datos se pierden, se modifican, se roban, quedan inaccesibles o se destruyen. Esto se relaciona directamente con las propiedades del apartado 1: la confidencialidad responde a *¿quién puede acceder?*, la integridad a *¿se han alterado?* y la disponibilidad a *¿podemos acceder cuando los necesitamos?*.

```
                 DATOS
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
 CONFIDENCIALIDAD INTEGRIDAD DISPONIBILIDAD
        │          │          │
   ¿Quién puede   ¿Se han    ¿Podemos
    acceder?      alterado?   acceder?
```

### 2.8. Principales amenazas contra los datos

Los datos pueden verse afectados por fallos de hardware (un disco que se avería), errores humanos (un administrador que borra por accidente un archivo o una base de datos), malware (que puede modificar, destruir o cifrar información), ransomware (el atacante cifra los datos y exige un rescate), robo de información confidencial, catástrofes (incendios, inundaciones) y fallos de software que corrompen, por ejemplo, una base de datos.

### 2.9. Medidas para proteger los datos

Aquí aparece la **defensa en profundidad**: no hay que confiar en una única medida de seguridad, sino combinar varias capas. Entre ellas están el control de acceso, la autenticación, la autorización, el cifrado, las copias de seguridad, RAID, la redundancia, el antivirus o EDR, el firewall, la segmentación de red, la monitorización, las actualizaciones y los planes de recuperación ante desastres.

```
              DATOS
                │
        ┌───────┴────────┐
        │                │
     PERMISOS          CIFRADO
        │                │
    FIREWALL          BACKUPS
        │                │
   MONITORIZACIÓN    REDUNDANCIA
        │                │
        └───────┬────────┘
                │
          DEFENSA EN
           PROFUNDIDAD
```

### 2.10. Vulnerabilidad no es lo mismo que fallo

Hay que evitar confundir una vulnerabilidad con un fallo. Una definición como «medida de la capacidad de un sistema para fallar de manera inesperada» lleva a esa confusión. La formulación más correcta es:

> Una vulnerabilidad es una debilidad o defecto de un sistema que puede ser aprovechado por una amenaza y que puede comprometer su confidencialidad, integridad o disponibilidad.

Así se separan bien los conceptos: la vulnerabilidad es la *debilidad*; la amenaza es lo que *puede aprovecharla*; el ataque es la acción que *la explota*; el incidente es cuando *se materializa* el ataque o el fallo; y el impacto son las *consecuencias* producidas.

---

## 3. Análisis de las principales vulnerabilidades de un sistema informático

### 3.1. Alcance de una vulnerabilidad

Retomando la definición del apartado 2.1, una vulnerabilidad puede afectar a hardware, sistemas operativos, aplicaciones, servicios de red, firmware, configuraciones, protocolos, y a los datos y sus mecanismos de protección. Además, puede afectar a uno o varios principios de seguridad: a la confidencialidad si permite acceder a información sin autorización, a la integridad si permite modificarla o destruirla, y a la disponibilidad si puede hacer que un servicio deje de funcionar. Por ejemplo, una vulnerabilidad en un servidor web que permita acceder a información de los usuarios compromete principalmente la confidencialidad.

### 3.2. Consecuencias de una vulnerabilidad

Una vulnerabilidad no implica que el sistema ya haya sido atacado, pero ofrece una vía para que una amenaza produzca un incidente. Las consecuencias más importantes son estas.

**Escalada de privilegios.** Un usuario con permisos limitados consigue permisos superiores explotando una vulnerabilidad (de usuario normal a administrador o root). Es especialmente peligrosa porque el atacante podría controlar prácticamente todo el sistema.

**Ejecución de código.** El atacante consigue ejecutar código en el equipo afectado y, según el contexto, podría instalar malware, crear usuarios, modificar configuraciones, acceder a información o usar el equipo para atacar otros sistemas.

**Robo de información.** Puede afectar a contraseñas, información personal, bases de datos, documentos, información financiera o claves criptográficas. Se compromete sobre todo la confidencialidad.

**Modificación o destrucción de información.** Afecta principalmente a la integridad.

**Denegación de servicio.** Una vulnerabilidad puede provocar que un servicio se bloquee y deje de estar disponible, lo que compromete la disponibilidad y conecta con el apartado 1.

### 3.3. Ciclo de vida de una vulnerabilidad

Una vulnerabilidad suele pasar por estas etapas: descubrimiento, análisis, publicación, identificación, evaluación del riesgo, corrección o mitigación, actualización y verificación.

```
DESCUBRIMIENTO → ANÁLISIS → PUBLICACIÓN → IDENTIFICACIÓN
      → EVALUACIÓN DEL RIESGO → CORRECCIÓN / MITIGACIÓN
      → ACTUALIZACIÓN → VERIFICACIÓN
```

Un caso típico sería: se descubre una vulnerabilidad en un servidor web, se analiza, se publica un aviso de seguridad, se le asigna un identificador CVE, se determina su gravedad, el fabricante publica un parche, el administrador actualiza el servidor y se comprueba que la vulnerabilidad ya no está presente. Esto nos introduce en la **gestión de vulnerabilidades**.

### 3.4. CVE — Common Vulnerabilities and Exposures

El programa CVE proporciona identificadores comunes para vulnerabilidades de ciberseguridad divulgadas públicamente. Su objetivo es que fabricantes, investigadores, administradores y herramientas de seguridad se refieran a una misma vulnerabilidad con un identificador común. El formato es `CVE-AÑO-NÚMERO`, por ejemplo `CVE-2024-12345`, donde CVE identifica el sistema de identificación, 2024 es el año asociado al identificador y 12345 es el número.

¿Para qué sirve? Si un fabricante, un administrador y una herramienta hablan de «la vulnerabilidad de Apache que permite…», el mensaje es ambiguo. Si todos usan el mismo CVE, están hablando de forma inequívoca de la misma vulnerabilidad. En resumen, **CVE proporciona un lenguaje común para identificar vulnerabilidades**.

### 3.5. CVE no es lo mismo que NVD

El **CVE Program** identifica y cataloga vulnerabilidades conocidas públicamente mediante registros CVE, que publican organizaciones autorizadas llamadas **CNA** (*CVE Numbering Authorities*). La **NVD** (*National Vulnerability Database*) está gestionada por el NIST y añade información complementaria: productos afectados, métricas de impacto y otros datos. Son programas separados, y la lista CVE alimenta a la NVD.

```
      CVE  →  identificación de la vulnerabilidad
       │
       ▼
      NVD  →  información adicional y métricas
       │
       ▼
  Gestión de vulnerabilidades
```

Como fuentes donde consultar vulnerabilidades, el profesor recomienda **Tenable** e **INCIBE**.

### 3.6. CVSS — medir la gravedad

Una vez identificada una vulnerabilidad, la pregunta siguiente es qué importancia tiene. Para eso existe **CVSS** (*Common Vulnerability Scoring System*), un estándar independiente del programa CVE que calcula la gravedad mediante una puntuación numérica y la asocia a una categoría cualitativa (baja, media, alta o crítica). La versión actual es CVSS 4.0, aunque las versiones anteriores todavía aparecen en documentación y herramientas, y la NVD ofrece soporte para la 4.0. La puntuación ayuda a las organizaciones a **priorizar** qué corregir primero.

```
VULNERABILIDAD → CVE → CVSS → GRAVEDAD / SEVERIDAD
```

Por ejemplo, con tres vulnerabilidades: la A tiene un CVSS de 2,1 (prioridad baja), la B de 6,5 (media) y la C de 9,8 (crítica). La idea que hay que llevarse es: **CVE identifica; CVSS ayuda a valorar la gravedad y priorizar**. No deben confundirse.

### 3.7. CWE — Common Weakness Enumeration

Mientras que CVE identifica una vulnerabilidad concreta, **CWE** describe *tipos* de debilidades de seguridad que pueden originar vulnerabilidades: debilidades de programación, control de acceso incorrecto, validación incorrecta, gestión incorrecta de memoria, etc. El programa CVE lo describe como una taxonomía de debilidades comunes de software y hardware que sirve de lenguaje común para identificarlas, prevenirlas y mitigarlas. Una forma sencilla de recordarlo: **CWE describe el tipo de debilidad; CVE identifica una vulnerabilidad concreta**.

```
CWE (tipo de debilidad) → Vulnerabilidad → CVE (caso concreto)
```

El profesor enlaza además el listado *Top CWE 2025* como material de consulta.

### 3.8. CPE — Common Platform Enumeration

**CPE** proporciona una forma estructurada de identificar productos, sistemas y componentes tecnológicos. Es muy útil para responder a la pregunta «¿qué equipos y versiones están afectados por esta vulnerabilidad?». El razonamiento sería: dado un CVE, se consultan los productos y versiones afectados, se comprueba si tenemos esa versión y, si es así, se decide aplicar el parche. Los registros CVE pueden incluir información CPE, y la NVD la utiliza para representar configuraciones y productos afectados.

### 3.9. Relación entre CVE, CVSS, CWE y CPE

```
                     VULNERABILIDAD
                            │
              ┌─────────────┼─────────────┐
              │             │             │
             CVE           CWE           CPE
              │             │             │
        ¿Cuál es?      ¿Qué tipo de   ¿Qué producto
                       debilidad?      está afectado?
              │
              ▼
            CVSS
              │
        ¿Qué gravedad tiene?
```

En una frase: CVE identifica la vulnerabilidad, CWE el tipo de debilidad, CPE los productos o plataformas afectados y CVSS ayuda a valorar la gravedad.

### 3.10. Gestión de vulnerabilidades

En un entorno profesional no basta con conocer las vulnerabilidades: hay que gestionarlas, y el proceso básico tiene seis pasos.

El primero es la **identificación de activos**: saber qué tenemos (servidores, PCs, routers, switches, sistemas operativos, aplicaciones, bases de datos, dispositivos IoT). El segundo es el **descubrimiento de vulnerabilidades**, mediante avisos de seguridad, bases de datos de vulnerabilidades, herramientas de análisis y escáneres. El tercero es la **evaluación**: determinar qué vulnerabilidades existen, qué sistemas afectan, qué gravedad tienen y qué impacto podrían producir. El cuarto es la **priorización**, porque no siempre se puede solucionar todo a la vez, y se tiene en cuenta la gravedad, la exposición, la criticidad del activo, la existencia de exploits, el impacto potencial y la disponibilidad de parche. El quinto es la **remediación**: instalar parches, actualizar software, modificar configuraciones, desactivar servicios, sustituir componentes o aplicar medidas de mitigación. El sexto es la **verificación**: comprobar que la vulnerabilidad ya no está presente, que no se han generado nuevos problemas y que el sistema sigue funcionando correctamente.

### 3.11. Escáneres de vulnerabilidades

Un escáner de vulnerabilidades analiza sistemas (servidores, PCs, routers…) en busca de vulnerabilidades conocidas y genera un informe que puede indicar la vulnerabilidad detectada, el CVE asociado, el producto y la versión afectados, la gravedad y recomendaciones de corrección. Esto permite pasar de una seguridad reactiva a una gestión más sistemática.

### 3.12. Ejemplo completo

Una empresa tiene un servidor web con una versión vulnerable de una aplicación. El proceso sería: se identifica el CVE, se consulta su CVSS, se determina la gravedad, se comprueba si el servidor está realmente afectado, se instala el parche, se vuelve a analizar y se confirma que la vulnerabilidad está corregida. Todo ello recupera la idea central: una vulnerabilidad puede comprometer la confidencialidad, la integridad o la disponibilidad, por lo que el análisis de vulnerabilidades es fundamental para mantener la seguridad y la alta disponibilidad.

---

## 4. Amenazas y sus tipos

### 4.1. Concepto de amenaza

Una **amenaza** es una circunstancia, agente o acontecimiento que puede provocar un daño a un sistema informático o comprometer su seguridad. Puede afectar a la confidencialidad (acceso no autorizado), la integridad (modificación o destrucción no autorizada), la disponibilidad (interrupción o degradación de un servicio) o la fiabilidad (funcionamiento incorrecto o inesperado). Una amenaza no tiene por qué ser intencionada ni proceder de una persona: un incendio, una inundación, un fallo eléctrico o un error humano también lo son.

### 4.2. Clasificación de las amenazas

Una primera clasificación, muy útil en administración de sistemas, distingue entre **amenazas físicas**, que afectan a los equipos, las instalaciones o el entorno, y **amenazas lógicas**, que afectan al software, los datos, las comunicaciones o los mecanismos de acceso. Según su origen se dividen en **naturales** (fenómenos naturales), **accidentales** (producidas de forma involuntaria) e **intencionadas** (provocadas deliberadamente). Una misma amenaza puede pertenecer a varias categorías: un empleado que borra por error una base de datos es una amenaza interna y accidental.

### 4.3. Amenazas físicas

**4.3.1. Robo de equipos.** El robo de un ordenador, servidor, disco u otro dispositivo puede causar pérdida del equipo, interrupción del servicio, pérdida de información y acceso no autorizado a los datos. Es especialmente grave si el dispositivo contiene información confidencial sin cifrar. Se mitiga con control de acceso físico, cámaras, armarios y racks cerrados, sistemas antirrobo, cifrado de los dispositivos de almacenamiento e inventario y control de activos.

**4.3.2. Incendios e inundaciones.** Pueden destruir servidores, equipos de red, sistemas de almacenamiento, cableado e instalaciones eléctricas, y en un centro de datos un único incidente puede afectar a muchísimos sistemas a la vez. Las medidas incluyen detección de incendios, sistemas de extinción, sensores de agua, climatización, control de humedad, ubicación adecuada de los equipos y centros de respaldo, complementadas con planes de recuperación ante desastres.

**4.3.3. Fallos eléctricos.** Cortes y alteraciones del suministro pueden provocar apagados inesperados, pérdida de información, corrupción de sistemas de archivos, interrupción de servicios y daños en componentes. Para reducir sus efectos se usan el **SAI**, que da alimentación temporal ante una interrupción y protege frente a ciertas alteraciones; los **generadores**, que mantienen la alimentación en cortes prolongados; y las **fuentes de alimentación redundantes**, que permiten que un servidor siga funcionando si una falla. Se relaciona con la redundancia, la alta disponibilidad y los TIER.

**4.3.4. Acceso físico no autorizado.** Quien accede físicamente a una sala de servidores puede desconectar equipos, manipular cableado, robar dispositivos, extraer discos, instalar dispositivos no autorizados, modificar configuraciones o provocar daños. La seguridad informática debe contemplar también la seguridad física, con cerraduras, tarjetas de acceso, biometría, cámaras, vigilancia, registro de accesos, control de visitantes y racks cerrados.

**4.3.5. Temperatura y condiciones ambientales.** Los equipos generan calor, y una temperatura excesiva reduce el rendimiento, causa errores y apagados, daña componentes y acorta su vida útil. También afectan negativamente la humedad excesiva, el polvo, el humo y el agua. Los centros de datos usan climatización y sensores que generan alertas cuando se superan ciertos valores.

**4.3.6. Averías de hardware.** Discos, RAM, fuentes, placas base, tarjetas de red, controladoras y ventiladores pueden fallar, y las consecuencias van desde una pérdida de rendimiento hasta la caída completa de un servicio. Se reducen con componentes redundantes, RAID, fuentes redundantes, dispositivos hot swap, monitorización, mantenimiento preventivo y equipos de sustitución.

### 4.4. Amenazas lógicas

**4.4.1. Malware.** Es el software diseñado para realizar acciones maliciosas o no autorizadas. Los tipos principales son el **virus** (se propaga incorporándose a otros archivos o programas), el **gusano** o *worm* (se propaga automáticamente por redes o sistemas), el **troyano** (aparenta ser legítimo pero incorpora funciones maliciosas), el **ransomware** (cifra información y exige un rescate), el **spyware** (recopila información del usuario o del sistema), el **keylogger** (registra las pulsaciones del teclado) y el **bot** (permite controlar un equipo de forma remota y usarlo como parte de una botnet). El malware puede comprometer la confidencialidad, la integridad y la disponibilidad.

**4.4.2. Phishing.** Utiliza técnicas de engaño para que una persona revele información o realice una acción que favorece al atacante. Puede llegar por correo electrónico, páginas web falsas, SMS, aplicaciones de mensajería o redes sociales, y busca conseguir nombres de usuario, contraseñas, información personal, datos bancarios o códigos de autenticación. El esquema típico es: mensaje aparentemente legítimo, enlace fraudulento, página web falsa y robo de credenciales. Demuestra que el usuario también forma parte de la superficie de seguridad del sistema.

**4.4.3. Robo de credenciales.** Las credenciales autentican a un usuario frente a un sistema, y el atacante puede intentar obtener nombres de usuario, contraseñas, claves privadas, tokens, cookies de sesión o códigos de autenticación. Las técnicas más habituales son el phishing, la fuerza bruta, el malware, la ingeniería social, la reutilización de contraseñas y el aprovechamiento de filtraciones. Se contrarresta con contraseñas robustas, autenticación multifactor (MFA), gestores de contraseñas, bloqueo tras múltiples intentos, políticas de contraseñas y el principio de mínimo privilegio.

**4.4.4. Explotación de vulnerabilidades.** Una amenaza aprovecha una vulnerabilidad para realizar un ataque: sistema sin actualizar, atacante, explotación, acceso al sistema y compromiso. Por eso son fundamentales la actualización de sistemas, la instalación de parches, el análisis de vulnerabilidades, la eliminación de servicios innecesarios, la monitorización y la configuración segura.

*Práctica del temario: averiguar la versión de un servidor web.* Se abre la web en el navegador, se pulsa F12 (o clic derecho e Inspeccionar), se va a la pestaña Red (*Network*) y se recarga la página con F5 para capturar las peticiones. Se hace clic en la primera petición de la lista (suele llevar el nombre del dominio o de `index.php`) y, en el panel derecho, se busca la sección de cabeceras de respuesta (*Response Headers*). Allí aparece el campo `Server`, que muestra si Apache sigue exponiendo su versión o si ya se ha ocultado. Una forma más rápida es hacer una petición con `curl -I https://tudominio.com`. Si el servidor no tiene la opción correspondiente para ocultar la información, ya tenemos la versión del servidor web, y con ella podemos buscar los CVE de Nginx o de Apache que le afectan. (Ver el apartado 7 para un matiz importante sobre esta práctica.)

**4.4.5. Acceso no autorizado.** Se produce cuando alguien accede a un sistema, recurso o información sin los permisos necesarios, por robo de credenciales, contraseñas débiles, explotación de vulnerabilidades, errores de configuración o escalada de privilegios. Para controlar los accesos se usan tres conceptos fundamentales: la **autenticación** (¿quién eres?), la **autorización** (¿qué puedes hacer?) y la **auditoría** (¿qué has hecho?).

**4.4.6. Ataques DoS y DDoS.** Un ataque **DoS** (*Denial of Service*) busca que un sistema o servicio deje de estar disponible o funcione de forma degradada. En un ataque **DDoS** (*Distributed*) el tráfico malicioso procede de múltiples sistemas a la vez, que saturan al objetivo.

```
   EQUIPO ─┐
   EQUIPO ─┤
   EQUIPO ─┼──────────► SERVIDOR  →  SOBRECARGA
   EQUIPO ─┤
   EQUIPO ─┘
```

El atacante puede intentar agotar el ancho de banda, la CPU, la memoria, las conexiones o los recursos de una aplicación. El principio de seguridad más afectado es la **disponibilidad**, lo que enlaza con la alta disponibilidad del apartado 1.

**4.4.7. Modificación o borrado malicioso de datos.** El atacante puede eliminar información, modificar archivos, alterar bases de datos, cifrar información o destruir copias accesibles. Se afecta la integridad si los datos se modifican, la disponibilidad si dejan de estar accesibles y la confidencialidad si además el atacante los obtiene. Las medidas son control de acceso, permisos, copias de seguridad, cifrado, monitorización, sistemas de detección y mecanismos de recuperación.

**4.4.8. Errores de configuración.** Puertos abiertos sin necesidad, servicios que no se usan, contraseñas por defecto, permisos excesivos, recursos compartidos sin protección, reglas incorrectas en un firewall o ausencia de cifrado convierten una configuración en una vulnerabilidad. Por eso la configuración segura es parte esencial de la administración de sistemas.

### 4.5. Amenazas según su origen

Las **naturales** las provocan fenómenos como inundaciones, terremotos, tormentas, incendios o temperaturas extremas. Las **accidentales** son involuntarias: eliminación accidental de archivos, error de configuración, fallo eléctrico, caída de un equipo o un cable desconectado. Las **intencionadas** se provocan deliberadamente para obtener información, causar daños o interrumpir servicios: robo, malware, phishing, DDoS, sabotaje o acceso no autorizado.

### 4.6. El factor humano como amenaza

Usuarios y administradores pueden provocar incidentes de forma accidental, negligente o intencionada: usar contraseñas débiles, compartir credenciales, abrir archivos maliciosos, instalar software no autorizado, borrar información, configurar mal un servidor o dar información confidencial a quien no debe. Por eso la seguridad debe combinar medidas técnicas, organizativas y formativas: formación y concienciación, políticas de seguridad, procedimientos, mínimo privilegio, separación de funciones, auditorías y control de accesos.

### 4.7. Relación entre activo, vulnerabilidad, amenaza, ataque, incidente, impacto y riesgo

```
     ACTIVO
        │
        ▼
  VULNERABILIDAD
        │  puede ser aprovechada por
        ▼
    AMENAZA
        │
        ▼
     ATAQUE
        │
        ▼
    INCIDENTE
        │
        ▼
    IMPACTO
        │
        ▼
     RIESGO
```

Ejemplo: un servidor web utiliza una versión de software sin actualizar, y un atacante lo aprovecha para acceder y obtener información de los clientes. El **activo** es el servidor y los datos de clientes; la **vulnerabilidad** es el software sin actualizar; la **amenaza** es el atacante; el **ataque** es la explotación; el **incidente** es el acceso no autorizado; el **impacto** es la pérdida de confidencialidad; y el **riesgo** es la posibilidad de que esta situación produzca consecuencias negativas.

### 4.8. Medidas frente a las amenazas

Las medidas actúan en cuatro momentos distintos. La **prevención** intenta evitar que el incidente llegue a producirse (firewall, antivirus/EDR, actualizaciones, control de acceso, contraseñas robustas, formación, redundancia). La **detección** permite saber que un incidente se está produciendo o ya se produjo (monitorización, IDS/IPS, logs, alertas, sensores ambientales). La **respuesta** consiste en actuar durante el incidente (aislar un equipo, bloquear una cuenta o una IP, detener un servicio comprometido, activar procedimientos de emergencia). La **recuperación** restaura el funcionamiento normal (copias de seguridad, RAID, sistemas redundantes, servidores de respaldo, recuperación ante desastres).

### 4.9. Resumen del apartado

| Concepto | Significado | Ejemplo |
|---|---|---|
| Amenaza física | Puede producir daños físicos | Incendio |
| Amenaza lógica | Afecta al software, datos o accesos | Malware |
| Amenaza natural | Proviene de un fenómeno natural | Inundación |
| Amenaza accidental | Se produce involuntariamente | Borrado accidental |
| Amenaza intencionada | Se produce deliberadamente | DDoS |
| Vulnerabilidad | Debilidad del sistema | Software sin parchear |
| Ataque | Acción para aprovechar una vulnerabilidad | Explotación |
| Incidente | Materialización de un problema de seguridad | Servidor comprometido |
| Impacto | Consecuencia del incidente | Robo de datos |
| Riesgo | Posibilidad de que ocurra y produzca consecuencias | Pérdida económica |

---

## 5. Seguridad física y ambiental

La seguridad física y ambiental es el conjunto de medidas que protegen los equipos informáticos, las instalaciones, las personas, los soportes de almacenamiento y la información frente a robos, incendios, inundaciones, fallos eléctricos, temperaturas inadecuadas o accesos físicos no autorizados. Protege sobre todo la **disponibilidad** (sistemas operativos y accesibles), la **integridad** (evitar daños o modificaciones en equipos e información) y la **confidencialidad** (impedir que personas no autorizadas accedan físicamente a los sistemas o a los datos).

> **Idea clave:** la seguridad informática no empieza en una contraseña o en un firewall. Si alguien puede acceder físicamente a un servidor, desconectarlo, robar sus discos o destruirlo, muchas de las medidas de seguridad lógica quedan inutilizadas.

La relación de fondo es: amenaza física → daño físico → interrupción o pérdida de información → impacto sobre la seguridad.

### 5.1. Controles físicos

Los controles físicos impiden o limitan el acceso de personas no autorizadas a instalaciones, equipos y soportes. El objetivo no es solo evitar robos: un acceso no autorizado permite extraer discos, desconectar servidores, manipular cableado, conectar dispositivos no autorizados, modificar configuraciones, instalar dispositivos maliciosos, destruir equipos u obtener información almacenada localmente. Por eso el control de acceso físico es la primera barrera de seguridad.

**5.1.1. Cerraduras y puertas de seguridad.** Las salas con sistemas críticos deben dificultar el acceso: cerraduras convencionales o electrónicas, puertas reforzadas, sistemas de control de acceso y apertura mediante tarjeta o credencial. En un CPD no debería poder entrar libremente cualquier trabajador.

**5.1.2. Identificación y autenticación.** El acceso se controla con tarjetas de proximidad, códigos PIN, biometría, credenciales electrónicas o combinaciones de varios factores; por ejemplo, tarjeta y código personal para una sala especialmente protegida. Además, permite registrar quién accedió y cuándo.

**5.1.3. Videovigilancia.** Las cámaras permiten supervisar las instalaciones, detectar accesos no autorizados, investigar incidentes y registrar movimientos en zonas críticas. Complementa al control de acceso, pero no lo sustituye.

**5.1.4. Registro de visitantes.** En instalaciones críticas conviene controlar las visitas: identificar al visitante, registrar la entrada, designar a un responsable, acompañarlo en las zonas restringidas y registrar la salida. Es especialmente importante con externos, como técnicos de mantenimiento.

**5.1.5. Protección de racks y armarios.** Los servidores, switches y routers pueden instalarse en racks cerrados, que impiden manipulaciones no autorizadas, protegen el cableado, organizan los equipos y facilitan la gestión del espacio. Aun así, cerrar el rack no sustituye al control de acceso a la propia sala.

### 5.2. Protección frente al robo

El robo puede afectar a la vez a la disponibilidad y a la confidencialidad. Si roban un portátil con información empresarial, el equipo deja de estar disponible, el atacante puede intentar acceder a los datos, pueden quedar comprometidas credenciales almacenadas y la empresa sufre una pérdida económica y de información. Por eso la protección debe combinar medidas físicas y lógicas.

**5.2.1. Anclajes y sistemas antirrobo.** Cables de seguridad, anclajes, sistemas de fijación y armarios cerrados, útiles sobre todo en portátiles, equipos de sobremesa y dispositivos en zonas accesibles.

**5.2.2. Inventario de equipos.** La organización debe saber qué equipos posee y dónde están. Un inventario incluye datos como el tipo de equipo, fabricante, modelo, número de serie, dirección MAC, ubicación, responsable y estado, y permite detectar rápidamente la desaparición de un dispositivo.

| Dato | Ejemplo |
|---|---|
| Equipo | Servidor web |
| Fabricante | Dell |
| Modelo | PowerEdge |
| Número de serie | XXXXX |
| Dirección MAC | XX:XX:XX:XX:XX:XX |
| Ubicación | CPD |
| Responsable | Departamento IT |
| Estado | Operativo |

**5.2.3. Cifrado de discos.** El cifrado no evita el robo del equipo; su función es impedir o dificultar que quien lo robe acceda a la información. La protección física intenta evitar el robo y el cifrado protege la información aunque el robo se produzca. La distinción es especialmente importante en portátiles y dispositivos que salen de las instalaciones.

**5.2.4. Protección de las copias de seguridad.** También hay que protegerlas físicamente. Si el servidor, el NAS y las copias están en la misma sala y hay un incendio, se pierde todo a la vez. Por eso conviene tener copias en ubicaciones distintas y, según las necesidades, combinar backup local, externo y remoto.

**5.2.5. Destrucción segura de soportes.** Discos, USB, cintas y otros soportes no deben desecharse sin más. Antes de retirarlos se recurre al borrado seguro, la sobrescritura, el cifrado previo o la destrucción física cuando sea necesario, para que la información no pueda recuperarse después.

### 5.3. Protección contra incendios

El incendio es una de las amenazas físicas con más capacidad de destrucción en un CPD: puede destruir servidores, discos y almacenamiento, dañar el cableado, interrumpir servicios, provocar pérdida de información e inutilizar la instalación. La protección debe combinar prevención, detección, extinción y recuperación.

**5.3.1. Detección.** Detectores de humo y de temperatura, sistemas de alarma y sensores conectados a la monitorización. Una detección temprana permite actuar antes de que el daño sea mayor.

**5.3.2. Sistemas de extinción.** Deben adecuarse al tipo de infraestructura. En determinados CPD se usan sistemas automáticos con agentes gaseosos, que controlan el fuego sin volcar grandes cantidades de agua sobre equipos electrónicos. La elección debe seguir la normativa y las características de la instalación.

**5.3.3. Extintores.** Los equipos de extinción manual deben ser adecuados a los riesgos y estar bien mantenidos. En instalaciones eléctricas no vale cualquier tipo de extintor.

**5.3.4. Materiales y distribución de la instalación.** Para reducir la propagación del fuego se usan materiales adecuados, separación de zonas, compartimentación, buena gestión del cableado, mantenimiento eléctrico y eliminación de materiales innecesarios.

**5.3.5. Planes de evacuación y emergencia.** La seguridad física no protege solo equipos: **la prioridad en un incendio es proteger a las personas**. Hacen falta planes de evacuación, señalización, procedimientos de emergencia, puntos de reunión, formación del personal y revisiones periódicas.

### 5.4. Alimentación eléctrica

Los sistemas necesitan alimentación estable. Sus alteraciones van desde una interrupción temporal hasta daños permanentes.

**5.4.1. Principales amenazas eléctricas.** Los **cortes de suministro** son la pérdida completa de alimentación; las **sobretensiones**, una tensión superior a la adecuada; las **bajadas de tensión**, una inferior a la necesaria; los **picos**, incrementos muy rápidos de tensión; los **microcortes**, interrupciones de muy corta duración; y las **variaciones de frecuencia**, alteraciones de la frecuencia de la corriente. Las consecuencias pueden ser apagados inesperados, pérdida de datos, corrupción del sistema de archivos, interrupción de servicios, daños en componentes, fallos en discos y caídas de servidores.

**5.4.2. SAI/UPS.** Un SAI (Sistema de Alimentación Ininterrumpida, *UPS* en inglés) proporciona alimentación temporal ante un problema eléctrico. Su función no es solo mantener el equipo encendido: detecta la interrupción o alteración, mantiene temporalmente la alimentación, evita el apagado inmediato y da tiempo para que vuelva el suministro o para hacer un apagado controlado. La cadena normal es *red eléctrica → SAI → servidor*, y ante un corte el servidor sigue funcionando gracias al SAI. El tiempo que aguanta depende de su capacidad y de la carga conectada.

**5.4.3. Protección contra sobretensiones.** Los protectores ayudan a proteger los equipos frente a ciertos aumentos de tensión, y deben integrarse en una instalación eléctrica bien diseñada y mantenida.

**5.4.4. Generadores de emergencia.** En infraestructuras críticas se usa un grupo electrógeno para cortes prolongados. Ante un corte, el SAI mantiene la continuidad durante el intervalo necesario para que el generador arranque: *red eléctrica ❌ → SAI → grupo electrógeno → SAI → servidores*.

**5.4.5. Fuentes de alimentación redundantes.** Los servidores críticos pueden tener varias fuentes, por ejemplo la fuente A conectada al circuito eléctrico A y la fuente B al circuito B. Si una falla, la otra mantiene el servidor. Es un ejemplo de redundancia y reduce puntos únicos de fallo (SPOF).

### 5.5. Condiciones ambientales

Los equipos necesitan unas condiciones adecuadas, y las variables a controlar son temperatura, humedad, polvo, agua, humo y ventilación. Una condición inadecuada puede causar desde una reducción del rendimiento hasta daños permanentes.

**Temperatura.** El exceso provoca sobrecalentamiento, menor vida útil de los componentes, apagados automáticos, fallos de hardware y pérdida de rendimiento, por lo que las salas de servidores necesitan climatización adecuada.

**Humedad.** El exceso favorece condensación y corrosión; un nivel demasiado bajo favorece la electricidad estática. Debe mantenerse dentro de los valores adecuados para el entorno y los equipos.

**Polvo y partículas.** Su acumulación obstruye ventiladores, disipadores, filtros y sistemas de refrigeración, dificulta la disipación del calor y sube la temperatura de funcionamiento.

**Sensores y monitorización ambiental.** Las instalaciones críticas incorporan sensores de temperatura, humedad, humo, presencia de agua y estado de la climatización, que generan alertas cuando los valores salen de los límites.

### 5.6. Protección frente al agua e inundaciones

El agua es una amenaza importante. Puede venir de inundaciones externas, fugas de tuberías, sistemas de climatización, incendios y sistemas de extinción o problemas en plantas superiores, y puede dañar servidores, almacenamiento, cableado, sistemas eléctricos, SAI y equipos de comunicaciones. Las medidas son sensores de detección de agua, elevar determinados equipos, ubicar adecuadamente servidores y sistemas eléctricos, proteger el cableado, disponer de drenaje, mantener las instalaciones hidráulicas y elegir bien la ubicación del CPD. La ubicación física de un CPD es, por tanto, una decisión de seguridad.

### 5.7. Seguridad de las instalaciones y del CPD

Un CPD concentra muchos recursos críticos y necesita varias capas de protección; no basta con poner servidores en una habitación cerrada.

**Seguridad perimetral.** Controla el acceso a la zona donde está el CPD mediante puertas de seguridad, tarjetas, biometría, cámaras y personal de seguridad.

**Seguridad de la sala.** Control específico del acceso a la sala de servidores, con distintos niveles de autorización: *edificio → zona restringida → CPD → rack → equipo*. Cuanto más crítico es el recurso, mayor debe ser el control.

**Redundancia de instalaciones.** Los sistemas críticos pueden duplicar alimentación, SAI, generadores, climatización, conectividad, servidores y almacenamiento, para que el fallo de un solo componente no tumbe el servicio completo.

**Monitorización.** La infraestructura se supervisa para detectar rápidamente fallos de alimentación, temperaturas elevadas, humedad, humo, accesos no autorizados y fallos de hardware. Conecta con la detección de amenazas del apartado 4.

### 5.8. Seguridad física y continuidad del servicio

Las medidas físicas no deben analizarse de forma aislada: su fin último es reducir el impacto de los incidentes y mantener la continuidad de los servicios, siguiendo el ciclo *prevención → detección → respuesta → recuperación*. Ante un incendio, por ejemplo: detector de humo, alarma, extinción, parada controlada y recuperación desde copias de seguridad.

Aquí aparecen conceptos que se desarrollarán más adelante en ASIR: alta disponibilidad, redundancia, copias de seguridad, recuperación ante desastres, continuidad de negocio y planes de contingencia. Es importante recordar que **RAID, redundancia y copias de seguridad no son equivalentes**: RAID puede permitir que un servidor siga funcionando tras el fallo de un disco, pero no evita que un incendio destruya todo el servidor. Por eso las medidas deben combinarse.

### 5.9. Defensa en profundidad

La seguridad física debe formar parte de una estrategia de defensa en profundidad, con varias capas de modo que el fallo de una medida no comprometa por completo el sistema. Un ejemplo de infraestructura:

```
Control de acceso físico
        ↓
Sala de servidores protegida
        ↓
Rack cerrado
        ↓
Servidores con discos redundantes
        ↓
SAI y alimentación redundante
        ↓
Monitorización ambiental
        ↓
Firewall y controles de acceso lógico
        ↓
Copias de seguridad
        ↓
Plan de recuperación ante desastres
```

Cada medida cubre una parte distinta del riesgo.

### 5.10. Resumen del apartado

| Amenaza | Posible consecuencia | Medidas |
|---|---|---|
| Robo | Pérdida de equipos y datos | Control de acceso, anclajes, cifrado |
| Acceso físico no autorizado | Manipulación o robo | Tarjetas, biometría, cámaras |
| Incendio | Destrucción de equipos | Detectores, extinción, planes de emergencia |
| Corte eléctrico | Caída de servicios | SAI, generador, redundancia |
| Sobretensión | Daños en hardware | Protección eléctrica |
| Temperatura elevada | Sobrecalentamiento | Climatización y monitorización |
| Humedad | Corrosión/condensación | Control ambiental |
| Agua | Daños en equipos | Sensores y ubicación adecuada |
| Fallo de hardware | Interrupción del servicio | Redundancia, RAID, repuestos |
| Robo de soportes | Exposición de información | Cifrado y destrucción segura |

---

## 6. Hilo conductor del tema

Todo el tema se puede leer como una secuencia de preguntas que se van encadenando. Primero, qué propiedades queremos proteger: confidencialidad, integridad, disponibilidad y fiabilidad. Después, qué elementos pueden ser vulnerables: hardware, software y datos. Luego, qué debilidades pueden aprovecharse: las vulnerabilidades, que se identifican con CVE, se clasifican por tipo con CWE, se asocian a productos con CPE y se valoran con CVSS. A continuación, qué situaciones pueden provocar daños: las amenazas físicas y lógicas. Y por último, cómo protegemos físicamente los sistemas: la seguridad física y ambiental.

El siguiente paso, que abrirá el tema siguiente, es determinar qué probabilidad hay de que una amenaza aproveche una vulnerabilidad y qué consecuencias tendría. Eso nos lleva al concepto fundamental de **riesgo**.

---

## 7. Matices y correcciones sobre el material original

Esta sección no forma parte del temario. Son puntos donde el material tiene imprecisiones o donde conviene afinar, por si te preguntan en un examen o si quieres citarlo con rigor.

**Práctica de la versión del servidor (apartado 4.4.4).** El temario dice que si el servidor no tiene la opción `ServerSignature Off` ya tenemos la versión, y que esa cabecera muestra «la versión del navegador web». Hay dos errores. El primero es terminológico: es la versión del *servidor* web, no del navegador. El segundo es técnico: en Apache, `ServerSignature` controla el pie de página que aparece en las páginas de error, pero lo que oculta la versión en la cabecera `Server` es `ServerTokens Prod`. Para ocultarla en la cabecera hay que ajustar `ServerTokens`; lo ideal es configurar ambas directivas. En Nginx se usa `server_tokens off`. Además, ocultar la versión es una medida de *ofuscación* que reduce información útil al atacante, pero no sustituye a parchear.

**Definición de integridad (apartado 1).** El temario la limita a alteraciones «por causas involuntarias», pero después habla de modificación «no autorizada o accidental». La integridad protege frente a ambas, y es lo que conviene responder en un examen.

**Definición de disponibilidad.** También se limita a «causas involuntarias». Un DDoS es una causa deliberada y afecta directamente a la disponibilidad, tal como el propio temario explica en el apartado 4.4.6.

**Riesgo al final de la cadena (apartado 4.7).** El esquema coloca el riesgo como último eslabón, pero el riesgo no es una etapa posterior al impacto: es la *probabilidad* de que la amenaza explote la vulnerabilidad multiplicada por el *impacto* que tendría. Se evalúa antes de que ocurra el incidente. Esto es justo lo que se desarrollará en el tema siguiente.

**Generador y SAI (apartado 5.4.4).** El esquema «SAI → grupo electrógeno → SAI» es una simplificación. En la práctica, el generador se conecta a través de un conmutador de transferencia automático y alimenta la entrada del SAI o de la instalación; no hay dos SAI en serie. Lo esencial de la idea es correcto: el SAI cubre el hueco hasta que el generador arranca.

**Año en el identificador CVE.** El año de `CVE-AÑO-NÚMERO` corresponde a cuando se reservó o asignó el identificador, no necesariamente a cuando se publicó o descubrió la vulnerabilidad.

**Duplicidades.** El apartado 3.1 repite el concepto de vulnerabilidad del 2.1, y la definición del 2.10 vuelve a corregirla. En estos apuntes están fusionados para que no haya que leerlos tres veces.
