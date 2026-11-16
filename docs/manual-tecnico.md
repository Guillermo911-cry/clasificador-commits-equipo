# Manual técnico - Clasificador de commits

Este documento explica cómo está armado el proyecto. Lo vamos completando cada semana.

## 1. Arquitectura

El cliente (el navegador con /docs o Postman) manda una petición a la API. La API decide qué motor usar, guarda el resultado en la base de datos y responde con el tipo de commit.

```
Cliente (navegador / Postman)
          |
          |  HTTP puerto 8000
          v
     API FastAPI
     /health  /clasificar  /inferencias
          |
    +-----+------+
    |            |
Motor eco     Motor ollama
(reglas)         |
    |            v
    |      Ollama puerto 11434
    |      (modelo qwen2.5:0.5b)
    |            |
    +-----+------+
          |
          v
 PostgreSQL puerto 5432
 (contenedor db-ia, base iadb)
          |
          v
 Respuesta JSON: tipo y latencia
```

Puertos que usamos:

- 8000: la API
- 11434: Ollama
- 5432: PostgreSQL

El motor eco clasifica con palabras clave (por ejemplo "corrige" o "error" da fix). No usa modelo, entonces responde casi al instante. El motor ollama usa el modelo local y es mucho más lento. En una prueba que hicimos con el mismo mensaje, el eco respondió en 0 ms y ollama en unos 7 segundos.

## 2. Seguridad

### Puertos expuestos

El 8000 se expone porque por ahí entran las peticiones a la API. El 5432 lo dejamos abierto solo para trabajar en local y poder entrar con psql; en un servidor real no debería quedar publicado. El 11434 solo lo usa la API desde la misma máquina.

### Roles de la base de datos

Hay dos roles:

- **postgres**: es el administrador. Se usa para crear la tabla y para los respaldos. La aplicación no se conecta con este.
- **app_ia**: es el usuario de la aplicación. Solo puede consultar e insertar en la tabla inferencias. No puede borrar, actualizar ni eliminar la tabla. Así, si alguien lograra usar las credenciales de la API, el daño sería menor (privilegio mínimo).

### Manejo de secretos

Las contraseñas no están en el código. Están en el archivo .env, que está en el .gitignore y no se sube al repositorio. En el repo solo va el .env.example, que tiene los nombres de las variables sin valores, para que otra persona sepa qué debe llenar.

### Si se filtra una contraseña

1. Cambiarla de una vez en la base de datos con ALTER ROLE.
2. Actualizar el .env y reiniciar la API.
3. Si se subió a GitHub, darla por perdida aunque se borre el archivo, porque queda en el historial de commits. Hay que cambiarla igual y limpiar el historial.
4. Revisar la tabla inferencias por si hubo registros raros.

## 3. Endpoints

| Método | Ruta | Entrada | Respuesta exitosa | Errores |
|---|---|---|---|---|
| GET | /health | Ninguna | 200 {"estado":"ok","base_datos":"ok"} | 503 si la base de datos no está disponible |
| POST | /clasificar | JSON con texto (obligatorio) y motor (opcional: "eco" u "ollama") | 200 con motor, modelo, entrada, tipo y latencia_ms | 400 si el motor no es eco ni ollama. 422 si falta texto o el JSON es inválido. 500 si Ollama no responde |
| GET | /inferencias | Parámetro limite (entero, por defecto 20) | 200 con la lista de las últimas inferencias, de la más nueva a la más vieja | 422 si limite no es un número |

Si no se envía motor, se usa el valor de MOTOR_POR_DEFECTO del .env (eco).

Ejemplo:

```
curl -X POST http://localhost:8000/clasificar -H "Content-Type: application/json" -d '{"texto":"corrige el error de login","motor":"eco"}'
```

## 4. Modelo de datos

Tabla inferencias:

| Campo | Tipo | Significado |
|---|---|---|
| id | SERIAL, clave primaria | Identificador automático de cada inferencia |
| fecha | TIMESTAMP, por defecto NOW() | Momento en que se registró |
| motor | VARCHAR(20) | Motor usado: eco u ollama |
| modelo | VARCHAR(120) | Modelo usado (reglas-v1 o el nombre del modelo de Ollama) |
| entrada | TEXT | Mensaje de commit que envió el usuario |
| salida | TEXT | Categoría devuelta |
| latencia_ms | INTEGER | Milisegundos que tardó la clasificación |

## 5. Respaldo y restauración

Procedimiento probado desde la carpeta del proyecto con la solución levantada:

```
# 1) Crear el respaldo
mkdir -p backups
docker compose exec -T db pg_dump -U postgres iadb > backups/respaldo_2026-11-16.sql | docker compose exec -T db psql -U postgres -d iadb

# 2) Restaurar
cat backups/respaldo_FECHA.sql | docker compose exec -T db psql -U postgres -d iadb
```

Prueba realizada: antes del desastre simulado la tabla tenía 50 registros. Se ejecutó TRUNCATE inferencias y el conteo quedó en 0. Después de restaurar, el conteo volvió a 50.

Detalle a tener en cuenta: el respaldo incluye CREATE TABLE, así que restaurarlo sobre una base que ya tiene la tabla puede mostrar avisos de "already exists". Para una restauración completa conviene hacerla sobre una base vacía.

- **Periodicidad propuesta:** una vez al día, y antes de cualquier cambio importante.
- **Responsable:** Guillermo Leon Salazar Taborda.
- Los archivos .sql nunca se suben al repositorio porque contienen datos. La carpeta backups/ está en .gitignore.

## 6. Decisiones de diseño

- **Dockerfile multi-etapa:** la primera etapa instala las dependencias y la segunda solo copia lo instalado y el código. La imagen final queda más liviana (unos 223 MB) y sin herramientas de construcción.
- **Usuario sin privilegios (appuser):** si alguien lograra ejecutar código dentro del contenedor, no tendría permisos de administrador.
- **Motor eco:** funciona sin modelo ni memoria extra, así que la solución se puede desplegar y probar en cualquier equipo. También es la línea base para comparar con el modelo y permite la prueba de carga del servicio sin mezclarla con la latencia de la inferencia.
- **Privilegios mínimos en la base:** la API solo necesita consultar e insertar. Si se filtran sus credenciales, no se pueden borrar ni modificar los registros.
- **Healthcheck en la base y depends_on:** la API no arranca hasta que PostgreSQL está lista.
- **Credenciales por variables de entorno:** ninguna contraseña está en el código fuente.

## 7. Limitaciones conocidas

- **Calidad del modelo:** qwen2.5:0.5b acertó 4 de 10 en nuestra prueba y respondió "feat" en 7 de 10 casos. El motor eco es más confiable en mensajes con palabras clave claras.
- **Latencia:** el motor ollama tardó unos 4,8 s en promedio (eco: 25 ms), en buena parte porque OLLAMA_KEEP_ALIVE=0 descarga el modelo después de cada respuesta.
- **Manejo de errores de Ollama:** si Ollama está apagado o inalcanzable, /clasificar responde 500 sin un mensaje claro.
- **Sin autenticación:** la API no pide credenciales; cualquiera que llegue al puerto 8000 puede usarla.
- **Ollama expuesto en la red local** por usar 0.0.0.0, y fuera de Docker, por lo que no queda dentro del docker-compose.
- **Una conexión nueva por petición:** no se usa pool de conexiones, lo que limita el rendimiento con mucha carga.
- **Datos personales:** se guarda el texto completo que envía el usuario, que podría contener datos personales (Ley 1581 de 2012). No hay política de retención ni borrado, porque app_ia no puede eliminar registros.
- **Contraseñas de ejemplo** del curso: en un entorno real deben reemplazarse por contraseñas fuertes y únicas.
EOF
