# Runner propio en el cluster para los checks de grupo5-mobile

## Contexto

El job `checks` de `grupo5-mobile/.github/workflows/mobile.yml` (tests, tsc, lint, prettier) corre en el cluster con Actions Runner Controller (ARC), gestionado por Flux como el resto de `middlewares-cluster-config`. El job `apk` sigue en `ubuntu-latest`, porque el build lo hace EAS.

**Por qué un nodo aparte (medido el 2026-10-28):**
- Los workers grupo5-2..5 (`nodegroup=weak`, 2 vCPU / 4 GiB) tienen ~3.7 GiB asignables y usan entre 2.7 y 3.1 GiB, así que quedan ~0.7–1.1 GiB libres.
- El job necesita ~1 GiB: Jest tuvo un pico de 516 MiB, lint 338 MiB, tsc 297 MiB, y hay que sumar ~150 MiB del runner.
- Cassandra pide 2Gi y puede usar hasta 4Gi. Si un nodo se queda sin memoria, el kubelet desaloja primero a los pods que usan más de lo pedido, así que Cassandra correría riesgo.

**Decisión de nodos.** Se sumaron dos nodos `m1.large` (4 vCPU / 8 GiB / 20 GiB de disco, label `nodegroup=strong`):
- **grupo5-7: exclusivo para CI.** Lleva label `ci=true` y taint `ci=true:NoSchedule`. Ahí solo corren los runners y los DaemonSets.
- **grupo5-6: uso general.** Sin taint. El scheduler prefiere los nodos con más recursos libres, así que las réplicas del HPA del backend y cualquier otra carga pueden ir ahí sin cambiar manifests.

**Credencial: PAT fine-grained.** No se puede usar una GitHub App porque instalarla en un repo de la organización requiere un owner. El token de registro que muestra la página "New self-hosted runner" de GitHub (`config.sh --token ...`) **no se usa**: es para instalar un runner a mano y vence en una hora. ARC registra los runners solo, con el PAT.

## Pasos manuales (antes del merge)

1. **Nodo CI:**
   - `kubectl label node grupo5-7 ci=true`
   - `kubectl taint node grupo5-7 ci=true:NoSchedule`

   El taint no desaloja a los DaemonSets que ya corren. metallb-speaker no lo tolera: si su pod en grupo5-7 se recrea, no vuelve. No afecta, porque los demás nodos anuncian las IPs.
2. **PAT fine-grained:** se crea en GitHub → Settings → Developer settings → Fine-grained tokens.
   - Resource owner: `FING-middlewares-distribuidos-2026`.
   - Repository access: *Only select repositories* → `grupo5-mobile`. Se puede sumar `grupo5-webapp` para reusarlo después.
   - Repository permissions → **Administration: Read and write**. Metadata: Read-only se agrega sola. Nada más.
   - El usuario que lo crea tiene que ser admin del repo. Se verifica porque en Settings → Actions → Runners aparece "New self-hosted runner".
   - Anotar la fecha de vencimiento: al rotarlo hay que recrear el Secret.
3. **Secret en el cluster** (después del merge, o antes con `kubectl create namespace arc-runners`):
   `kubectl -n arc-runners create secret generic arc-github-token --from-literal=github_token=<PAT>`

## Qué despliega `manifests/arc-runners.yaml`

- Namespaces `arc-systems` y `arc-runners`.
- `HelmRepository` OCI `oci://ghcr.io/actions/actions-runner-controller-charts`.
- `HelmRelease arc-controller`: chart `gha-runner-scale-set-controller` 0.15.0, release `arc`, `logLevel: info`.
- `HelmRelease grupo5-mobile-runners`: chart `gha-runner-scale-set` 0.15.0, con `dependsOn: arc-controller` (mismo patrón que `metallb-config`).
  - `runnerScaleSetName: grupo5-mobile`: es el valor de `runs-on` en el workflow, no `self-hosted`.
  - `minRunners: 0`, `maxRunners: 2`: dos jobs a la vez en grupo5-7.
  - Imagen `ghcr.io/actions/actions-runner:2.338.2`.
  - Requests de 250m / 1Gi / 4Gi de `ephemeral-storage`. Limits de 2 CPU / 3Gi.
  - `nodeSelector ci: "true"` y toleration del taint `ci=true:NoSchedule`.
- `NetworkPolicy solo-internet` (el CNI es Calico): solo permite DNS y salida a internet. Bloquea 10/8, 172.16/12, 192.168/16 y 169.254/16 (metadata de OpenStack), así que el código de los PRs no llega a Cassandra, SeaweedFS, Prometheus ni a los nodos.

En `grupo5-mobile/.github/workflows/mobile.yml`, el job `checks` usa `runs-on: grupo5-mobile`.

## Orden de despliegue

1. Pasos manuales 1–2.
2. Merge del PR de `middlewares-cluster-config`. Flux crea los namespaces y los HelmReleases.
3. Paso manual 3 (Secret). El listener arranca cuando el Secret existe.
4. Verificar que el runner aparece en GitHub.
5. Merge del PR de `grupo5-mobile`.

## Verificación

- Antes del merge: `kubectl apply --dry-run=server -f manifests/arc-runners.yaml` (no modifica nada).
- `kubectl describe node grupo5-7 | grep -A2 Taints` muestra `ci=true:NoSchedule`. grupo5-6 no tiene taints.
- `kubectl -n flux-system get helmreleases arc-controller grupo5-mobile-runners` en Ready. `kubectl -n arc-systems get pods` con el controller y el listener `grupo5-mobile-*` corriendo.
- En GitHub → grupo5-mobile → Settings → Actions → Runners aparece el scale set `grupo5-mobile`.
- Con un PR abierto, `kubectl -n arc-runners get pods -o wide -w` muestra los runners **solo** en grupo5-7. El job pasa y el pod se borra al terminar.
- `kubectl get pods -A -o wide --field-selector spec.nodeName=grupo5-7` muestra solo DaemonSets y runners.
- Durante los jobs, `kubectl top node grupo5-7` para ajustar limits y `maxRunners`.

## A futuro: checks de grupo5-webapp

Otro `HelmRelease` igual a `grupo5-mobile-runners`, con `githubConfigUrl` del webapp y `runnerScaleSetName: grupo5-webapp`. Reusa el mismo Secret si el PAT incluye ese repo. Corre en el mismo nodo CI: revisar `maxRunners` de los dos para que entren en 8 GiB.

## Fuera de alcance: compilar el APK en el cluster

El job `apk` manda el build a los servidores de Expo (`eas build`). Compilarlo en el cluster implica `expo prebuild` + Gradle (Expo SDK 57 / RN 0.86, con C++ nativo). Pediría ~4 vCPU / 8 GiB y 30–40 GiB de disco solo para ese job, más un keystore propio en un Secret y las variables del environment `preview` de EAS. Con el disco de 20 GiB de grupo5-7 no entra, así que se sigue usando EAS.

## Rollback

- Rápido: revertir `runs-on` a `ubuntu-latest` en grupo5-mobile.
- Completo: revertir el commit en `middlewares-cluster-config`. Flux borra todo porque está con `prune: true`. Después:
  - Borrar el Secret y revocar el PAT.
  - `kubectl taint node grupo5-7 ci=true:NoSchedule-`
  - `kubectl label node grupo5-7 ci-`
