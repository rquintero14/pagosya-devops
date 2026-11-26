# Laboratorio N.º 15 — Caso de estudio: el despliegue del viernes

Semana 14+ · Unidad III · **1 sesión**

Un despliegue manual dejó a PagosYA con el servicio caído **3 h 50 min**. En
este laboratorio se reconstruye ese despliegue, se rompe igual que el viernes y
se vuelve a la versión anterior en segundos, sobre la misma API del caso y con
Docker.

> La teoría de la sesión está en
> [`../Teoria-DevOps-Despliegue.pptx`](../Teoria-DevOps-Despliegue.pptx) (o en
> texto: [`../TEORIA-DEVOPS-DESPLIEGUE.md`](../TEORIA-DEVOPS-DESPLIEGUE.md)).

## 1. El caso

**PagosYA** es una API de pagos pequeña: tres personas, un servidor con Docker y
Compose, y ningún pipeline. Desplegar consistía en que **Ana** entrara por SSH,
hiciera `git pull` y ejecutara `docker compose up -d --build`. Funcionó ocho
meses.

| Hora | Qué pasó |
|------|----------|
| 15:30 | Se aprueba el PR «descuentos de temporada» y se fusiona a `main`. |
| 15:45 | Ana construye la imagen **en su laptop** y copia el `compose.yml` al servidor. |
| 16:10 | En el servidor: `git pull` + `compose up -d --build`. La app nueva arranca y **se cae** a los segundos. |
| 16:20 | Un cliente reporta **error 502** al pagar: el `nginx` de adelante no tiene a quién reenviar. |
| 16:35 | El contenedor **reinicia en bucle**. En la laptop de Ana funcionaba. |
| 17:00 | Deciden «volver atrás». Nadie sabe qué imagen corría: todo se publicó como `latest`. |
| 17:40 | Reconstruir desde el commit anterior **no compila**: las dependencias no están fijadas. |
| 18:30 | Restauran el respaldo de la base de datos de las 2:00 a. m.: se pierden **8 horas de pagos**. |
| 20:10 | El servicio queda estable. **Tiempo total caído: 3 h 50 min.** |

No falló el código del PR. Faltaban tres piezas:

1. Una **imagen versionada** (hoy todo es `latest`).
2. Un **tag explícito** en el `compose.yml`, para desplegar un artefacto concreto.
3. Un **registro de la última versión buena**, para volver sin reconstruir.

## 2. Los cinco flujos del despliegue

| # | Flujo | Origen → destino | Puerto | Si se bloquea |
|---|-------|------------------|--------|----------------|
| 1 | Código | el host → servidor Git | 22/443 | no hay commit |
| 2 | Push | daemon → registry | 5000/TCP | el artefacto no sale |
| 3 | Pull | servidor → registry | 5000/TCP | no encuentra la imagen |
| 4 | Servicio | cliente → app publicada | 8080 → 8000 (**NAT**) | `RST` / sin respuesta |
| 5 | Interno | app → base de datos | 5432/TCP · DNS `db` | arranca y falla al conectar |

En la práctica se recorren los flujos 2, 3 y 4: el registry, el artefacto y
el puerto publicado son donde el caso se rompió.

## 3. El sistema que se va a desplegar

La carpeta trae la API de PagosYA y su despliegue:

| Archivo | Qué es |
|---------|--------|
| `app/app.py` | La API: responde `/` con la versión y `/health` con `ok` |
| `app/Dockerfile` | Mete la versión en `/app/VERSION` con `ARG APP_VERSION` |
| `compose.yml` | Publica `8080` del host → `8000` del contenedor, usa `TAG` e `init: true` |

La práctica se escribe comando por comando, sin scripts.

## ✅ Lo que necesitas

- Docker Engine con el plugin **Compose v2** (`docker compose version`).
- `curl`.
- `git` (para el paso 11).
- Linux, macOS o WSL. En Windows nativo los comandos cambian.

No hace falta haber hecho los laboratorios anteriores: la carpeta trae la API y
su despliegue completos.

> La primera vez, Docker descarga las dos imágenes base (`python:3.11-slim` y
> `registry:2`), así que necesitas conexión para eso. Después todo el
> laboratorio corre en tu máquina.

---

# Guía paso a paso

## Paso 0 · Prepara la carpeta y el registry

En el caso las imágenes vivían sueltas en el servidor. Aquí levantas un registry
propio:

```bash
# Desde la raíz del repo del curso
cp -r DevOps/Laboratorio-15-DevOps-Despliegue ~/pagosya-devops
cd ~/pagosya-devops
docker run -d --name registry -p 127.0.0.1:5000:5000 registry:2
```

Debes ver un ID de contenedor y, al comprobarlo:

```bash
docker ps --filter name=registry
# registry  Up ...  127.0.0.1:5000->5000/tcp
```

> `127.0.0.1` es la **loopback**: el registry solo es alcanzable desde tu
> máquina. Con `-p 5000:5000` quedaría publicado en todas las interfaces.

## Paso 1 · Ponle nombre a la versión

```bash
TAG=v1
echo "$TAG"
# v1
```

> En el caso el tag salía del commit (`git rev-parse --short HEAD`). Aquí lo
> eliges tú para tener el control de qué versión es cuál.

## Paso 2 · Construye la imagen

En el caso la imagen se construía en la laptop de Ana. Aquí se construye desde
la carpeta, para que salga del mismo sitio todas las veces:

```bash
docker build --build-arg APP_VERSION="$TAG" \
  -t localhost:5000/pagosya-api:$TAG app/
```

Debes ver al final líneas como:

```
#9 naming to localhost:5000/pagosya-api:v1 done
#9 DONE 0.1s
```

> El `ARG` se **hornea** en la imagen: cambiar el tag cambia el contenido de
> `/app/VERSION`. Por eso el smoke test puede comparar lo que sirve con lo que
> se desplegó.

## Paso 3 · Comprueba el artefacto

```bash
docker run --rm localhost:5000/pagosya-api:$TAG cat /app/VERSION
# v1
```

Si responde otra cosa, la imagen no es la que crees: no sigas.

## Paso 4 · Publícala en el registry

```bash
docker push localhost:5000/pagosya-api:$TAG
curl -s localhost:5000/v2/_catalog
# {"repositories":["pagosya-api"]}
```

Debes ver el digest de la imagen y el catálogo con `pagosya-api`.

## Paso 5 · Despliega ese tag explícito

```bash
TAG="$TAG" docker compose up -d
# Container pagosya-devops-api-1  Started
```

Comprueba la traducción de puertos:

```bash
docker port pagosya-devops-api-1
# 8000/tcp -> 0.0.0.0:8080
```

> El contenedor se llama `<nombre-de-la-carpeta>-api-1`. Si copiaste la carpeta
> con otro nombre, usa ese (`docker compose ps` lo muestra).

> `8080 → 8000` es **DNAT**: el puerto del host se traduce al del contenedor.

## Paso 6 · Smoke test (la puerta que el caso no tuvo)

```bash
curl localhost:8080/
# PagosYA api v1
```

Si sale vacío, espera 2 segundos y repite: la app tarda en arrancar.

> En el caso lo detectó un **cliente** a los 10 minutos. Este `curl` prueba el
> **camino**: host:8080 → DNAT → contenedor:8000 → proceso → HTTP, y decide si
> el despliegue pasó.

## Paso 7 · Registra la última versión buena

```bash
echo "$TAG" > .last-good
cat .last-good
# v1
```

> Esto es lo que nadie tenía a las 17:00 del viernes: una respuesta escrita a
> «¿qué versión funcionaba?».

## Paso 8 · Rompe el despliegue (como el viernes)

Cambia el puerto interno de la app y despliega como `v2`:

```bash
sed -i 's/8000/9000/' app/app.py
# En macOS: sed -i '' 's/8000/9000/' app/app.py
TAG=v2
docker build --build-arg APP_VERSION="$TAG" \
  -t localhost:5000/pagosya-api:$TAG app/
docker push localhost:5000/pagosya-api:$TAG
TAG="$TAG" docker compose up -d
```

Ahora el smoke falla:

```bash
curl localhost:8080/
# curl: (56) Recv failure: Conexión reinicializada por la máquina remota
```

Y la red te dice por qué:

```bash
docker port pagosya-devops-api-1
# 8000/tcp -> 0.0.0.0:8080      ← la regla sigue apuntando a 8000

docker network inspect pagosya-devops_default \
  --format '{{range .IPAM.Config}}{{.Subnet}} gw {{.Gateway}}{{end}}'
# 172.23.0.0/16 gw 172.23.0.1   ← tu subred puede variar

ss -ltnp | grep -E ':(5000|8080)'
# 0.0.0.0:8080 (app) · 127.0.0.1:5000 (registry)
```

> `docker ps` dirá `Up`, pero el DNAT apunta a `8000` y la app escucha en
> `9000`. El kernel cierra con `RST` una conexión a un puerto sin listener: es
> un problema de **puerto**, no de ruta ni de DNS.

## Paso 9 · Vuelve a la versión anterior

```bash
TAG=$(cat .last-good) docker compose up -d
curl localhost:8080/
# PagosYA api v1
```

Tarda **menos de 2 segundos**: no se reconstruye nada, se elige una imagen que
ya está en el registry. En el caso eso tomó 3 h 50 min y costó 8 horas de pagos.

> El `init: true` del `compose.yml` es lo que hace que sea así de rápido: sin
> él, Python queda como PID 1, ignora el `SIGTERM` y `docker stop` espera 10 s.

## Paso 10 · Deja la evidencia

```bash
curl -s localhost:5000/v2/pagosya-api/tags/list
# {"name":"pagosya-api","tags":["v2","v1"]}
```

La versión mala **no se borra**: queda en el registry como evidencia. Volver
atrás fue elegir `v1`, no reconstruirla.

---

## Paso 11 · Tu repositorio y tu pipeline

Los pasos anteriores los escribiste a mano, uno por uno. Este paso los junta en
un repositorio y en un script que se pueda repetir, y que vuelva atrás solo
cuando el despliegue falla.

### 11.1 · Crea el repositorio

```bash
cd ~/pagosya-devops
git init
printf '.last-good\n' > .gitignore
git add .
git commit -m "API de PagosYA y su compose"
git log --oneline
```

`git init` basta: no necesitas GitHub. El repositorio guarda el código; el
registry sigue guardando las imágenes.

### 11.2 · Escribe el pipeline

Crea `deploy.sh` en la raíz del repo. Hace lo mismo que los pasos 1 al 9, en
orden, y decide solo:

```bash
#!/usr/bin/env bash
set -euo pipefail

TAG="${1:-$(git rev-parse --short HEAD)}"
IMAGE="localhost:5000/pagosya-api:$TAG"

docker build --build-arg APP_VERSION="$TAG" -t "$IMAGE" app/
docker push "$IMAGE"
TAG="$TAG" docker compose up -d

for i in $(seq 1 30); do
  if curl -fsS localhost:8080/ >/dev/null 2>&1; then
    echo "$TAG" > .last-good
    echo "OK $TAG"
    exit 0
  fi
  sleep 0.5
done

echo "smoke falló con $TAG" >&2
if [ -f .last-good ]; then
  TAG="$(cat .last-good)" docker compose up -d
  echo "vuelto a $(cat .last-good)" >&2
fi
exit 1
```

```bash
chmod +x deploy.sh
git add deploy.sh .gitignore
git commit -m "Pipeline de despliegue"
```

Lo que el script tiene y el caso no tenía:

- `set -euo pipefail`: si un comando falla, el pipeline se detiene ahí.
- El tag sale del commit (`git rev-parse --short HEAD`), y se puede pasar a mano
  como argumento.
- El `curl` al puerto publicado decide si el despliegue pasó.
- El rollback está escrito en el script, así que no depende de quién esté de
  guardia.

### 11.3 · Despliega con el script

```bash
./deploy.sh
# OK 1a2b3c4
cat .last-good
# 1a2b3c4
```

El tag ahora es el commit: cualquiera que mire el repo sabe qué código corre.

### 11.4 · Rompe y deja que el pipeline vuelva solo

```bash
sed -i 's/8000/9000/' app/app.py
# En macOS: sed -i '' 's/8000/9000/' app/app.py
git add -A && git commit -m "cambio el puerto interno"
./deploy.sh
echo "código de salida: $?"
```

Debes ver el smoke fallando, el mensaje de vuelta al tag anterior y el código de
salida `1`. Después:

```bash
curl localhost:8080/
# PagosYA api 1a2b3c4
```

El servicio queda en la versión anterior.

### 11.5 · Levántalo desde un clon

Para comprobar que el repo es suficiente y no depende de tu carpeta, baja el
stack, arregla el puerto en el clon y despliega desde otro directorio:

```bash
cd ~/pagosya-devops
docker compose down
git clone ~/pagosya-devops /tmp/pagosya-clon
cd /tmp/pagosya-clon
git revert --no-edit HEAD      # deshace el commit que rompió el puerto
./deploy.sh
```

Si funciona desde el clon, el despliegue está completo en el repo. Si no, algo
quedó fuera: un archivo sin commitear, una ruta absoluta o un `.gitignore` de más.

---

## Reto · Tu pipeline en GitHub Actions

El `deploy.sh` corre porque tú lo lanzas. El reto es que lo lance GitHub cuando
subes un commit, y para eso hace falta un repositorio en GitHub. El `.yml` va
incompleto a propósito: las respuestas están en la documentación de Actions.

Crea `.github/workflows/deploy.yml` en tu repo:

```yaml
name: deploy

on:
  # TODO: qué evento dispara el pipeline y en qué rama

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # TODO: iniciar sesión en el registry (¿qué acción?, ¿qué secretos?)
      # TODO: construir la imagen con el tag del commit
      # TODO: arrancar el contenedor y correr el smoke test
      # TODO: publicar la imagen solo si el smoke pasó
```

Lo que tienes que averiguar por tu cuenta:

- Qué evento usar para que un push a `main` dispare el job, y cómo evitar que dos
  despliegues corran a la vez.
- Dónde se guardan el usuario y el token del registry, y cómo se leen desde el
  `.yml` sin escribirlos en el archivo.
- Cómo obtener el hash del commit dentro del workflow y recortarlo a 7 caracteres.
- Cómo levantar el contenedor dentro del runner y esperar a que responda en
  `localhost:8080`.
- Cómo hacer que el job quede en rojo cuando el smoke falla, para que el push
  quede marcado.

Dónde buscar: la pestaña *Actions* de tu repo, la documentación de
`actions/checkout`, la de *secrets* y la de los eventos `push`. Si te trabas, el
error del job dice el paso y la línea.

Entregable: el `.yml` en el repo, el enlace a una ejecución en verde y otra en
rojo, y una línea por cada `TODO` explicando qué pusiste y por qué.

---

## 🧹 Limpieza

```bash
docker compose down
docker rm -f registry
```

## 🔧 Si algo falla

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Otro contenedor usa 8080 | `docker ps`, y detén el que lo ocupa |
| `Bind for 127.0.0.1:5000 failed` | Ya hay un registry corriendo | `docker rm -f registry` y repite el paso 0 |
| `pull access denied` al hacer `compose up` | La imagen no está en el registry | Repite el paso 4 y revisa el tag |
| El smoke responde un tag viejo | El contenedor no se recreó | `TAG="$TAG" docker compose up -d --force-recreate` |
| `curl: (56) Recv failure` | El puerto interno no coincide con el DNAT | `docker port` y revisa el puerto en `app.py` |
| `cat: .last-good: No such file` | Nunca hubo un despliegue verde | Completa los pasos 5-7 antes de romper |
| `permission denied` al usar Docker | Tu usuario no está en el grupo `docker` | Usa `sudo` o agrega el usuario al grupo |

## 📝 Qué entregas

1. **Bitácora** de los 10 pasos con el tag real en cada uno.
2. **Mapa de los 5 flujos**: origen → destino, puerto y qué se rompe si se bloquea.
3. **Evidencia**: salida de `_catalog`, el tag en `curl`, `docker port`/`ss`, el
   smoke en rojo y el tiempo del rollback.
4. **Postmortem del caso** (una página): línea de tiempo, los 5 porqués y las tres
   piezas que faltaban. Cierra comparando el caso con tu práctica: 3 h 50 min
   frente a tu rollback medido, y **qué flujo de red** falló en cada uno.
5. **Repositorio Git** con `deploy.sh` y el historial de los dos despliegues
   (`git log --oneline`).
6. **Salida del pipeline** en verde y en rojo: el tag desplegado, el smoke
   fallando, el rollback automático y el código de salida.
7. **Reto de Actions**: el `.yml`, una ejecución en verde y otra en rojo, y la
   explicación de cada `TODO`.

> Se evalúa la explicación de red, no la memorización: quien pega los comandos
> pero no sabe qué puerto abrió, no aprueba.
#   p a g o s y a - d e v o p s  
 