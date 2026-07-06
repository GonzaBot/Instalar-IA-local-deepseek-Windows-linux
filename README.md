# Ollama + DeepSeek-R1: IA de razonamiento local en hardware modesto

Guía práctica para instalar y correr un modelo de inteligencia artificial de razonamiento (DeepSeek-R1) de forma 100% local y offline, sin depender de una GPU de alta gama ni de servicios pagos en la nube. Pensada especialmente para uso en ciberseguridad — análisis de logs, explicación de vulnerabilidades, apoyo en la redacción de informes técnicos — pero el proceso sirve para cualquier caso de uso de IA local.

## Qué vas a encontrar acá

- Instalación de **Ollama** (el motor de inferencia) paso a paso.
- Descarga y prueba de **DeepSeek-R1-Distill-Qwen-7B**, elegido por su balance entre calidad de razonamiento y tamaño (~4,7 GB, cuantización Q4_K_M).
- Uso por terminal y, opcionalmente, una interfaz web tipo ChatGPT con **Open WebUI** vía Docker.
- Cómo prender y apagar cada pieza según la necesites, para no dejar recursos consumiéndose de fondo cuando no estás usando el modelo.
- Problemas comunes y cómo resolverlos.

**No hace falta GPU dedicada.** Todo el proceso está probado en equipos de gama baja: si tenés una placa de video, Ollama la aprovecha; si no, corre igual sobre el procesador (más lento, pero funcional).

## Elegí tu sistema operativo

- **[README_linux.md](README_linux.md)** — instalación completa en Linux (probado en Linux Mint / Ubuntu).
- **[README_windows.md](README_windows.md)** — instalación completa en Windows 10/11.

El comando para descargar y correr el modelo (`ollama pull deepseek-r1:7b` / `ollama run deepseek-r1:7b`) es idéntico en los dos sistemas operativos. Lo que cambia entre una guía y otra es cómo se instala Ollama, cómo se gestiona el servicio en segundo plano, y el setup de Docker para la interfaz web opcional — cada guía lo cubre de punta a punta para su plataforma.

---

Autor: [GonzaBot](https://github.com/GonzaBot)
