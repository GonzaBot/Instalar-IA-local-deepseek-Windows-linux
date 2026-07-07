# Cómo instalar un LLM local (DeepSeek-R1) con Ollama en Linux

Guía paso a paso para correr un modelo de inteligencia artificial de razonamiento en forma 100% local y offline, sin depender de una GPU de alta gama ni de servicios en la nube. Pensada para uso en ciberseguridad: análisis de logs, explicación de vulnerabilidades, apoyo en redacción de informes técnicos.

> **Requisito real:** una PC de gama baja, sin GPU dedicada para IA, alcanza. Esta guía está probada en ese escenario, no en el caso ideal que suelen mostrar otros tutoriales.
>
> Versión para Windows: [README_windows.md](README_windows.md)

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

- Distribución Linux basada en Ubuntu/Debian (probado en Linux Mint).
- Al menos 8 GB de RAM (16 GB recomendado para mayor margen).
- Al menos 6 GB de espacio libre en disco.
- Conexión a internet solo para la descarga inicial (después funciona 100% offline).
- **No hace falta GPU dedicada.** Si tenés una, Ollama la va a aprovechar en la medida que pueda; si no, corre igual sobre el procesador.

## 2. Instalar Ollama

Ollama es el motor que gestiona la descarga, carga en memoria y ejecución del modelo, y expone una API local compatible con el estándar de la industria.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

El instalador detecta automáticamente el hardware disponible (incluida GPU NVIDIA si hay una), crea un usuario de sistema dedicado y registra un servicio systemd para que arranque solo en segundo plano.

Verificá que el servicio quedó activo:

```bash
systemctl status ollama
```

## 3. Descargar el modelo

Se recomienda **DeepSeek-R1-Distill-Qwen-7B** en cuantización Q4_K_M, por su buen balance entre calidad de razonamiento y tamaño en memoria (~4,7 GB):

```bash
ollama pull deepseek-r1:7b
```

Si el equipo es particularmente limitado o la respuesta se siente muy lenta, existe una versión más liviana:

```bash
ollama pull deepseek-r1:1.5b
```

Confirmá que quedó instalado:

```bash
ollama list
```

## 4. Probar el modelo por terminal

Modo de sesión interactiva (recomendado, consume menos recursos que la interfaz web):

```bash
ollama run qwen2.5:0.5b (este va a ser el de nuestro lab el otro va a fallar en las PC)



ollama run deepseek-r1:7b
```

Escribís tu consulta y Enter. Para salir: `/bye` (o Ctrl+D).

También podés tirar un prompt puntual sin entrar a sesión interactiva:

```bash
ollama run deepseek-r1:7b "Explicame qué es un ataque de privilege escalation"
```

**Expectativa realista en hardware de gama baja:** las respuestas pueden tardar entre varios segundos y un par de minutos según la longitud del prompt, ya que gran parte del trabajo recae en el procesador. Es normal y sigue siendo útil para tareas que no requieren tiempo real.

Vas a notar que el modelo razona primero dentro de una etiqueta `<think>...</think>` antes de dar la respuesta final, y ese razonamiento suele salir en inglés aunque el prompt esté en español. Es un comportamiento propio del modelo, no un error.

## 5. Interfaz web (opcional)

Si preferís una interfaz tipo chat en el navegador en vez de la terminal, podés levantar **Open WebUI** con Docker:

```bash
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Qué hace cada parte:
- `-p 3000:8080` → expone la interfaz en el navegador en el puerto 3000
- `--add-host=host.docker.internal:host-gateway` → permite que el contenedor le hable a Ollama, que corre fuera del contenedor
- `-v open-webui:...` → volumen persistente para no perder el historial de chats al reiniciar

Confirmá que está corriendo:

```bash
docker ps
```

Después abrí `http://localhost:3000` en el navegador de la **misma máquina** donde corre Docker. La primera vez te pide crear una cuenta local (queda todo en tu equipo, no es un servicio externo). En la selección de modelo va a aparecer automáticamente `deepseek-r1:7b`.

## 6. Comandos de gestión

Referencia rápida para prender y apagar cada pieza según la necesites — útil para no dejar recursos consumiéndose de fondo cuando no estás usando el modelo.

**Verificar qué está corriendo:**

```bash
ollama list              # modelos instalados
ollama ps                 # qué modelo está cargado en memoria ahora mismo (CPU/GPU/mixto)
systemctl status ollama   # estado del servicio
docker ps                 # contenedores corriendo (incluye Open WebUI si lo usás)
```

**Apagar, de más liviano a más pesado:**

```bash
docker stop open-webui           # apaga la interfaz web (no borra el historial)
ollama stop deepseek-r1:7b        # descarga el modelo de RAM/VRAM sin desinstalarlo
sudo systemctl stop ollama        # apaga el motor completo (libera hasta la base, ~300 MB)
```

Nota: si no descargás el modelo manualmente, Ollama lo hace solo después de 5 minutos de inactividad.

**Volver a prender:**

```bash
sudo systemctl start ollama      # motor de Ollama
docker start open-webui           # interfaz web (arranca más rápido que crearla de nuevo)
```

Correr `ollama run deepseek-r1:7b` ya vuelve a cargar el modelo en memoria automáticamente, no hace falta ningún comando extra para eso.

**Para el uso diario**, con `docker stop`/`docker start` y dejar que Ollama descargue el modelo solo por el timeout de inactividad ya alcanza — rara vez hace falta tocar el servicio completo.

**Forzar solo CPU** (si una GPU con poca VRAM da problemas de estabilidad):

```bash
sudo systemctl edit ollama.service
```

Agregar en el archivo que se abre:

```ini
[Service]
Environment="CUDA_VISIBLE_DEVICES=-1"
```

Aplicar:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## 7. Problemas comunes

**La pantalla parpadea o el sistema se pone inestable (GPU con poca VRAM).**
Forzá el uso exclusivo del procesador con los pasos de "Forzar solo CPU" de la sección anterior.

**Las respuestas tardan mucho.**
Es esperable en hardware de gama baja. Si el tiempo no es aceptable para tu caso de uso, probá `deepseek-r1:1.5b`, notablemente más rápido a costa de algo de profundidad de razonamiento.

**El navegador no carga `http://localhost:3000`.**
Primero confirmá el estado del contenedor:

```bash
docker ps -a
```

Si dice `Exited`, arrancalo con `docker start open-webui`. Si dice `Up` pero igual no conecta, revisá los logs:

```bash
docker logs open-webui --tail 50
```

Si en el arranque ves líneas de `alembic.runtime.migration`, es normal — son las migraciones de base de datos de Open WebUI en su primer arranque, no un error. Esperá un minuto y probá de nuevo.

---

Autor: [GonzaBot](https://github.com/GonzaBot)
