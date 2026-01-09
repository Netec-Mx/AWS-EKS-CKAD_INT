---
layout: lab
title: "Práctica 21: Simulación de examen Kubernetes (reto estilo CKAD)"
permalink: /lab21/lab21/
images_base: /labs/lab21/img
duration: "120 minutos"
objective:
  - "Simular un mini-examen tipo CKAD en un clúster Amazon EKS resolviendo una serie de retos prácticos (pods, deployments, services, configuración, observabilidad, seguridad y networking) bajo presión de tiempo, validando cada entrega con comandos estilo examen y dejando evidencia reproducible en manifiestos YAML dentro de una carpeta de práctica."
prerequisites:
  - "Amazon EKS operativo y kubectl apuntando al contexto correcto"
  - "Windows + Visual Studio Code + Terminal GitBash integrado"
  - "Herramientas: kubectl, aws (si usas ECR), docker, git, curl"
  - "Permisos Kubernetes: crear namespaces, workloads, services, ingress, RBAC, networkpolicy, PVC, jobs/cronjobs"
  - "(Recomendado) Un Ingress Controller disponible (ingress-nginx o AWS Load Balancer Controller)"
  - "(Recomendado) CNI/add-on que **aplique** NetworkPolicy (si no, las policies no hacen enforcement)"
  - "(Recomendado) StorageClass por defecto para PVC dinámico"
cost:
  - "No hay costo directo por objetos Kubernetes; el costo real es EKS + nodos (EC2/EBS) mientras existan."
  - "PVC puede crear volúmenes (EBS) y un Ingress Controller tipo LoadBalancer puede crear un LB con costo."
introduction:
  - "Este laboratorio está diseñado como **reto**: primero ves qué debes lograr y cómo se validará (como en CKAD). Tu entrega final es un set de YAML reproducibles en `manifests/` (y, si aplica, scripts de validación)."
slug: lab21
lab_number: 21
final_result: >
  Al finalizar tendrás una carpeta con manifiestos reproducibles que resuelven un set de retos tipo CKAD en EKS: despliegues, servicios, configuración segura, probes, rolling updates y rollback, multi-container, ingress, network policies, workloads batch, persistencia con PVC y RBAC mínimo validado con `kubectl auth can-i`, junto con evidencia de verificación.
notes:
  - "CKAD: optimiza tu CLI (namespace por defecto, alias, dry-run) y valida rápido con describe/events/logs/top."
  - "Modo reto: prioriza cumplir el resultado y dejar evidencia. Evita ‘decoración’ innecesaria."
  - "NetworkPolicy e Ingress dependen de componentes del clúster: si no están, el YAML puede ‘crear’ pero no tendrá efecto real."
references:
  - text: "CNCF / Linux Foundation — Certified Kubernetes Application Developer (CKAD)"
    url: https://www.cncf.io/training/certification/ckad/
  - text: "Kubernetes — kubectl Cheat Sheet (atajos útiles en examen)"
    url: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
  - text: "Kubernetes — Services"
    url: https://kubernetes.io/docs/concepts/services-networking/service/
  - text: "Kubernetes — ConfigMaps"
    url: https://kubernetes.io/docs/concepts/configuration/configmap/
  - text: "Kubernetes — Secrets"
    url: https://kubernetes.io/docs/concepts/configuration/secret/
  - text: "Kubernetes — Probes (liveness/readiness/startup)"
    url: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
  - text: "Kubernetes — Deployments (rollout/rollback)"
    url: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
  - text: "Kubernetes — Multi-container Pods (patrones)"
    url: https://kubernetes.io/docs/concepts/workloads/pods/
  - text: "Kubernetes — Ingress"
    url: https://kubernetes.io/docs/concepts/services-networking/ingress/
  - text: "Kubernetes — Network Policies"
    url: https://kubernetes.io/docs/concepts/services-networking/network-policies/
  - text: "Kubernetes — Jobs / CronJobs"
    url: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
  - text: "Kubernetes — PersistentVolumeClaim"
    url: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims
  - text: "Kubernetes — RBAC / Authorization"
    url: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
prev: /lab20/lab20/
next: /lab1/lab1/
---

---

## Instrucciones (modo reto)

- Crea una carpeta para la práctica y guarda **todo** en YAML reproducible.
- Trabaja con enfoque “examen”:
  - usa `kubectl ... --dry-run=client -o yaml` para generar YAML rápido,
  - edítalo y **aplícalo** con `kubectl apply -f`.
- Cada reto incluye:
  - **Definición**
  - **Qué debes hacer**
  - **Resultado esperado**
  - **Validaciones**
- Tu entrega principal es el contenido de `manifests/` (y opcionalmente `scripts/validate.sh` y `notes/` con evidencias).

> **NOTA (CKAD):** La velocidad viene de reducir errores (namespace por defecto) y validar con **describe/events/logs**.
{: .lab-note .info .compact}

---

## Estructura de carpetas (obligatoria)

#### Tarea 0.1

- {% include step_label.html %} Abre VSCode y dentro del directorio **labs-eks-ckad** crea la estructura base de la simulación.

  ```bash
  mkdir -p lab21/{manifests,solutions,scripts,notes}
  cd lab21
  ```

- {% include step_label.html %} (Opcional) Crea un script de validación rápido (plantilla).

  ```bash
  cat > scripts/validate.sh <<'EOF'
  #!/usr/bin/env bash
  set -euo pipefail

  NS="${NS:-ckad-reto}"

  echo "== NS =="
  kubectl get ns "$NS"

  echo "== WORKLOADS =="
  kubectl -n "$NS" get deploy,po,svc,ingress || true

  echo "== CONFIG =="
  kubectl -n "$NS" get cm,secret || true

  echo "== SECURITY/NET =="
  kubectl -n "$NS" get networkpolicy,sa,role,rolebinding || true

  echo "== BATCH =="
  kubectl -n "$NS" get job,cronjob || true

  echo "== STORAGE =="
  kubectl -n "$NS" get pvc || true

  echo "OK"
  EOF

  chmod +x scripts/validate.sh
  ```

#### Tarea 0.2 Crear un clúster EKS para la simulacion

Crearas un cluseter rapido y directo.

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  eksctl create cluster \
    --name "sim-exam-eks" \
    --region "us-west-2" \
    --version "1.33" \
    --managed \
    --nodegroup-name "mgn1" \
    --node-type "t3.medium" \
    --nodes 2 --nodes-min 1 --nodes-max 3
  ```

---

## Pre-check recomendado (no puntúa, evita sorpresas)

Ejecuta esto una sola vez para saber qué *sí* puedes validar en tu clúster:

```bash
kubectl get nodes -o wide
kubectl get storageclass
kubectl get pods -A | egrep -i "ingress|aws-load-balancer|nginx" || true
kubectl api-resources | egrep -i "networkpolicy|ingress" || true
```

> Si no hay Ingress Controller o no hay enforcement de NetworkPolicy, **aún puedes crear** los recursos (YAML correcto), pero no podrás validar el comportamiento real de tráfico. Documenta esa limitación en `notes/`.
{: .lab-note .warning .compact}

---

# Retos del mini-examen (12 retos)

> **Regla del reto:** cada entrega debe quedar guardada en `manifests/` con un nombre claro, por ejemplo `01-namespace.yaml`, `02-webapp-deploy.yaml`, etc.
{: .lab-note .info .compact}

---

## Reto 1: Preparación tipo examen (namespace + contexto) (4 min)

**Definición**  
Preparar un entorno “limpio” para trabajar rápido y sin errores.

**Qué debes hacer**
- Crear el namespace `ckad-reto`.
- Ajustar el contexto actual para que use ese namespace por defecto.
- (Opcional) Crear alias `k=kubectl` en la sesión.

**Archivos a crear (para score)**
- *(No obligatorio)*

**Resultado esperado**
- Namespace creado y activo.
- Comandos operan por defecto en `ckad-reto`.

**Validaciones**
```bash
kubectl get ns ckad-reto
kubectl config view --minify | grep namespace
```

> **NOTA (CKAD):** perder tiempo por olvidar `-n` es muy común.
{: .lab-note .info .compact}

---

## Reto 2: Construcción de imagen y Deployment (12 min)

**Definición**  
Crear una imagen simple y usarla en Kubernetes.

**Qué debes hacer**
- Crear una app mínima HTTP (p.ej., Nginx con `index.html` custom).
- Crear el repositorio en Amazon ECR **Opcional**.
- Construir imagen con `docker build`.
- Publicarla en un registry utilizable (ECR o Docker Hub).
- Desplegar un Deployment `webapp` con 2 réplicas usando esa imagen.
- Asegurar label `app=webapp` en el Deployment/Pods.

**Archivos a crear (para score)**
- `manifests/02-webapp-deploy.yaml` (Deployment `webapp`)

**Resultado esperado**
- `Deployment/webapp` disponible con 2 Pods listos.

**Validaciones**
```bash
kubectl -n ckad-reto get deploy webapp
```
```bash
kubectl -n ckad-reto get pods -l app=webapp -o wide
```
```bash
kubectl -n ckad-reto describe deploy webapp | egrep -n "Image:|Replicas:" -A1 -B1
```

> Si no puedes publicar imagen (restricciones de red/registry), documenta el motivo en `notes/`.
{: .lab-note .warning .compact}

---

## Reto 3: Service + prueba de conectividad (8 min)

**Definición**  
Exponer el Deployment internamente.

**Qué debes hacer**
- Crear `Service/ClusterIP` `webapp-svc` (port 80 → targetPort 80).
- Asegurar selector `app=webapp`.
- Probar desde un Pod “tester” que responda contenido.

**Archivos a crear (para score)**
- `manifests/03-webapp-svc.yaml` (Service `webapp-svc`)

**Resultado esperado**
- DNS/Service funcionando dentro del clúster.

**Validaciones**
```bash
kubectl -n ckad-reto get svc webapp-svc
```
```bash
kubectl -n ckad-reto run tester --image=curlimages/curl:8.5.0 -it --rm --restart=Never -- \
  curl -sS http://webapp-svc | head
```

---

## Reto 4: ConfigMap + Secret consumidos por la app (10 min)

**Definición**  
Separar configuración del código.

**Qué debes hacer**
- Crear `ConfigMap/app-config` con:
  - `APP_COLOR=blue`
  - `APP_TITLE=CKAD-Reto`
- Crear `Secret/app-secret` con `API_KEY=supersecret`.
- Montar el ConfigMap como variables de entorno en `webapp` (via `envFrom`).
- Montar el Secret como archivo en `/etc/secret/API_KEY` (read-only recomendado).

**Archivos a crear (para score)**
- *(Recomendado)* `manifests/04-config.yaml` (ConfigMap + Secret)
- Editar `manifests/02-webapp-deploy.yaml` para consumir `app-config` y `app-secret`.

**Resultado esperado**
- Pods con env vars del ConfigMap.
- Archivo del Secret montado.

**Validaciones**
```bash
kubectl -n ckad-reto get cm app-config -o yaml | sed -n '1,120p'
```
```bash
kubectl -n ckad-reto get secret app-secret -o yaml | sed -n '1,120p'
```
```bash
POD=$(kubectl -n ckad-reto get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}')
```
```bash
kubectl -n ckad-reto exec "$POD" -- printenv | grep -E "APP_COLOR|APP_TITLE"
```
```bash
kubectl -n ckad-reto exec "$POD" -- sh -c 'cat /etc/secret/API_KEY && echo'
```

---

## Reto 5: Probes (readiness/liveness) y observación de comportamiento (10 min)

**Definición**  
Agregar `readinessProbe` y `livenessProbe` correctas.

**Qué debes hacer**
- Agregar `readinessProbe` HTTP a `/` port 80.
- Agregar `livenessProbe` HTTP a `/` port 80 con tiempos razonables.
- Forzar un fallo controlado (p.ej., path inválido) y observar:
  - readiness: Pod sale de endpoints (NoReady)
  - liveness: contenedor se reinicia

**Resultado esperado**
- Entender y evidenciar diferencia entre readiness vs liveness.

**Validaciones**
```bash
kubectl -n ckad-reto describe pod -l app=webapp | egrep -n "Readiness|Liveness" -A6 -B2
kubectl -n ckad-reto get pods -l app=webapp -w
kubectl -n ckad-reto get events --sort-by=.lastTimestamp | tail -n 30
```

---

## Reto 6: Rolling update + rollback (10 min)

**Definición**  
Actualizar la app y regresar si algo falla.

**Qué debes hacer**
- Cambiar la imagen a una nueva etiqueta (p.ej., `v2`).
- Verificar rollout.
- Simular fallo (imagen inexistente) y ejecutar rollback.

**Resultado esperado**
- Manejo correcto de `rollout status`, `rollout history`, `rollout undo`.

**Validaciones**
```bash
kubectl -n ckad-reto rollout status deploy/webapp
kubectl -n ckad-reto rollout history deploy/webapp
kubectl -n ckad-reto rollout undo deploy/webapp
```

---

## Reto 7: Pod multi-contenedor (sidecar) (12 min)

**Definición**  
Implementar un patrón multi-container.

**Qué debes hacer**
- Crear un Pod (o Deployment pequeño) con:
  - Contenedor principal que escriba logs en `/var/log/app.log` (en un `emptyDir`).
  - Sidecar que haga `tail -F` del archivo y lo imprima a stdout.
- Etiquetar con `app=sidecar-demo`.

**Resultado esperado**
- Logs del sidecar mostrando lo que escribe el contenedor principal.

**Validaciones**
```bash
kubectl -n ckad-reto get pod -l app=sidecar-demo
kubectl -n ckad-reto logs -l app=sidecar-demo -c sidecar --tail=20
```

---

## Reto 8: Ingress (HTTP routing) (10 min)

**Definición**  
Exponer HTTP con reglas.

**Qué debes hacer**
- Crear un Ingress que rote `/` hacia `webapp-svc:80`.
- Host lógico: `webapp.ckad.local`.

**Resultado esperado**
- Ingress creado (y reconciliado por un controller, si existe).

**Validaciones**
```bash
kubectl -n ckad-reto get ingress
kubectl -n ckad-reto describe ingress
```

> Si tu clúster tiene LB/DNS para el Ingress Controller, prueba con `curl -H "Host: webapp.ckad.local" ...` y documenta resultado.
{: .lab-note .info .compact}

---

## Reto 9: NetworkPolicy (default deny + allow) (12 min)

**Definición**  
Restringir tráfico entre Pods por etiquetas.

**Qué debes hacer**
- Crear NetworkPolicy “default deny ingress” para `app=webapp`.
- Crear otra policy que permita ingress solo desde Pods con `role=tester` (TCP 80).

**Resultado esperado**
- Tráfico bloqueado desde Pods sin `role=tester` y permitido desde tester autorizado (si hay enforcement).

**Validaciones**
```bash
kubectl -n ckad-reto get networkpolicy

# Pod no autorizado:
kubectl -n ckad-reto run bad --image=curlimages/curl:8.5.0 -it --rm --restart=Never -- \
  curl -m 2 -sS http://webapp-svc || echo "BLOQUEADO (esperado si hay enforcement)"

# Pod autorizado:
kubectl -n ckad-reto run good --labels=role=tester --image=curlimages/curl:8.5.0 -it --rm --restart=Never -- \
  curl -m 5 -sS http://webapp-svc | head
```

> Si NO hay enforcement de NetworkPolicy en tu clúster, **documenta la limitación** (el YAML sigue siendo válido).
{: .lab-note .warning .compact}

---

## Reto 10: Job y CronJob (batch) (10 min)

**Definición**  
Ejecutar tareas “run-to-completion”.

**Qué debes hacer**
- Crear un `Job/pi` (o similar) que corra y termine exitoso.
- Crear `CronJob/heartbeat` cada 1 minuto que imprima “tick”.

**Resultado esperado**
- Job completado y CronJob programando Jobs.

**Validaciones**
```bash
kubectl -n ckad-reto get job
kubectl -n ckad-reto describe job pi | egrep -n "Succeeded|Completed" -A2 -B2 || true
kubectl -n ckad-reto get cronjob
kubectl -n ckad-reto get jobs --watch
```

---

## Reto 11: PVC y persistencia (10 min)

**Definición**  
Consumir almacenamiento persistente.

**Qué debes hacer**
- Crear `PersistentVolumeClaim/data-pvc` de 1Gi.
- Crear un Pod `pvc-writer` que monte el PVC en `/data` y escriba un archivo.
- Borrar y recrear el pod para demostrar persistencia.

**Resultado esperado**
- PVC `Bound` y archivo persiste entre recreaciones del Pod (si hay StorageClass dinámica).

**Validaciones**
```bash
kubectl -n ckad-reto get pvc data-pvc
kubectl -n ckad-reto describe pvc data-pvc | egrep -n "Status|StorageClass|Bound|Volume" -A1 -B1

kubectl -n ckad-reto exec pvc-writer -- sh -c 'ls -la /data && cat /data/hello.txt'
```

---

## Reto 12: RBAC mínimo + verificación con kubectl auth can-i (12 min)

**Definición**  
Crear permisos mínimos para un ServiceAccount.

**Qué debes hacer**
- Crear `ServiceAccount/app-sa`.
- Crear Role que permita:
  - `get,list,watch` sobre `pods`
  - `get` sobre subrecurso `pods/log`
- Crear RoleBinding para enlazar Role ↔ ServiceAccount.
- Verificar permisos usando `kubectl auth can-i` con impersonation `--as=system:serviceaccount:ckad-reto:app-sa`.

**Resultado esperado**
- “Least privilege” funcionando y comprobado rápidamente.

**Validaciones**
```bash
kubectl -n ckad-reto auth can-i list pods --as=system:serviceaccount:ckad-reto:app-sa
kubectl -n ckad-reto auth can-i delete pods --as=system:serviceaccount:ckad-reto:app-sa
kubectl -n ckad-reto auth can-i get pods --subresource=log --as=system:serviceaccount:ckad-reto:app-sa
```

---

## Checklist final de entrega (guardar evidencia)

Ejecuta y guarda salida en `notes/final-check.txt`:

```bash
{
  echo "===== FINAL CHECK (CKAD RETO) ====="
  echo "Generated: $(date -Iseconds)"
  echo

  echo "== NS =="
  kubectl get ns ckad-reto
  echo

  echo "== WORKLOADS =="
  kubectl -n ckad-reto get deploy,po,svc,ingress
  echo

  echo "== CONFIG =="
  kubectl -n ckad-reto get cm,secret
  echo

  echo "== SECURITY/NET =="
  kubectl -n ckad-reto get networkpolicy,sa,role,rolebinding
  echo

  echo "== BATCH =="
  kubectl -n ckad-reto get job,cronjob
  echo

  echo "== STORAGE =="
  kubectl -n ckad-reto get pvc
  echo
} | tee notes/final-check.txt
```

---

#### Tarea 2 - Script de evaluación

- {% include step_label.html %} Crea el script dentro del directorio **lab21**.

  ```bash
  cat > scripts/score.sh <<'EOF'
  #!/usr/bin/env bash
  set -u

  NS="${NS:-ckad-reto}"

  PASS=0
  TOTAL=100

  say() { printf "%s\n" "$*"; }
  ok()  { say "✅ $*"; }
  bad() { say "❌ $*"; }
  note(){ say "ℹ️  $*"; }

  add() { PASS=$((PASS + $1)); }

  exists_ns() { kubectl get ns "$1" >/dev/null 2>&1; }
  exists() { kubectl -n "$NS" get "$1" "$2" >/dev/null 2>&1; }

  score_item() {
    local pts="$1" name="$2" cmd="$3"
    if eval "$cmd" >/dev/null 2>&1; then
      add "$pts"; ok "[$pts] $name"
    else
      bad "[$pts] $name"
    fi
  }

  say "===== CKAD RETO SCORE ====="
  say "Namespace: $NS"
  say "Generated: $(date -Iseconds)"
  say

  # =========================
  # R1 (4 pts): namespace + contexto
  # =========================
  score_item 2 "R1: Namespace exists" "exists_ns $NS"
  # namespace en contexto (si no está, suele salir vacío)
  CTX_NS="$(kubectl config view --minify -o jsonpath='{..namespace}' 2>/dev/null || true)"
  if [[ "$CTX_NS" == "$NS" ]]; then
    add 2; ok "[2] R1: Context namespace is $NS"
  else
    bad "[2] R1: Context namespace is '$CTX_NS' (expected '$NS')"
  fi

  # =========================
  # R2 (12 pts): Deployment listo con 2 réplicas
  # =========================
  score_item 4 "R2: Deployment webapp exists" "exists deploy webapp"
  if exists deploy webapp; then
    READY="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0)"
    if [[ "${READY:-0}" =~ ^[0-9]+$ ]] && (( READY >= 2 )); then
      add 8; ok "[8] R2: webapp readyReplicas >= 2 (readyReplicas=$READY)"
    else
      bad "[8] R2: webapp readyReplicas >= 2 (readyReplicas=${READY:-0})"
    fi
  else
    bad "[8] R2: webapp readyReplicas (deployment missing)"
  fi

  # =========================
  # R3 (6 pts): Service correcto
  # =========================
  score_item 4 "R3: Service webapp-svc exists" "exists svc webapp-svc"
  if exists svc webapp-svc; then
    PORT="$(kubectl -n "$NS" get svc webapp-svc -o jsonpath='{.spec.ports[0].port}' 2>/dev/null || true)"
    TPORT="$(kubectl -n "$NS" get svc webapp-svc -o jsonpath='{.spec.ports[0].targetPort}' 2>/dev/null || true)"
    # targetPort vacío equivale a port; aceptamos 80/80 o 80/(vacío)
    if [[ "$PORT" == "80" && ( "$TPORT" == "80" || -z "$TPORT" ) ]]; then
      add 2; ok "[2] R3: Service port 80 -> targetPort 80"
    else
      bad "[2] R3: Service port/targetPort expected 80/80 (got ${PORT:-?}/${TPORT:-empty})"
    fi
  else
    bad "[2] R3: Service port check (service missing)"
  fi

  # =========================
  # R4 (10 pts): ConfigMap + Secret consumidos
  # =========================
  score_item 2 "R4: ConfigMap app-config exists" "exists cm app-config"
  score_item 2 "R4: Secret app-secret exists" "exists secret app-secret"

  if exists deploy webapp; then
    # envFrom contiene app-config
    CMREFS="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{range .spec.template.spec.containers[0].envFrom[*]}{.configMapRef.name}{" "}{end}' 2>/dev/null || true)"
    if echo "$CMREFS" | grep -qw "app-config"; then
      add 3; ok "[3] R4: Deployment envFrom includes app-config"
    else
      bad "[3] R4: Deployment envFrom includes app-config"
    fi

    # volumen secretName incluye app-secret
    SECRETS="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{.spec.template.spec.volumes[*].secret.secretName}' 2>/dev/null || true)"
    MOUNTS="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{.spec.template.spec.containers[0].volumeMounts[*].mountPath}' 2>/dev/null || true)"
    if echo "$SECRETS" | grep -qw "app-secret" && echo "$MOUNTS" | grep -qw "/etc/secret"; then
      add 3; ok "[3] R4: app-secret volume + mounted at /etc/secret"
    else
      bad "[3] R4: app-secret volume + mounted at /etc/secret"
    fi
  else
    bad "[3] R4: Deployment checks (deployment missing)"
    bad "[3] R4: Secret mount checks (deployment missing)"
  fi

  # =========================
  # R5 (6 pts): Probes
  # =========================
  if exists deploy webapp; then
    RPATH="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.path}' 2>/dev/null || true)"
    LPATH="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{.spec.template.spec.containers[0].livenessProbe.httpGet.path}' 2>/dev/null || true)"
    RPORT="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.port}' 2>/dev/null || true)"
    LPORT="$(kubectl -n "$NS" get deploy webapp -o jsonpath='{.spec.template.spec.containers[0].livenessProbe.httpGet.port}' 2>/dev/null || true)"

    if [[ "$RPATH" == "/" && "$RPORT" == "80" ]]; then add 3; ok "[3] R5: readinessProbe httpGet /:80"; else bad "[3] R5: readinessProbe httpGet /:80"; fi
    if [[ "$LPATH" == "/" && "$LPORT" == "80" ]]; then add 3; ok "[3] R5: livenessProbe httpGet /:80"; else bad "[3] R5: livenessProbe httpGet /:80"; fi
  else
    bad "[3] R5: readinessProbe (deployment missing)"
    bad "[3] R5: livenessProbe (deployment missing)"
  fi

  # =========================
  # R6 (8 pts): Rollout history >=2
  # =========================
  if exists deploy webapp; then
    REVCOUNT="$(kubectl -n "$NS" rollout history deploy/webapp 2>/dev/null | awk '$1 ~ /^[0-9]+$/ {c++} END{print c+0}')"
    if [[ "${REVCOUNT:-0}" =~ ^[0-9]+$ ]] && (( REVCOUNT >= 2 )); then
      add 8; ok "[8] R6: rollout history has >=2 revisions"
    else
      bad "[8] R6: rollout history has >=2 revisions (found ${REVCOUNT:-0})"
    fi
  else
    bad "[8] R6: rollout history (deployment missing)"
  fi

  # =========================
  # R7 (10 pts): Sidecar
  # =========================
  score_item 4 "R7: sidecar-demo pod exists" "exists pod sidecar-demo"
  if exists pod sidecar-demo; then
    NAMES="$(kubectl -n "$NS" get pod sidecar-demo -o jsonpath='{.spec.containers[*].name}' 2>/dev/null || true)"
    if echo "$NAMES" | grep -qw "app" && echo "$NAMES" | grep -qw "sidecar"; then
      add 3; ok "[3] R7: pod has containers app + sidecar"
    else
      bad "[3] R7: pod has containers app + sidecar"
    fi

    if kubectl -n "$NS" logs sidecar-demo -c sidecar --tail=50 2>/dev/null | grep -qi 'msg'; then
      add 3; ok "[3] R7: sidecar logs contain 'msg'"
    else
      bad "[3] R7: sidecar logs contain 'msg'"
    fi
  else
    bad "[3] R7: container names check (pod missing)"
    bad "[3] R7: sidecar logs check (pod missing)"
  fi

  # =========================
  # R8 (6 pts): Ingress
  # =========================
  score_item 4 "R8: Ingress webapp-ing exists" "exists ingress webapp-ing"
  if exists ingress webapp-ing; then
    HOST="$(kubectl -n "$NS" get ingress webapp-ing -o jsonpath='{.spec.rules[0].host}' 2>/dev/null || true)"
    SVCN="$(kubectl -n "$NS" get ingress webapp-ing -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}' 2>/dev/null || true)"
    SVCP="$(kubectl -n "$NS" get ingress webapp-ing -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.port.number}' 2>/dev/null || true)"
    if [[ "$HOST" == "webapp.ckad.local" && "$SVCN" == "webapp-svc" && "$SVCP" == "80" ]]; then
      add 2; ok "[2] R8: host+backend correct"
    else
      bad "[2] R8: expected webapp.ckad.local -> webapp-svc:80 (got $HOST -> $SVCN:$SVCP)"
    fi
  else
    bad "[2] R8: host+backend check (ingress missing)"
  fi

  # =========================
  # R9 (6 pts): NetPol (YAML existence)
  # =========================
  score_item 3 "R9: NetPol deny exists" "exists networkpolicy webapp-deny-ingress"
  score_item 3 "R9: NetPol allow exists" "exists networkpolicy webapp-allow-from-tester"
  note "R9: enforcement depende del CNI; aquí solo evaluamos existencia/estructura básica."

  # =========================
  # R10 (8 pts): Job/CronJob
  # =========================
  if exists job pi; then
    if kubectl -n "$NS" get job pi -o jsonpath='{.status.succeeded}' 2>/dev/null | grep -qE '^[1-9]'; then
      add 4; ok "[4] R10: Job pi succeeded"
    else
      bad "[4] R10: Job pi succeeded"
    fi
  else
    bad "[4] R10: Job pi exists"
  fi

  score_item 2 "R10: CronJob heartbeat exists" "exists cronjob heartbeat"
  if exists cronjob heartbeat; then
    SCHED="$(kubectl -n "$NS" get cronjob heartbeat -o jsonpath='{.spec.schedule}' 2>/dev/null || true)"
    if [[ "$SCHED" == "*/1 * * * *" ]]; then
      add 2; ok "[2] R10: heartbeat schedule is */1 * * * *"
    else
      bad "[2] R10: heartbeat schedule is */1 * * * * (got '$SCHED')"
    fi
    if kubectl -n "$NS" get jobs 2>/dev/null | awk '{print $1}' | grep -q '^heartbeat-'; then
      add 0; ok "[0] R10: heartbeat has created jobs (info)"
    fi
  else
    bad "[2] R10: heartbeat schedule check (cronjob missing)"
  fi

  # =========================
  # R11 (10 pts): PVC
  # =========================
  if exists pvc data-pvc; then
    PHASE="$(kubectl -n "$NS" get pvc data-pvc -o jsonpath='{.status.phase}' 2>/dev/null || true)"
    if [[ "$PHASE" == "Bound" ]]; then
      add 6; ok "[6] R11: PVC data-pvc is Bound"
    else
      bad "[6] R11: PVC data-pvc is Bound (phase=$PHASE)"
    fi
  else
    bad "[6] R11: PVC data-pvc exists"
  fi

  if exists pod pvc-writer; then
    if kubectl -n "$NS" exec pvc-writer -- sh -c 'test -f /data/hello.txt' >/dev/null 2>&1; then
      add 4; ok "[4] R11: pvc-writer wrote /data/hello.txt"
    else
      bad "[4] R11: pvc-writer wrote /data/hello.txt"
    fi
  else
    bad "[4] R11: pvc-writer pod exists"
  fi

  # =========================
  # R12 (14 pts): RBAC
  # =========================
  score_item 2 "R12: ServiceAccount app-sa exists" "exists sa app-sa"
  score_item 2 "R12: Role app-reader exists" "exists role app-reader"
  score_item 2 "R12: RoleBinding app-reader-binding exists" "exists rolebinding app-reader-binding"

  if kubectl -n "$NS" auth can-i list pods --as="system:serviceaccount:$NS:app-sa" 2>/dev/null | grep -qi '^yes$'; then
    add 3; ok "[3] R12: can list pods"
  else
    bad "[3] R12: can list pods"
  fi

  if kubectl -n "$NS" auth can-i delete pods --as="system:serviceaccount:$NS:app-sa" 2>/dev/null | grep -qi '^no$'; then
    add 3; ok "[3] R12: cannot delete pods (least privilege)"
  else
    bad "[3] R12: cannot delete pods (should be no)"
  fi

  if kubectl -n "$NS" auth can-i get pods --subresource=log --as="system:serviceaccount:$NS:app-sa" 2>/dev/null | grep -qi '^yes$'; then
    add 4; ok "[4] R12: can get pods/log"
  else
    bad "[4] R12: can get pods/log"
  fi

  say
  say "===== SCORE ====="
  say "Total: $PASS / $TOTAL"
  if (( PASS == TOTAL )); then
    say "Result: PERFECT ✅"
  elif (( PASS >= 80 )); then
    say "Result: PASS ✅ (>=80)"
  else
    say "Result: NEEDS WORK ❌ (<80)"
  fi
  EOF

  chmod +x scripts/score.sh
  ```

- {% include step_label.html %} Ejecuta el script con el siguiente comando.

  > **IMPORTANTE:** Recuerda que el script solo funcionara si se realizaron los retos con los nombres correctos.
  {: .lab-note .important .compact}

  ```bash
  NS=ckad-reto ./scripts/score.sh
  ```

---

#### Tarea 3 — Elimina el clúster EKS

Limpieza de la simulación

- {% include step_label.html %} Si ya terminaste elimina el namespace.

  ```bash
  kubectl delete ns ckad-reto
  ```

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **9 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "sim-exam-eks" --region "us-west-2"
  ```

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  ```bash
  aws eks describe-cluster --name "sim-exam-eks" --region "us-west-2" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
