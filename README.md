# 🛡️ Mi Laboratorio SOC: Wazuh + Alertas en Discord 🚀

¡Hola! Decidí armar este laboratorio para meter las manos en la masa y llevar a la práctica lo que voy aprendiendo sobre redes y seguridad. La idea no era simplemente darle "siguiente, siguiente" a un instalador, sino armar una red desde cero, separar el tráfico y ver cómo funciona un SIEM (Wazuh) en la vida real usando servidores Linux.

El objetivo final: Monitorear mis máquinas y que me llegue una notificación al celular por Discord si alguien intenta entrar a la mala.

## 🛠️ Lo que usé para este proyecto
* **Máquinas Virtuales:** VirtualBox (manejado todo por consola, sin interfaz gráfica).
* **Sistemas Operativos:** Ubuntu Server 24.04 LTS.
* **Redes:** Netplan, OpenSSH y Port Forwarding.
* **Seguridad:** Wazuh (el cerebro que vigila todo) + un Webhook de Discord.

---

## 🗺️ Cómo armé la Red

Creé dos máquinas virtuales y a cada una le puse dos tarjetas de red: una para tener salida a internet (NAT) y otra para crear una red interna (`SOC-LAN`) donde se comunican entre ellas de forma aislada.

1. **Wazuh-Server (El Cerebro):** Tiene la IP estática `192.168.10.10`. Acá se guardan y analizan los logs.
2. **Ubuntu-Agent (La Víctima):** Tiene la IP estática `192.168.10.20`. Es la máquina que simula a un usuario normal y envía sus reportes.

**Dato:** Para no usar la pantallita chica de VirtualBox, mapeé los puertos en el hipervisor (Port Forwarding a los puertos 2222 y 2223) y ahora controlo ambas máquinas tranquilamente por SSH desde mi Windows usando MobaXterm.

---

## 🚀 Pasos que seguí y problemas que me topé

### Fase 1: Peleando con la red (Netplan)
Para que las IPs no cambien cada vez que reinicio, tuve que meterme a editar los archivos YAML de Netplan a mano. Dejé las interfaces con internet en DHCP y le clavé las IPs estáticas a las de la red interna (usando la subred `192.168.10.0/24`).

### Fase 2: El servidor se quedaba sin RAM (OOM Killer)
Instalando Wazuh, el servidor se me moría. Resulta que el sistema de Linux estaba matando el proceso (el famoso OOM Killer) porque la base de datos se comía toda la RAM. Lo solucioné apagando la máquina, subiéndole la RAM a 6GB y, por si acaso, le armé 4GB de memoria Swap a pura consola. Después de eso, instaló como seda.

### Fase 3 y 4: Instalando Wazuh y conectando a la víctima
Instalé todo el ecosistema de Wazuh en el servidor principal. Luego generé un instalador para el Agente, lo corrí en la máquina víctima, comprobé con unos pings que se vieran bien, y listo. El agente empezó a mandar la telemetría encriptada por la red interna.

<img width="495" height="177" alt="maquina1 a maquina2" src="https://github.com/user-attachments/assets/36fd6771-bc53-4996-a73f-aed5d34346c5" />
<img width="501" height="158" alt="maquina 2 a maquina 1" src="https://github.com/user-attachments/assets/bfc380fe-ef09-42b3-9070-c3ebdf2a1cfc" />

### Fase 5: Simulando un ataque (Fuerza bruta)
Para comprobar que el SIEM no estaba ahí de adorno, me puse a fallar inicios de sesión por SSH a propósito. En cuestión de milisegundos, el panel web de Wazuh detectó el comportamiento raro y me generó la alerta en rojo.

### Fase 6: Avisos por Discord (Mi mayor dolor de cabeza)
Quería que Wazuh me mande un mensaje a Discord si pasaba algo grave. Intenté usar la configuración que trae por defecto para Slack, pero no funcionaba y me tiraba un "Error 7" en los logs. Descubrí que Wazuh manda los datos crudos en texto plano (como `ruledescription='...'`) y la API de Discord lo rechazaba.
* **La solución:** Me armé un script propio en Bash (`custom-discord`). Usé `grep` y `cut` para extraer solo el texto del ataque y mandarlo a Discord usando `curl`. Le acomodé los permisos de Linux al grupo `wazuh` y ¡funcionó!

<img width="753" height="327" alt="custom-discord" src="https://github.com/user-attachments/assets/673ed0a8-0e6d-49dd-a3fe-673d01cbb783" />

Para que Wazuh supiera que tenía que usar este script, tuve que modificar su archivo de configuración principal (`ossec.conf`). Le puse que solo use el webhook con alertas de nivel 10 para no llenarme de spam por cosas sin importancia. Así quedó el código:

<img width="762" height="926" alt="ossec conf foto" src="https://github.com/user-attachments/assets/4522773d-eb8a-4338-b91c-6ac213557ce7" />


---

## 📸 Evidencias del Despliegue

**1. Ataque detectado:** Esta imagen muestra la IP origen del ataque, la alerta en el dashboard y el nivel de severidad.
<img width="1895" height="972" alt="alertas" src="https://github.com/user-attachments/assets/ef8afa91-1f5e-456a-85e9-2fe48a41aa09" />

**2. Red configurada:** Esta imagen muestra la configuración de IP para el servidor utilizando el comando `ip a`, y también el archivo `01-netcfg.yaml` que usé con Netplan.
<img width="801" height="512" alt="servidor" src="https://github.com/user-attachments/assets/211f1b95-88ac-45bb-ab74-044db9ad0ba9" />

**3. Salvando la RAM:** Esta imagen muestra la memoria Swap creada y los recursos asignados para evitar el OOM Killer.
<img width="670" height="76" alt="memoria" src="https://github.com/user-attachments/assets/257fd080-f8d6-4d5f-aff5-2d60ba03732a" />

**4. El resultado final (Funcionando al 100%):** Aquí se ve lado a lado cómo el ataque salta en el dashboard de Wazuh y, al mismo instante, me llega la alerta al canal de Discord.
<img width="1919" height="1075" alt="dashboardydiscord" src="https://github.com/user-attachments/assets/f988c531-df0c-4c8c-8309-dbbcbdbded71" />


---

## 📝 Apuntes para no olvidarme
* **Ojo con la versión de Ubuntu:** Estoy usando la 24.04 LTS y varias cosas cambian respecto a tutoriales más viejos. Por ejemplo, cómo maneja los servicios de red (`dhcpcd` vs antiguos) y la sintaxis de Netplan.
* **Sintaxis estricta:** Siempre hay que tener cuidado con la indentación de los archivos YAML o los scripts en Bash, un espacio mal puesto y el servicio no levanta.
