# Analizador de incidentes SOC

Aplicación web de una sola página para analizar logs exportados de un SIEM (Excel), detectar indicadores de compromiso, correlacionarlos en incidentes, clasificar el riesgo y generar un informe técnico en PDF. Proyecto del reto práctico de la Sesión #7, Bootcamp Analista SOC Nivel 2.

Todo corre en el navegador: el archivo no se envía a ningún servidor.

## Uso

1. Abre `index.html` en el navegador (requiere internet para cargar SheetJS, Chart.js y jsPDF desde cdnjs).
2. Pulsa **Cargar archivo .xlsx** y selecciona el export del SIEM.
3. Revisa el panel, los incidentes con su evidencia y el informe.
4. **Exportar PDF** descarga el informe. **Redactar con IA** solo funciona dentro de claude.ai (ver más abajo).

## Columnas esperadas

`Log_ID, Timestamp, Hostname, Event_ID, User, Source_IP` (obligatorias) y `Destination, Event_Description, Process_CommandLine, Result, Severity` (usadas por las reglas).

## Reglas de detección

| Categoría | Criterio | Puntaje base |
|---|---|---|
| Fuerza bruta / spraying | 3 o más eventos 4625 desde una misma IP externa (3 o más cuentas = spraying); sube si hay 4624 posterior desde esa IP | 40 (70 con acceso exitoso) |
| Escalamiento | Evento 4672 con IP de origen externa | 55 |
| Comandos | PowerShell codificado, `vssadmin delete shadows`, compresión (7z, tar, rar), reconocimiento (`whoami`, `net view`, etc.) en 4688/4104 | 40 a 85 |
| Persistencia | Eventos 4698 (tarea programada) y 7045 (servicio), más ejecuciones del binario instalado | 75 |
| Movimiento lateral | Inicios de sesión con `source=<host>` en la descripción, más acceso posterior a recursos compartidos | 65 a 70 |
| Exfiltración | 5 o más POST al proxy (o 50 MB o más) a un mismo destino; conexión saliente a IP externa con proceso | 70 a 85 |
| Ataque web | 10 o más peticiones HTTP desde IP externa; sube si el pool de IIS ejecuta un proceso | 50 (75 con ejecución) |

**Correlación:** los hallazgos se unen en un incidente cuando comparten IP externa o usuario.

**Puntaje del incidente:** puntaje del hallazgo más alto, más 5 por cada categoría adicional, con tope de 100. Niveles: Crítico 85 o más, Alto 65 o más, Medio 40 o más, Bajo el resto.

Cada hallazgo conserva los `Log_ID` que lo sustentan. La columna `Severity` del SIEM no se usa para clasificar.

## IA integrada

El botón **Redactar con IA** usa la capacidad `sample` de los artifacts de claude.ai y recibe únicamente los hallazgos y sus evidencias. Fuera de claude.ai no está disponible y el informe usa el texto generado por las reglas. Para usarla en otro entorno, reemplaza esa llamada por tu propio acceso a una API de modelos.

## Límites

- Los umbrales y puntajes son criterios propios y se pueden ajustar en la función `analyze()`.
- Las conclusiones se limitan a lo que muestran los registros; el tráfico ambiguo (por ejemplo, accesos de cuentas de servicio a recursos compartidos) no genera hallazgo.
