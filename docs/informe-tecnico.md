# Informe técnico - Caracterización del modelo

| Dato | Cómo obtenerlo | Valor |
|---|---|---|
| Perfil de hardware | Sección 2 de la guía | B (8 GB) |
| RAM total del equipo | `free -h` | 4.0 GB visibles en WSL (equipo de 16 GB) |
| Modelo y etiqueta | `ollama list` | qwen2.5:0.5b |
| Tamaño en disco | `ollama list` | 397 MB |
| Latencia de 5 ejecuciones (ms) | `time curl ...` cinco veces | 5099, 3871, 3136, 3297, 3339|
| Latencia promedio | Promedio de las cinco | 3748 ms |
| RAM usada durante la inferencia | `free -h` mientras responde | 1001 MB |
| Calidad percibida (1 a 5) | Su criterio, con una frase que lo justifique | 3: responde en español y acierta lo básico, pero inventa datos y no respeta instrucciones como responder en una sola frase |
## Plan de pruebas del despliegue

Las pruebas se hicieron con la solución levantada con Docker Compose. Las pruebas funcionales también corren automáticamente en el pipeline de CI.

| ID | Tipo | Qué se verifica | Resultado esperado | Obtenido | Estado |
|---|---|---|---|---|---|
| P-01 | Funcional | GET /health responde | Código 200 y estado ok | 200, {"estado":"ok","base_datos":"ok"} | Aprobado |
| P-02 | Funcional | POST /clasificar con motor eco | Código 200 y tipo correcto | 200, tipo "fix" para "corrige el error de conexion a la base de datos" | Aprobado |
| P-03 | Funcional | Motor inválido | Código 400 | 400 (prueba automática con pytest) | Aprobado |
| P-04 | Acceso | Rol app_ia intenta DROP TABLE | Error de permisos | DELETE: "permission denied for table inferencias". DROP TABLE: "must be owner of table inferencias". El SELECT sí funcionó | Aprobado |
| P-05 | Conectividad | La API resuelve el host db | Devuelve una IP interna | 172.18.0.2 | Aprobado |
| P-06 | Disponibilidad | Reinicio del contenedor de BD | La API se recupera sola | Después de docker compose restart db y 15 segundos, /health respondió ok | Aprobado |
| P-07 | Persistencia | down y up conservan los datos | Los registros siguen existiendo | Los registros con id 1 y 2 siguieron apareciendo en /inferencias | Aprobado |
| P-08 | Carga | 10 usuarios sobre el motor eco | p95 < 800 ms y errores < 5 % | p95 = 32.58 ms, errores 0 %, 648 peticiones (promedio 25.02 ms, mediana 23.1 ms, máximo 155.4 ms) | Aprobado |
| P-09 | Caracterización | 10 inferencias con modelo | Promedio, mediana y p95 | Promedio 4787 ms, mediana 4776 ms, p95 5199 ms (modelo qwen2.5:0.5b) | Aprobado |

### Análisis del cuello de botella

Al comparar P-08 con P-09 se ve que el tiempo se pierde en la inferencia del modelo. Con el motor eco el p95 fue de 32.58 ms, y esa cifra ya incluye la API, la conexión y el INSERT en PostgreSQL. Con el motor ollama el promedio fue de 4787 ms, unas 190 veces más lento. Entonces la API y la base de datos no son el problema: casi todo el tiempo se va en cargar el modelo y generar la respuesta.

Otra causa es la configuración OLLAMA_KEEP_ALIVE=0, que descarga el modelo de la memoria después de cada respuesta. Eso ahorra RAM, pero obliga a cargarlo de nuevo en cada petición. El consumo de la API fue bajo (api-ia usaba cerca de 39 MiB de memoria en reposo), así que el equipo no se quedó corto por el lado del servicio.

### Calidad del modelo

En las 10 inferencias, el modelo qwen2.5:0.5b clasificó bien solo 4 (40 %) según nuestro criterio, y respondió "feat" en 7 de los 10 casos. El motor eco, que usa reglas, acierta cuando la palabra clave es clara (por ejemplo "corrige" da fix). Con esta configuración el modelo pequeño no supera a las reglas en esta tarea. Esto coincide con lo que anticipa la guía: los modelos de menos de mil millones de parámetros tienen calidad limitada.

### Propuestas de mejora

1. **Mantener el modelo cargado en memoria.** Subir OLLAMA_KEEP_ALIVE (por ejemplo a 5m) en equipos con RAM suficiente. Así solo la primera petición paga el costo de carga y las demás bajan de latencia.
2. **Agregar una caché para mensajes repetidos.** Si llega el mismo texto otra vez, se devuelve el resultado guardado sin llamar al modelo.
3. **Usar el motor eco como filtro previo.** Si una regla clasifica con claridad, no hace falta gastar segundos de inferencia; el modelo se usaría solo cuando las reglas no deciden.
4. **Limitar la concurrencia hacia el modelo,** porque el modelo local atiende una petición a la vez y varias simultáneas se acumularían.
