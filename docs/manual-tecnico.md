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
