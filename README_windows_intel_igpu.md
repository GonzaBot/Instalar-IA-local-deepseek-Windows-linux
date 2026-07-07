# Ollama + DeepSeek-R1 en Windows con GPU Intel Iris Xe (aceleración real)

Guía para correr DeepSeek-R1 en Windows aprovechando la GPU integrada Intel Iris Xe en vez de dejar que todo el peso caiga en la CPU. El Ollama estándar (el que baja `winget` o el instalador oficial) **no acelera bien las iGPU Intel** — detecta la tarjeta pero la descarta o la usa al mínimo. Esta guía usa el backend SYCL de Intel (vía [IPEX-LLM](https://github.com/intel/ipex-llm)) para que la inferencia corra de verdad sobre la Iris Xe.

Probado en Windows 11 con Intel Iris Xe Graphics (memoria compartida ~10GB) y 20GB de RAM.

## Por qué esto es un proceso aparte

Con el Ollama normal vas a ver algo así en el Administrador de Tareas: 2% de uso de GPU, 80%+ de RAM, CPU al tope. Eso significa que el modelo está corriendo 100% por CPU aunque la GPU esté ahí sin usarse. Pasa porque:

1. Ollama estándar no trae el backend SYCL que necesitan las iGPU Intel.
2. Incluso el soporte Vulkan experimental de Ollama descarta las iGPU por defecto (hace falta forzarlo, y aun así puede fallar).

La solución estable es reemplazar el ejecutable de Ollama por uno compilado con soporte SYCL vía IPEX-LLM, el proyecto de Intel para esto.

## Prerrequisitos

- Windows 10/11 64-bit
- Intel Iris Xe Graphics (11va a 14va gen de Intel Core)
- Driver de gráficos Intel **31.0.101.x o más nuevo** (clave — con versiones más viejas falla la asignación de memoria SYCL)
- Python 3.11 (IPEX-LLM no soporta versiones más nuevas todavía)
- ~10GB libres en disco (dependencias + modelo)

## Paso 1: Actualizar el driver de Intel

Bajalo desde la página oficial (no confíes en Windows Update, suele quedarse atrás):

https://www.intel.com/content/www/us/en/download/864990/intel-11th-14th-gen-processor-graphics-windows.html

Instalá y reiniciá. Para verificar la versión: `dxdiag` → pestaña "Pantalla".

## Paso 2: Instalar Python 3.11 (en paralelo, sin tocar tu Python actual)

Por consola (PowerShell):

```powershell
Invoke-WebRequest -Uri "https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe" -OutFile "$env:USERPROFILE\Downloads\python-3.11.9-amd64.exe"

Start-Process -FilePath "$env:USERPROFILE\Downloads\python-3.11.9-amd64.exe" -ArgumentList "/quiet InstallAllUsers=0 PrependPath=1 Include_launcher=1" -Wait
```

Verificá con `py -0` que aparezcan tanto tu versión anterior como la 3.11.

## Paso 3: Crear el entorno virtual e instalar IPEX-LLM

```powershell
py -3.11 -m venv ipexenv
.\ipexenv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install --pre --upgrade ipex-llm[cpp]
```

## Paso 4: Inicializar el Ollama parcheado

**Importante:** este paso necesita permisos de administrador (crea symlinks). Abrí la terminal como administrador.

```powershell
init-ollama.bat
```

Esto crea un `ollama.exe` propio (con `ggml-sycl.dll` incluido) en la carpeta actual, junto a un puñado de DLLs de soporte.

## Paso 5: Variables de entorno necesarias

El ejecutable de IPEX-LLM necesita ciertas DLLs de Intel en el PATH y varias variables seteadas. Usá **cmd.exe**, no PowerShell (la sintaxis de abajo es de cmd; con PowerShell hay diferencias que complican innecesariamente esta parte).

```cmd
cd C:\Users\<tu-usuario>
ipexenv\Scripts\activate.bat
set PATH=C:\Users\<tu-usuario>\ipexenv\Library\bin;%PATH%
set SYCL_DEVICE_FILTER=level_zero:gpu
set ONEAPI_DEVICE_SELECTOR=level_zero:0
set ZES_ENABLE_SYSMAN=1
set OLLAMA_NUM_GPU=999
set OLLAMA_INTEL_GPU=true
set OLLAMA_NUM_PARALLEL=1
set UR_L0_ENABLE_RELAXED_ALLOCATION_LIMITS=1
```

## Paso 6: Levantar el servidor

```cmd
ollama serve
```

Antes de correr esto, asegurate de que no quede corriendo una instancia del Ollama normal (el de winget se reinicia solo como servicio):

```cmd
tasklist | findstr ollama
taskkill /F /IM ollama.exe
taskkill /F /IM "ollama app.exe"
```

En el log deberías ver algo como:

```
msg="inference compute" ... library=oneapi ... total="9.8 GiB"
```

Si en cambio dice `library=cpu`, algo quedó mal configurado — revisá el PATH y las variables del paso 5.

## Paso 7: Correr DeepSeek-R1

En **otra** ventana de cmd (dejando el servidor corriendo en la primera), activá el mismo entorno:

```cmd
cd C:\Users\<tu-usuario>
ipexenv\Scripts\activate.bat
set PATH=C:\Users\<tu-usuario>\ipexenv\Library\bin;%PATH%
ollama run deepseek-r1:7b
```

La primera vez baja el modelo (~4,7GB); las siguientes arranca directo.

## Cómo apagar todo

- Salir del chat: `/bye`
- Cortar el servidor: `Ctrl+C` en la ventana de `ollama serve`

## Problemas comunes

**"No encontro sycl8.dll / svml_dispmd.dll / libmmd.dll"**
Faltan las DLLs de Intel en el PATH. Agregá manualmente la carpeta que las contiene:
```cmd
set PATH=C:\Users\<tu-usuario>\ipexenv\Library\bin;%PATH%
```

**"unable to allocate SYCL0 buffer"**
Dos causas posibles, probar en este orden:
1. Driver desactualizado — confirmá que estás en 31.0.101.x o más nuevo.
2. Ollama por defecto abre 4 "copias" de paralelismo (`OLLAMA_NUM_PARALLEL=4`), lo que multiplica x4 la memoria que pide de golpe — algo que la iGPU no banca. Bajalo a 1: `set OLLAMA_NUM_PARALLEL=1`.

**El server detecta la GPU pero la "dropea" (`dropping integrated GPU`)**
Ollama descarta iGPUs por defecto en su backend Vulkan nativo. Si en algún punto probás sin IPEX-LLM, hace falta `set OLLAMA_IGPU_ENABLE=1`. Con el flujo de esta guía (IPEX-LLM/SYCL) no debería pasar.

**Ver y borrar modelos descargados (liberar espacio)**
```cmd
ollama list
ollama rm nombre-del-modelo
```

## Nota de expectativas

Una Iris Xe integrada no compite con una GPU dedicada — la memoria compartida es más lenta que la VRAM dedicada. Con esta guía dejás de correr todo por CPU, que ya es una mejora real, pero seguí usando cuantizaciones Q4 y modelos de hasta ~8B para que la experiencia sea fluida.

---

Autor: [GonzaBot](https://github.com/GonzaBot)
