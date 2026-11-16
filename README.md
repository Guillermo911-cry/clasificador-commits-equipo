# clasificador-commits-equipo
# Clasificador de mensajes de commit

Servicio web que recibe el texto de un mensaje de commit de Git y responde a qué tipo corresponde según la convención *Conventional Commits* (`feat`, `fix`, `docs`, `test`, `chore` o `refactor`). Puede clasificar con un **motor eco** (reglas de palabras clave, sin modelo) o con un **motor ollama** (un LLM pequeño que corre en la propia máquina). Cada petición queda registrada en PostgreSQL, y toda la solución está contenerizada con Docker, se levanta con un solo comando y es validada por un pipeline de integración continua en GitHub Actions. Es el proyecto integrador del curso *Implementación de Soluciones de Inteligencia Artificial* (SENA, código 21730210).

## Integrantes y hardware

| Integrante | Rol / tareas principales |
|---|---|
| Guillermo Leon Salazar Taborda | Individual, Toda la guia |

**Perfil de hardware con el que se trabajó:** Perfil B (8 GB de RAM), con el modelo `qwen2.5:0.5b`.

## Requisitos mínimos

**Hardware**

- 8 GB de RAM
- 15 GB libres en disco
- Virtualización habilitada en el BIOS (solo si se usa Windows con WSL2 como es mi caso)

**Software**

- Ubuntu 24.04 (nativo o mediante WSL2 en Windows 10 versión 2004 o superior)
- Git, Docker Engine y Docker Compose (los instala `setup.sh`)
- Ollama (lo instala `setup.sh`; solo es necesario para el motor `ollama`)
- Cuenta de GitHub con una llave SSH registrada

## Instalación paso a paso

### 1. Preparar el entorno Linux

**Si usa Windows**, abra PowerShell como administrador y ejecute:

```
wsl --install -d Ubuntu-24.04
```

Reinicie el equipo, cree el usuario de Linux cuando se le solicite y luego active systemd:

```
sudo nano /etc/wsl.conf
```

Escriba dentro del archivo:

```
[boot]
systemd=true
```

Guarde (`Ctrl + O`, Enter) y salga (`Ctrl + X`). En PowerShell ejecute `wsl --shutdown` y vuelva a abrir Ubuntu.

**Si usa Ubuntu nativo**, no necesita hacer nada: abra la terminal y continúe.

### 2. Clonar el repositorio

```
cd ~
git clone https://github.com/Guillermo911-cry/clasificador-commits-equipo.git
cd clasificador-commits-equipo
```


### 3. Instalar Docker, Git y Ollama con el script de aprovisionamiento

```
chmod +x setup.sh diagnostico.sh
./setup.sh
```

Al terminar, **cierre la terminal por completo y vuelva a abrirla** para poder usar `docker` sin `sudo`.

### 4. Descargar el modelo (solo para el motor ollama)

Configure Ollama para consumir poca memoria y para que los contenedores puedan alcanzarlo:

```
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/memoria.conf
```

Contenido del archivo:

```
[Service]
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_KEEP_ALIVE=0"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Aplique la configuración y descargue el modelo:

```
sudo systemctl daemon-reload
sudo systemctl restart ollama
ollama pull qwen2.5:0.5b
ollama list
```

### 5. Crear el archivo de variables de entorno

```
cp .env.example .env
nano .env
```

Complete los valores. Los de ejemplo del curso son:

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=iadb
DB_USER=app_ia
DB_PASSWORD=claveApp456
DB_ADMIN_PASSWORD=claveAdmin123
OLLAMA_URL=http://localhost:11434/api/generate
MODELO_OLLAMA=qwen2.5:0.5b
MOTOR_POR_DEFECTO=eco
```

> **Importante:** `DB_PASSWORD` debe coincidir con la contraseña del rol `app_ia` definida en `db/init.sql`. En un entorno real, use contraseñas fuertes y únicas. El archivo `.env` nunca se sube al repositorio.

### 6. Levantar la solución

```
docker compose up -d --build
docker compose ps
```

La primera construcción puede tardar varios minutos.

## Cómo verificar que quedó funcionando

`docker compose ps` debe mostrar los servicios `db-ia` y `api-ia` en estado *running* (la base de datos, *healthy*). Luego:

```
curl http://localhost:8000/health
```

Debe responder `{"estado":"ok","base_datos":"ok"}`.

### Probar los tres endpoints

Abra **http://localhost:8000/docs** en el navegador (documentación interactiva de FastAPI) o use `curl`.

**1. `GET /health`**: estado del servicio y de la base de datos.

```
curl http://localhost:8000/health
```

**2. `POST /clasificar`**: clasifica un mensaje de commit y registra la inferencia.

```
curl -X POST http://localhost:8000/clasificar \
  -H "Content-Type: application/json" \
  -d '{"texto":"corrige el error de login","motor":"eco"}'
```

Respuesta esperada (los milisegundos pueden variar):

```
{"motor":"eco","modelo":"reglas-v1","entrada":"corrige el error de login","tipo":"fix","latencia_ms":25}
```

Para usar el modelo, cambie `"motor":"eco"` por `"motor":"ollama"`. La primera respuesta puede tardar varios segundos mientras el modelo se carga en memoria.

**3. `GET /inferencias`**: últimas inferencias registradas.

```
curl "http://localhost:8000/inferencias?limite=5"
```

El historial debe crecer con cada petición a `/clasificar`.

### Pruebas automatizadas

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pytest -v
```

## Solución de problemas

| Mensaje o síntoma | Causa | Solución |
|---|---|---|
| Al instalar Ubuntu en Windows solo se instalaba WSL y Ubuntu no abría | Instalación incompleta | `wsl --list --verbose`; si Ubuntu no aparece, reinstale con `wsl --install -d Ubuntu-24.04` |
| `wsl --install` falla mencionando virtualización | VT-x o AMD-V desactivado en el BIOS | Reinicie, entre al BIOS (F2, F10, Supr o Esc) y active Intel VT-x o AMD-V |
| `permission denied ... docker daemon socket` | No se cerró la terminal después de `usermod -aG docker` | Cierre la terminal por completo y ábrala de nuevo |
| `System has not been booted with systemd` | systemd no está activado en WSL | Cree `/etc/wsl.conf` con `[boot]` y `systemd=true`, luego `wsl --shutdown` |
| `port is already allocated` | Otro contenedor usa el puerto | `docker ps` para ubicarlo y `docker rm -f <nombre>`, o cambie el puerto en `docker-compose.yml` |
| `psycopg2.OperationalError: connection refused` | La base de datos no está lista o el host es incorrecto | Revise `docker compose ps`. Dentro de Compose el host es `db`, no `localhost` |
| `FATAL: password authentication failed` | La contraseña del `.env` no coincide con `db/init.sql` | Compare ambos archivos. Para reiniciar desde cero: `docker compose down -v` |
| La API responde 500 con el motor ollama | Ollama detenido o modelo no descargado | `ollama list` y `sudo systemctl status ollama` |
| `Permission denied (publickey)` al hacer push | La llave SSH no está registrada en GitHub | Registre la llave en GitHub y pruebe con `ssh -T git@github.com` |



## Documentación adicional

- [Manual técnico](docs/manual-tecnico.md): arquitectura, seguridad, endpoints, modelo de datos, respaldo y decisiones de diseño.
- [Informe técnico](docs/informe-tecnico.md): caracterización del modelo, plan de pruebas, análisis del cuello de botella y escalamiento.

## Video de demostración

TODO:

## Versión

`v1.0.0`
