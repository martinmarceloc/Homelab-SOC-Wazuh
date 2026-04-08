# 🛡️ Homelab SOC: Despliegue de un SIEM en Red Segmentada

¡Hola! Decidí armar este laboratorio para llevar a la práctica los conceptos teóricos de mi carrera en telecomunicaciones y redes. El objetivo principal de este proyecto es desplegar un Centro de Operaciones de Seguridad (SOC) local utilizando **Wazuh** para monitorear, detectar y analizar vulnerabilidades en un entorno de servidores Linux.

No quería simplemente instalar un programa, sino diseñar la arquitectura de red desde cero, aislar el tráfico y automatizar la administración remota.

## 🛠️ Stack Tecnológico
* **Virtualización:** VirtualBox (Gestión vía CLI)
* **Sistemas Operativos:** Ubuntu Server 24.04 LTS
* **Redes y Enrutamiento:** Netplan, OpenSSH, Port Forwarding
* **Ciberseguridad (SIEM):** Wazuh (Indexer, Manager, Dashboard) y Elastic Stack.

---

## 🗺️ Arquitectura y Topología de Red

El laboratorio consta de dos máquinas virtuales configuradas con múltiples interfaces de red para simular un entorno corporativo segmentado:

1. **Wazuh-Server (El Cerebro):**
   * Adaptador 1 (NAT): Salida a internet.
   * Adaptador 2 (Internal Network `SOC-LAN`): IP estática `192.168.10.10`
   * Función: Recibir, indexar y analizar los logs de seguridad.

2. **Ubuntu-Agent (La Víctima):**
   * Adaptador 1 (NAT): Salida a internet.
   * Adaptador 2 (Internal Network `SOC-LAN`): IP estática `192.168.10.20`
   * Función: Máquina de un usuario o servidor interno que envía telemetría.

Para administrar todo de forma centralizada sin usar la interfaz gráfica (GUI) de las máquinas, configuré túneles SSH mediante **Port Forwarding** en el hipervisor (Puertos `2222` y `2223` mapeados al puerto `22` local). Todo el proyecto se opera remotamente desde Windows usando MobaXterm.

---

## 🚀 Fases del Proyecto (Hasta ahora)

### Fase 1: Infraestructura como Código (Netplan)
Para asegurar que la red privada sea persistente, configuré el enrutamiento escribiendo directamente los archivos YAML de Netplan. Mantuve las interfaces NAT con DHCP para las actualizaciones, mientras asigne el direccionamiento estático de la subred `192.168.10.0/24` a las interfaces de la LAN.

### Fase 2: Troubleshooting de Recursos (El OOM Killer)
Durante la instalación del indexador de Wazuh, me topé con un problema clásico de servidores: el *Out of Memory (OOM) Killer* de Linux estaba matando el proceso del *Manager* porque la base de datos consumía toda la RAM. 
* **La solución:** Apagué la máquina, reasigné la memoria física a 6GB y, para blindar el servidor, le particioné 4GB de memoria virtual (**Swap**) mediante línea de comandos. La instalación fluyó sin problemas después de esto.

### Fase 3: Despliegue del Cerebro (All-in-One)
Ejecuté la instalación del ecosistema Wazuh en el servidor principal. Esto levantó el Indexer (para búsquedas rápidas de logs), el Manager (el motor de reglas MITRE ATT&CK) y el Dashboard web, el cual expuse mediante HTTPS.

### Fase 4: Despliegue y Enlace del Agente
Generé el payload de instalación desde el servidor y lo ejecuté en la máquina víctima. Validé la conexión capa 3 con pings continuos y confirmé el *handshake* criptográfico. El agente ahora lee los logs locales de Ubuntu y los envía encriptados por la red `SOC-LAN`.

### Fase 5: Simulación de Ataques y Detección (Red Teaming Básico)
Para probar que el SIEM no está de adorno, simulé tráfico malicioso en la máquina víctima:
* Ejecuté múltiples intentos fallidos de inicio de sesión por SSH (Fuerza bruta).
* En cuestión de milisegundos, el dashboard web de Wazuh detectó la anomalía, la clasificó bajo las tácticas de acceso inicial y generó las alertas correspondientes en el panel visual.

---

## Nota
* Tener en cuenta la version del Ubuntu ya que algunos comandos cambian, para este caso utilice la version 24.04 TLS y por ejemplo el comando dhcp en esta nueva version se usa dhcpcd, y cosas asi hay que tener en cuenta siempre.

---

### Evidencias del Despliegue

**1. Esta imagen muestra la IP origen del ataque, la alerta y el nivel.
<img width="1895" height="972" alt="alertas" src="https://github.com/user-attachments/assets/ef8afa91-1f5e-456a-85e9-2fe48a41aa09" />

**2. Esta imagen muestra la configuracion de IP para el servidor utilizando el comando ip a, y tambien muestra la configuracion del archivo "01-netcfg.yaml" que utilice.

<img width="801" height="512" alt="servidor" src="https://github.com/user-attachments/assets/211f1b95-88ac-45bb-ab74-044db9ad0ba9" />

**3. Esta imagen muestra la memoria dada al servidor para solucionar el OOM Killer.

<img width="670" height="76" alt="memoria" src="https://github.com/user-attachments/assets/257fd080-f8d6-4d5f-aff5-2d60ba03732a" />
