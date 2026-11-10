# Runners de GitHub Actions en el cluster (ARC)

El job `checks` de `grupo5-mobile` (tests, typecheck, lint y formato) corre en el cluster con [Actions Runner Controller](https://github.com/actions/actions-runner-controller) (ARC), en lugar de usar los runners de GitHub. Flux lo despliega desde `manifests/arc-runners.yaml`.

## Cómo funciona

- **Controller** (namespace `arc-systems`): gestiona los scale sets.
- **Listener** (`arc-systems`): mantiene una conexión saliente con GitHub y espera jobs. No hace falta exponer nada del cluster.
- **Runners** (`arc-runners`): por cada job con `runs-on: grupo5-mobile`, ARC crea un pod efímero que ejecuta el job y se borra al terminar. Pueden correr hasta `maxRunners` jobs a la vez.

| Componente | Valor |
|---|---|
| Charts | `gha-runner-scale-set-controller` y `gha-runner-scale-set` 0.15.0 |
| Imagen del runner | `ghcr.io/actions/actions-runner:2.338.2` |
| Scale set (`runs-on`) | `grupo5-mobile` |
| Runners | `minRunners: 0`, `maxRunners: 2` |
| Recursos por runner | requests 250m / 1Gi / 4Gi de disco; limits 2 CPU / 3Gi |

### Nodo dedicado

Los runners corren solo en `grupo5-7` (m1.large: 4 vCPU / 8 GiB), reservado para CI con un label y un taint. El `template.spec` del runner tiene el `nodeSelector` y la toleration que corresponden. Así, un job pesado no compite por memoria con Cassandra en los nodos `weak`. El resto de los nodos, incluido `grupo5-6`, quedan para cargas generales.

### Seguridad

Los runners ejecutan el código de los PRs. La NetworkPolicy `solo-internet` les permite solo DNS y salida a internet (GitHub, npm). Bloquea 10/8, 172.16/12, 192.168/16 y 169.254/16, así que no llegan a Cassandra, SeaweedFS, Prometheus, a los nodos ni a la metadata de OpenStack.

## Instalación

### 1. Reservar el nodo para CI

```sh
kubectl label node grupo5-7 ci=true
kubectl taint node grupo5-7 ci=true:NoSchedule
```

El taint no desaloja a los DaemonSets que ya están corriendo.

### 2. Crear el token de GitHub

Se usa un PAT *fine-grained*: GitHub → Settings → Developer settings → Fine-grained tokens.

- **Resource owner:** `FING-middlewares-distribuidos-2026`.
- **Repository access:** *Only select repositories* → `grupo5-mobile`.
- **Repository permissions:** *Administration: Read and write*. *Metadata: Read-only* se agrega sola.

Quien crea el token tiene que ser admin del repo. Si lo es, en Settings → Actions → Runners aparece el botón "New self-hosted runner". No se usan los comandos `config.sh` de esa página: ARC registra los runners por su cuenta.

### 3. Desplegar

Mergear `manifests/arc-runners.yaml` a `main`. Flux crea los namespaces, el controller y el scale set.

### 4. Cargar el token en el cluster

```sh
kubectl -n arc-runners create secret generic arc-github-token --from-literal=github_token=<PAT>
```

El listener arranca cuando existe el Secret. El token nunca se versiona.

### 5. Usarlo en el workflow

```yaml
jobs:
  checks:
    runs-on: grupo5-mobile
```

## Verificación

```sh
kubectl -n flux-system get helmreleases arc-controller grupo5-mobile-runners   # Ready
kubectl -n arc-systems get pods                                                # controller y listener Running
kubectl -n arc-runners get pods -o wide -w                                     # durante un job: pod en grupo5-7
```

En GitHub → `grupo5-mobile` → Settings → Actions → Runners tiene que aparecer el scale set `grupo5-mobile`.

## Mantenimiento

- **Rotar el PAT:** al vencer el token, los jobs quedan en cola. Generar un PAT nuevo y recrear el Secret con `kubectl -n arc-runners delete secret arc-github-token` y el comando del paso 4.
- **Ajustar recursos:** mirar `kubectl top node grupo5-7` durante los jobs y ajustar `resources` o `maxRunners` en el HelmRelease.
- **Sumar otro repo** (por ejemplo, `grupo5-webapp`): agregar otro `HelmRelease` igual a `grupo5-mobile-runners`, cambiando `githubConfigUrl` y `runnerScaleSetName`. Puede reusar el mismo Secret si el PAT incluye ese repo. Corre en el mismo nodo, así que hay que revisar que la suma de `maxRunners` entre en 8 GiB.

## Desinstalar

1. Volver `runs-on` a `ubuntu-latest` en los workflows.
2. Borrar `manifests/arc-runners.yaml`. Flux elimina los recursos (`prune: true`).
3. Borrar el Secret si quedó y revocar el PAT.
4. Liberar el nodo:
   ```sh
   kubectl taint node grupo5-7 ci=true:NoSchedule-
   kubectl label node grupo5-7 ci-
   ```
