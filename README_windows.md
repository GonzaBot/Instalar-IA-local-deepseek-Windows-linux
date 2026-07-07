# Cómo instalar un LLM local (DeepSeek-R1) con Ollama en Windows

Guía paso a paso para correr el mismo modelo de razonamiento 100% local y offline que la versión Linux, sin depender de una GPU dedicada ni de servicios en la nube. Pensada para uso en ciberseguridad: análisis de logs, explicación de vulnerabilidades, apoyo en redacción de informes técnicos.

> Versión para Linux: [README_linux.md](README_linux.md)

## Índice

1. [Requisitos previos](#1-requisitos-previos)
2. [Instalar Ollama](#2-instalar-ollama)
3. [Descargar el modelo](#3-descargar-el-modelo)
4. [Probar el modelo por terminal](#4-probar-el-modelo-por-terminal)
5. [Interfaz web (opcional)](#5-interfaz-web-opcional)
6. [Comandos de gestión](#6-comandos-de-gestión)
7. [Problemas comunes](#7-problemas-comunes)

---

## 1. Requisitos previos

- Windows 10 versión 1903 o superior (64 bits), o Windows 11.
- Al menos 8 GB de RAM (16 GB recomendado para mayor margen).
- Al menos 6 GB de espacio libre en disco.
- Conexión a internet solo para la descarga inicial (después funciona 100% offline).
- **No hace falta GPU dedicada.** Si tenés una NVIDIA, Ollama la va a aprovechar; si no, corre igual sobre el procesador.
- Solo si vas a usar la interfaz web con Docker (sección 5): virtualización activada en la BIOS/UEFI (Intel VT-x o AMD-V) y una cuenta con permisos de administrador.

## 2. Instalar Ollama

A diferencia de Linux, acá hay dos caminos que llegan al mismo lugar.

**Opción A — instalador gráfico (la más simple):**

Bajá `OllamaSetup.exe` desde ollama.com/download/windows y hacé doble clic. No pide permisos de administrador: se instala dentro de tu cuenta de usuario, no en Archivos de programa. Al terminar, Ollama arranca solo en segundo plano y queda un ícono en la bandeja del sistema, al lado del reloj.

**Opción B — instalador por PowerShell:**

```powershell
irm https://ollama.com/install.ps1 | iex
```

Si la política de ejecución bloquea el script:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Importante: cerrá la terminal que tenías abierta antes de instalar y abrí una nueva. El PATH no se actualiza en la sesión vieja, así que el comando `ollama` no se va a reconocer ahí todavía.

Verificá la instalación:

```powershell
ollama --version
```

Si Windows Defender SmartScreen tira una advertencia al abrir el instalador (porque no está firmado digitalmente), click en "Más información" → "Ejecutar de todas formas". El instalador oficial de ollama.com es seguro.

## 3. Descargar el modelo

La CLI de Ollama es idéntica en todas las plataformas — el mismo comando que en Linux:

```powershell
ollama run qwen2.5:0.5b (este va a ser el de nuestro lab el otro va a fallar en las PC)


ollama pull deepseek-r1:7b
```

Baja los mismos ~4,7 GB en cuantización Q4_K_M y verifica el hash criptográfico al terminar.

Si el equipo es particularmente limitado o la respuesta se siente muy lenta:

```powershell
ollama pull deepseek-r1:1.5b
```

Confirmá que quedó instalado:

```powershell
ollama list
```

## 4. Probar el modelo por terminal

Modo de sesión interactiva (escribís el comando sin texto al final y chateás dentro de la sesión; para salir, `/bye`):

```powershell
ollama run deepseek-r1:7b
```

O un prompt puntual sin entrar a sesión interactiva:

```powershell
ollama run deepseek-r1:7b "Explicame qué es un ataque de privilege escalation"
```

**Expectativa realista en hardware de gama baja:** las respuestas pueden tardar entre varios segundos y un par de minutos según la longitud del prompt. Es normal y sigue siendo útil para tareas que no requieren tiempo real.

Igual que en Linux, la respuesta va a incluir un bloque `<think>` con el razonamiento del modelo antes de la respuesta final, generalmente en inglés aunque el prompt esté en español — es un comportamiento del modelo, no un error del sistema.

## 5. Interfaz web (opcional)

**Opción A — la app nativa de Ollama (más simple, no necesita Docker):**

Ollama para Windows trae una app de escritorio propia. Click en el ícono de la bandeja del sistema (o buscá "Ollama" en el menú Inicio), elegís el modelo del desplegable y chateás directo, sin terminal ni contenedores.

**Opción B — Open WebUI vía Docker (para tener la misma interfaz que en Linux):**

A diferencia de Linux, acá Docker no viene instalado de entrada y el setup pide permisos de administrador.

Habilitar WSL2 (una sola vez, PowerShell como administrador):

```powershell
wsl --install
```

Reiniciá Windows cuando lo pida. Instalá Docker Desktop desde docker.com/products/docker-desktop y confirmá que está activo:

```powershell
docker --version
```

Levantar el contenedor — mismo comando que en Linux, con una diferencia clave: en Windows `host.docker.internal` se resuelve solo, así que no hace falta el flag `--add-host`:

```powershell
docker run -d -p 3000:8080 `
  -v open-webui:/app/backend/data `
  --name open-webui `
  --restart always `
  ghcr.io/open-webui/open-webui:main
```

(La comilla invertida al final de línea es el continuador en PowerShell, reemplaza a la barra invertida `\` de bash.)

Confirmá que está corriendo:

```powershell
docker ps
```

Abrís `http://localhost:3000` en el navegador. La primera vez pide crear una cuenta local (queda todo en tu equipo) y detecta `deepseek-r1:7b` automáticamente.

## 6. Comandos de gestión

Igual que en Linux, referencia rápida para prender y apagar cada pieza — útil para no dejar recursos consumiéndose de fondo. La diferencia central es que **Windows no usa systemd**, así que no hay `systemctl`: Ollama corre como aplicación en la bandeja del sistema.

**Verificar qué está corriendo:**

```powershell
ollama list              # modelos instalados
ollama ps                 # qué modelo está cargado en memoria ahora (CPU/GPU/mixto)
docker ps                  # contenedores corriendo (incluye Open WebUI, si lo usás)
```

**Apagar, de más liviano a más pesado:**

```powershell
docker stop open-webui              # apaga la interfaz web (no borra el historial)
ollama stop deepseek-r1:7b           # descarga el modelo de RAM/VRAM sin desinstalarlo
```

Para apagar Ollama por completo: click derecho en el ícono de la bandeja del sistema → Quit (o `Stop-Process -Name ollama` desde PowerShell).

Nota: si no descargás el modelo manualmente, Ollama lo hace solo después de 5 minutos de inactividad.

**Volver a prender:**

```powershell
docker start open-webui     # interfaz web (arranca más rápido que crearla de nuevo)
```

Para Ollama, si lo cerraste desde la bandeja, volvé a abrirlo desde el menú Inicio. Correr `ollama run deepseek-r1:7b` ya carga el modelo en memoria automáticamente, no hace falta ningún paso extra.

**Para el uso diario**, con `docker stop`/`docker start` y dejar que Ollama descargue el modelo solo por el timeout de inactividad ya alcanza.

**Forzar solo CPU** (si una GPU con poca VRAM da problemas de estabilidad): creá la variable de entorno `CUDA_VISIBLE_DEVICES` con valor `-1` desde Configuración → Variables de entorno de tu cuenta, y reiniciá la app de Ollama para que tome el cambio.

## 7. Problemas comunes

**El comando `ollama` no se reconoce después de instalar.**
Cerrá todas las terminales abiertas y abrí una nueva — recién ahí se actualiza el PATH. Si sigue sin andar, reiniciá Windows.

**Las respuestas tardan mucho.**
Es esperable en hardware de gama baja. Si el tiempo no es aceptable para tu caso de uso, probá `deepseek-r1:1.5b`.

**El navegador no carga `localhost:3000`.**
Mismo diagnóstico que en Linux: `docker ps -a` para confirmar que el contenedor sigue "Up", y `docker logs open-webui --tail 50` para ver si todavía está corriendo las migraciones de base de datos (arranque en frío normal, tarda un par de minutos) o si hay un error real.

**Puerto 11434 ocupado.**

```powershell
netstat -ano | findstr 11434
```

Es el equivalente Windows de `lsof`/`ss` en Linux.

**Docker Desktop no arranca o dice que WSL2 no está soportado.**
Confirmá que la virtualización esté activa: Administrador de tareas → Rendimiento → CPU, tiene que decir "Virtualización: Habilitada". Si dice "Deshabilitada", activala en la BIOS/UEFI (Intel VT-x o AMD-V) antes de que Docker Desktop pueda arrancar.

---

Autor: [GonzaBot](https://github.com/GonzaBot)
