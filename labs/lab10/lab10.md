---
layout: lab
title: "Práctica 10: Implementación de GitOps avanzado con ArgoCD (multi-entorno)"
permalink: /lab10/lab10/
images_base: /labs/lab10/img
duration: "60 minutos"
objective:
  - "Implementar un flujo GitOps multi-entorno (dev/stage/prod) en un clúster EKS usando Argo CD: Kustomize overlays, ApplicationSet para generar aplicaciones por entorno, AppProject para aislar permisos (repos/namespace/recursos), auto-sync + prune + self-heal (solo donde aplica) y promoción controlada por cambios en Git; validando Synced/OutOfSync/Healthy, eventos y diferencias (drift) con troubleshooting estilo CKAD."
prerequisites:
  - "Cuenta AWS con permisos para: EKS, IAM, EC2 y VPC."
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, eksctl, helm v3, git, curl."
  - "Cuenta en Git (GitHub/GitLab) con un repo al que Argo CD pueda leer manifests (recomendado repo público para el lab)."
  - "(Opcional) argocd CLI (acelera validaciones/sync/diff)."
introduction:
  - "En GitOps, el clúster aplica únicamente lo definido en Git. Argo CD compara “deseado vs real”, marca Synced/OutOfSync/Healthy y puede auto-sincronizar; opcionalmente puede hacer prune y self-heal para revertir cambios manuales (drift). En esta práctica lo haremos multi-entorno dentro del mismo clúster (por namespaces), usando ApplicationSet (para generar apps por entorno) y AppProject (para restringir orígenes/destinos/recursos), reforzando troubleshooting tipo CKAD con kubectl describe, eventos y diferencias (drift)."
slug: lab10
lab_number: 10
final_result: >
  Al finalizar tendrás un flujo GitOps multi-entorno profesional: un repo con Kustomize overlays para dev/stage/prod, Argo CD con AppProject como guardrails, ApplicationSet generando Applications por entorno, dev/stage con auto-sync + prune + self-heal y prod con sincronización manual (promoción controlada), más evidencia práctica de estado Synced/OutOfSync/Healthy, rollouts y corrección de drift.
notes:
  - "CKAD: práctica núcleo (Deployments/Services/ConfigMaps, namespaces, labels/selectors, rollout, troubleshooting con kubectl describe/events/logs/rollout, diferencias deseado vs real)."
  - "Recomendación para el lab: usa un repo público. Si usas repo privado, primero configura credenciales de repo en Argo CD (Kubernetes Secret / argocd CLI / tu estándar)."
  - "Regla de oro GitOps: el deploy es el commit; los cambios manuales en el clúster son drift y (si self-heal aplica) serán revertidos."
references:
  - text: "Argo CD - Automated Sync Policy (prune/selfHeal)"
    url: https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
  - text: "Argo CD - Projects (AppProject)"
    url: https://argo-cd.readthedocs.io/en/stable/user-guide/projects/
  - text: "Argo CD - Project Specification (campos: sourceRepos, destinations, whitelists)"
    url: https://argo-cd.readthedocs.io/en/stable/operator-manual/project-specification/
  - text: "Argo CD - ApplicationSet (conceptos)"
    url: https://argo-cd.readthedocs.io/en/latest/user-guide/application-set/
  - text: "Argo CD - ApplicationSet List Generator"
    url: https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-List/
  - text: "Argo CD - Sync Options (CreateNamespace, etc.)"
    url: https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/
  - text: "Argo CD - Sync Waves (argocd.argoproj.io/sync-wave)"
    url: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
  - text: "Kubernetes - Declarative Management Using Kustomize"
    url: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
  - text: "Kubernetes - kubectl kustomize"
    url: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/
  - text: "AWS - Working with Argo CD on Amazon EKS"
    url: https://docs.aws.amazon.com/eks/latest/userguide/working-with-argocd.html
  - text: "AWS - Working with Argo CD Projects"
    url: https://docs.aws.amazon.com/eks/latest/userguide/argocd-projects.html
  - text: "AWS - Use ApplicationSets (EKS)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/argocd-applicationsets.html
prev: /lab9/lab9/
next: /lab11/lab11/
---

## Costo (resumen)

- **EKS** cobra por clúster por hora (y puede haber diferencia entre Standard vs Extended Support). Revisa pricing antes de iniciar.
- **Argo CD** no se cobra como servicio separado, pero consume recursos (CPU/Mem) en tus nodos.
- La app de ejemplo (whoami) es ligera; el costo dominante es el **clúster y nodos**.

> **IMPORTANTE:** Si creas un clúster solo para esta práctica, elimínalo al final para evitar costos.
{: .lab-note .important .compact}

---

### Tarea 1. Preparación del workspace + clúster EKS + Argo CD

Crearás la carpeta de la práctica, validarás conectividad al clúster y confirmarás que Argo CD está sano (pods/servicios/CRDs). **Si NO tienes clúster o Argo CD**, los crearás en esta tarea.

> **NOTA (CKAD):** La rutina base de troubleshooting es: confirmar contexto → describir recursos → leer eventos → revisar logs del controlador.
{: .lab-note .info .compact}

#### Tarea 1.1 — Workspace, herramientas y variables

- {% include step_label.html %} Abre **Visual Studio Code** y la terminal integrada.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Crea el directorio del laboratorio y entra en él.

  ```bash
  mkdir -p lab10 && cd lab10
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura estándar del laboratorio.

  ```bash
  mkdir -p repo k8s/bootstrap outputs logs scripts
  ```

- {% include step_label.html %} Confirma la estructura de carpetas creada.

  ```bash
  find . -maxdepth 3 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica herramientas (si alguna falla, corrige PATH/instalación antes de continuar).

  ```bash
  aws --version
  ```
  ```bash
  kubectl version --client=true
  ```
  ```bash
  eksctl version
  ```
  ```bash
  helm version --short
  ```
  ```bash
  git --version
  ```
  ```bash
  curl --version | head -n 1
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base.

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-argoadv-lab"
  export ARGOCD_NS="argocd"
  ```

- {% include step_label.html %} Guarda variables en `scripts/vars.env` para recargarlas rápido (sin credenciales).

  ```bash
  cat > scripts/vars.env <<EOF
  export AWS_REGION="${AWS_REGION}"
  export CLUSTER_NAME="${CLUSTER_NAME}"
  export ARGOCD_NS="${ARGOCD_NS}"
  EOF
  ```

- {% include step_label.html %} Carga variables desde el archivo (si reabriste terminal).

  ```bash
  source scripts/vars.env
  ```

- {% include step_label.html %} Valida identidad AWS (reduce el riesgo de operar en otra cuenta/región) y guarda evidencia.

  ```bash
  aws sts get-caller-identity | tee outputs/01_aws_identity.json
  ```
  ```bash
  aws configure get region || true
  ```
  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID" | tee outputs/02_aws_account_id.txt
  ```
  {% include step_image.html %}

#### Tarea 1.2 — Crear clúster EKS (si no existe)

> **IMPORTANTE:** Si **ya tienes** un clúster para el curso, **omite** la creación. Solo asegúrate de que `CLUSTER_NAME` apunta al clúster correcto y ejecuta las verificaciones.
{: .lab-note .important .compact}

- {% include step_label.html %} (Recomendado) Lista versiones disponibles en tu región y confirma que tu versión objetivo está disponible.

  > **Tip:** Si tu región no ofrece `1.33`, elige la versión más alta disponible y usa ese valor en `K8S_VERSION`.
  {: .lab-note .info .compact}

  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables del clúster (ajusta si tu listado difiere).

  ```bash
  export K8S_VERSION="1.33"
  export NODEGROUP_NAME="mng-1"
  ```

- {% include step_label.html %} (Opcional) Crea un clúster de laboratorio con `eksctl` (Managed Node Group).

  > **NOTA:** A partir de EKS/Kubernetes `1.33`, AWS ya no publica AMIs optimizadas basadas en Amazon Linux 2 para EKS. Usa AMIs AL2023/Bottlerocket según tu flujo.
  {: .lab-note .info .compact}

  > **NOTA:** `eksctl` crea VPC/subnets/SG/roles y el clúster con un Managed Node Group listo para workloads.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.medium \
    --nodes 2 --nodes-min 2 --nodes-max 3
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el clúster esté `ACTIVE` (evidencia).

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.status" --output text | tee outputs/04_cluster_status.txt
  ```
  {% include step_image.html %}

#### Tarea 1.3 — Conectar kubectl y validar conectividad

- {% include step_label.html %} Apunta `kubectl` al clúster.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  kubectl config current-context | tee outputs/05_kubectl_context.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica API del clúster y nodos.

  ```bash
  kubectl cluster-info | tee outputs/06_cluster_info.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get nodes -o wide | tee outputs/07_nodes.txt
  ```
  {% include step_image.html %}

#### Tarea 1.4 — Instalar Argo CD (si no existe) y verificar

> Si Argo CD ya existe en tu clúster, ejecuta igual los comandos de verificación. Si no existe, instálalo con Helm.
{: .lab-note .info .compact}

- {% include step_label.html %} Agrega el repo Helm de Argo y actualiza índices.

  ```bash
  helm repo add argo https://argoproj.github.io/argo-helm
  helm repo update
  ```
  {% include step_image.html %}

- {% include step_label.html %} Instala/actualiza Argo CD en el namespace `argocd`.

  ```bash
  helm upgrade --install argocd argo/argo-cd \
    --namespace "$ARGOCD_NS" --create-namespace \
    --wait
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica Argo CD (namespace/pods/servicios) y guarda evidencias.

  ```bash
  kubectl get ns "$ARGOCD_NS" | tee outputs/08_argocd_ns.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get pods -o wide | tee outputs/09_argocd_pods.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get svc -o wide | tee outputs/10_argocd_svc.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica CRDs clave (Application / AppProject / ApplicationSet).

  > **IMPORTANTE:** Si **no** aparece `applicationsets.argoproj.io`, tu instalación de Argo CD no incluye ApplicationSet controller. Instálalo/actívalo antes de continuar (según tu método Helm/manifiestos).
  {: .lab-note .important .compact}

  ```bash
  kubectl get crd | egrep "applications.argoproj.io|appprojects.argoproj.io|applicationsets.argoproj.io" \
    | tee outputs/11_argocd_crds.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén la contraseña inicial del usuario `admin` (si existe) y guárdala como evidencia.

  > **Nota:** En muchas instalaciones, la contraseña inicial está en el Secret `argocd-initial-admin-secret`. Si no existe, tu chart/valores pudieron deshabilitarla o ya fue rotada. **Guarda tu contraseña en un bloc de notas**
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$ARGOCD_NS" get secret argocd-initial-admin-secret >/dev/null 2>&1 \
    && kubectl -n "$ARGOCD_NS" get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' \
      | base64 -d | tee outputs/12_argocd_admin_password.txt \
    || echo "WARN: No existe argocd-initial-admin-secret (usa tu método de auth corporativo/valores Helm)" | tee outputs/12_argocd_admin_password.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Abre una **segunda terminal** para activar la UI con port-forward para observar Sync/Health/Diff

  ```bash
  export ARGOCD_NS="argocd"
  kubectl -n "$ARGOCD_NS" port-forward svc/argocd-server 8080:443
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre el navegador **(Chrome/Edge)** en el equipo de trabajo y escribe la siguiente URL

  > **IMPORTANTE:** Acepta la advertencia de que la conexión no es privada.
  {: .lab-note .important .compact}

  ```bash
  https://localhost:8080
  ```

- {% include step_label.html %} Puedes acceder usando el usuario **`admin`** y la contraseña que guardaste en el bloc de notas.

  {% include step_image.html %}

- {% include step_label.html %} Regresa a la **primera terminal** para capturar los eventos recientes de Argo CD (útil para troubleshooting).

  ```bash
  kubectl -n "$ARGOCD_NS" get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/13_argocd_events_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear repo GitOps con Kustomize multi-entorno

Crearás una app simple (Deployment + Service) y la parametrizarás con Kustomize overlays para dev/stage/prod. Cada entorno tendrá su propio namespace, réplicas e “identidad” (ConfigMap).

> **NOTA (CKAD):** Aunque Kustomize no es “tema directo” del examen, **los recursos sí**: Deployment/Service/ConfigMap + namespaces + labels/selectors + rollouts.
{: .lab-note .info .compact}

#### Tarea 2.1 — Inicializar repo local (y remoto)

- {% include step_label.html %} Accede a tu cuenta de **GitHub**, sino tienes cuenta crea una nueva y crea un repositorio **publico**.

  - Da clic [AQUÍ](https://github.com/login) para abrir GitHub.
  - La sugerencia del nombre del repositorio puede ser: `gitops-adv-argocd-##xx`
  - Sustituye los **#** por numeros y las **x** por letras aleatorias.
  - Continua al siguiente paso una vez creado tu repositorio.
  - **Copia la URL HTTPS del repositorio creado y guardala en un bloc de notas.**

  {% include step_image.html %}

- {% include step_label.html %} Entra a la carpeta del repo y crea el repositorio local.

  > **IMPORTANTE:** Antes de ejecutar los comandos ajusta tu **tu_nombre** y **tu_correo**
  {: .lab-note .important .compact}

  ```bash
  cd repo
  git config --global user.name "tu_nombre"
  git config --global user.email "tu_correo"
  git config --global core.autocrlf input
  ```
  {% include step_image.html %}
  ```bash
  export REPO_URL="REPO_URL_AQUI"
  ```
  ```bash
  git clone "$REPO_URL"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Entra a **tu** repositorio y valida la fuente remota.

  > **NOTA:** No se coloca el comando **cd** ya que el nombre de tu repositorio puede ser diferente.
  {: .lab-note .info .compact}

  ```bash
  git remote -v | tee ../../outputs/14_git_remote.txt
  ```
  {% include step_image.html %}

#### Tarea 2.2 — Estructura de directorios

- {% include step_label.html %} Crea estructura Kustomize (base + overlays) y la carpeta `argocd/` (bootstrap GitOps).

  ```bash
  mkdir -p apps/whoami/base
  mkdir -p apps/whoami/overlays/{dev,stage,prod}
  mkdir -p argocd
  ```
  ```bash
  find . -maxdepth 4 -type d | sort | tee ../../outputs/15_repo_tree.txt
  ```
  {% include step_image.html %}

#### Tarea 2.3 — Manifiestos base (Deployment/Service)

- {% include step_label.html %} Crea `apps/whoami/base/deployment.yaml`.

  > **NOTA:** `envFrom` vía ConfigMap marca el entorno (auditoría/troubleshooting). La anotación `sync-wave` ayuda a ordenar recursos cuando agregas dependencias (ej. ConfigMap antes de Deployment).
  {: .lab-note .info .compact}

  ```bash
  cat > apps/whoami/base/deployment.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: whoami
    labels:
      app: whoami
    annotations:
      argocd.argoproj.io/sync-wave: "0"
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: whoami
    template:
      metadata:
        labels:
          app: whoami
      spec:
        containers:
        - name: whoami
          image: traefik/whoami:v1.10.3
          ports:
          - containerPort: 80
          envFrom:
          - configMapRef:
              name: whoami-env
  EOF
  ```

- {% include step_label.html %} Crea `apps/whoami/base/service.yaml`.

  ```bash
  cat > apps/whoami/base/service.yaml <<'EOF'
  apiVersion: v1
  kind: Service
  metadata:
    name: whoami
    labels:
      app: whoami
  spec:
    selector:
      app: whoami
    ports:
    - name: http
      port: 80
      targetPort: 80
  EOF
  ```

- {% include step_label.html %} Crea `apps/whoami/base/kustomization.yaml`.

  ```bash
  cat > apps/whoami/base/kustomization.yaml <<'EOF'
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - deployment.yaml
    - service.yaml
  EOF
  ```

#### Tarea 2.4 — Overlays por entorno

- {% include step_label.html %} Crea overlay **DEV** (namespace + réplicas + ConfigMap).

  ```bash
  cat > apps/whoami/overlays/dev/kustomization.yaml <<'EOF'
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: whoami-dev
  resources:
    - ../../base
  replicas:
    - name: whoami
      count: 1
  images:
    - name: traefik/whoami
      newTag: v1.10.3
  configMapGenerator:
    - name: whoami-env
      literals:
        - ENV=dev
  generatorOptions:
    disableNameSuffixHash: true
    annotations:
      argocd.argoproj.io/sync-wave: "-1"
  EOF
  ```

- {% include step_label.html %} Crea overlay **STAGE**.

  ```bash
  cat > apps/whoami/overlays/stage/kustomization.yaml <<'EOF'
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: whoami-stage
  resources:
    - ../../base
  replicas:
    - name: whoami
      count: 2
  images:
    - name: traefik/whoami
      newTag: v1.10.3
  configMapGenerator:
    - name: whoami-env
      literals:
        - ENV=stage
  generatorOptions:
    disableNameSuffixHash: true
    annotations:
      argocd.argoproj.io/sync-wave: "-1"
  EOF
  ```

- {% include step_label.html %} Crea overlay **PROD**.

  ```bash
  cat > apps/whoami/overlays/prod/kustomization.yaml <<'EOF'
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  namespace: whoami-prod
  resources:
    - ../../base
  replicas:
    - name: whoami
      count: 3
  images:
    - name: traefik/whoami
      newTag: v1.10.3
  configMapGenerator:
    - name: whoami-env
      literals:
        - ENV=prod
  generatorOptions:
    disableNameSuffixHash: true
    annotations:
      argocd.argoproj.io/sync-wave: "-1"
  EOF
  ```

- {% include step_label.html %} Render local (sin aplicar) para comprobar YAML final y guardar evidencia.

  > **NOTA:** Los siguientes comandos muestran los documentos creados previamente, no debe generar errores.
  {: .lab-note .info .compact}

  ```bash
  kubectl kustomize apps/whoami/overlays/dev   | head -n 60 | tee ../../outputs/16_render_dev_head.txt
  ```
  ```bash
  kubectl kustomize apps/whoami/overlays/stage | head -n 60 | tee ../../outputs/17_render_stage_head.txt
  ```
  ```bash
  kubectl kustomize apps/whoami/overlays/prod  | head -n 60 | tee ../../outputs/18_render_prod_head.txt
  ```

- {% include step_label.html %} Commit inicial y push.

  ```bash
  git add .
  ```
  ```bash
  git commit -m "GitOps base: whoami con overlays dev/stage/prod"
  ```
  {% include step_image.html %}
  ```bash
  git push -u origin
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear AppProject y Bootstrap GitOps

Crearás un AppProject para restringir **repo**, **namespaces destino** y **tipos de recursos**. Luego crearás un “root app” (bootstrapping) que aplica la configuración Argo CD desde Git (carpeta `argocd/`).

> **IMPORTANTE:** Evita el anti-patrón de usar siempre el project `default` (demasiado permisivo) en escenarios reales.
{: .lab-note .important .compact}

#### Tarea 3.1 — AppProject “whoami-platform”

- {% include step_label.html %} Crea `argocd/project.yaml` en tu repo (REPO_URL debe ser exacto).

  > **Nota pro:** Para usar `CreateNamespace=true` en el ApplicationSet, el project debe permitir `Namespace` como recurso cluster-scoped.
  {: .lab-note .info .compact}

  ```bash
  cat > argocd/project.yaml <<EOF
  apiVersion: argoproj.io/v1alpha1
  kind: AppProject
  metadata:
    name: whoami-platform
    namespace: ${ARGOCD_NS}
  spec:
    description: "Proyecto GitOps multi-entorno para whoami"
    sourceRepos:
    - ${REPO_URL}

    destinations:
    - server: https://kubernetes.default.svc
      namespace: whoami-dev
    - server: https://kubernetes.default.svc
      namespace: whoami-stage
    - server: https://kubernetes.default.svc
      namespace: whoami-prod

    # Permitimos Namespace (necesario para CreateNamespace=true)
    clusterResourceWhitelist:
    - group: ""
      kind: Namespace

    # Permitimos SOLO recursos namespaced típicos de app
    namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
  EOF
  ```

- {% include step_label.html %} Commit y push del Project.

  ```bash
  git add argocd/project.yaml
  git commit -m "ArgoCD Project: restricciones repo/destinos/recursos"
  git push
  ```
  {% include step_image.html %}

#### Tarea 3.2 — Bootstrap Application (root)

- {% include step_label.html %} Crea el “root app” (se aplica 1 vez desde tu estación de trabajo).

  ```bash
  cat > ../../k8s/bootstrap/00-root-app.yaml <<EOF
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: whoami-bootstrap
    namespace: ${ARGOCD_NS}
  spec:
    # Root app en default (solo para arrancar). La seguridad real vive en whoami-platform.
    project: default
    source:
      repoURL: ${REPO_URL}
      targetRevision: main
      path: argocd
    destination:
      server: https://kubernetes.default.svc
      namespace: ${ARGOCD_NS}
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
      - ApplyOutOfSyncOnly=true
  EOF
  ```

- {% include step_label.html %} Aplica el root app y guarda evidencia.

  ```bash
  kubectl apply -f ../../k8s/bootstrap/00-root-app.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-bootstrap -o wide | tee ../../outputs/19_root_app_get.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Describe el root app (diagnóstico) y guarda evidencia.

  ```bash
  kubectl -n "$ARGOCD_NS" describe app whoami-bootstrap | sed -n '1,200p' | tee ../../outputs/20_root_app_describe_head.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}


---

### Tarea 4. ApplicationSet multi-entorno (dev/stage/prod) + políticas avanzadas

Crearás un ApplicationSet con List generator para generar 3 Applications: `whoami-dev`, `whoami-stage`, `whoami-prod`. Dev/Stage tendrán auto-sync con prune + self-heal. Prod quedará en sync manual.

> **NOTA (CKAD):** `CreateNamespace=true` te obliga a entender “namespace destino” y a diagnosticar fallos de permisos/namespace inexistente.
{: .lab-note .info .compact}

#### Tarea 4.1 — ApplicationSet

- {% include step_label.html %} Crea `argocd/applicationset.yaml` (con goTemplate para autosync condicional).

  {% raw %}
  ```bash
  cat > argocd/applicationset.yaml <<EOF
  apiVersion: argoproj.io/v1alpha1
  kind: ApplicationSet
  metadata:
    name: whoami-multi-env
    namespace: $ARGOCD_NS
  spec:
    goTemplate: true
    goTemplateOptions: ["missingkey=error"]
    generators:
    - list:
        elements:
        - env: dev
          ns: whoami-dev
          path: apps/whoami/overlays/dev
          autoSync: "true"
        - env: stage
          ns: whoami-stage
          path: apps/whoami/overlays/stage
          autoSync: "true"
        - env: prod
          ns: whoami-prod
          path: apps/whoami/overlays/prod
          autoSync: "false"

    template:
      metadata:
        name: 'whoami-{{ .env }}'
        labels:
          app.kubernetes.io/part-of: whoami
          env: '{{ .env }}'
      spec:
        project: whoami-platform
        source:
          repoURL: $REPO_URL
          targetRevision: main
          path: '{{ .path }}'
        destination:
          server: https://kubernetes.default.svc
          namespace: '{{ .ns }}'
        syncPolicy:
          syncOptions:
          - CreateNamespace=true
    templatePatch: |
      {{- if .autoSync }}
      spec:
        syncPolicy:
          automated:
            prune: true
            selfHeal: true
      {{- end }}
  EOF
  ```
  {% endraw %}

- {% include step_label.html %} Sustituye las variables del archivo con los siguientes comandos.

  ```bash
  sed -i "s|\${REPO_URL}|${REPO_URL}|g" argocd/applicationset.yaml
  sed -i "s|\${ARGOCD_NS}|${ARGOCD_NS}|g" argocd/applicationset.yaml
  ```

- {% include step_label.html %} Commit y push del ApplicationSet.

  ```bash
  git add argocd/applicationset.yaml
  git commit -m "ApplicationSet: whoami multi-entorno dev/stage/prod"
  git push
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que Argo CD creó el ApplicationSet y generó 3 Applications (evidencia).

  ```bash
  kubectl -n "$ARGOCD_NS" get applicationset whoami-multi-env -o wide | tee ../../outputs/21_appset_get.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$ARGOCD_NS" get app | egrep "whoami-dev|whoami-stage|whoami-prod" | tee ../../outputs/22_apps_list.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida namespaces y recursos aplicados (CreateNamespace=true).

  ```bash
  kubectl get ns | egrep "whoami-dev|whoami-stage|whoami-prod" | tee ../../outputs/23_env_namespaces.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n whoami-stage get deploy,svc,cm -o wide | tee ../../outputs/24_stage_resources.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura estado Sync/Health (evidencia).

  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-dev   -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/25_dev_sync_health.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-stage -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/26_stage_sync_health.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-prod  -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/27_prod_sync_health.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Promoción (dev → stage → prod) + drift/self-heal + controles

Simularás un cambio (nuevo tag de imagen) primero en dev y observarás auto-sync. Luego promoverás el mismo cambio a stage (auto-sync) y finalmente a prod (manual). También provocarás drift manual (cambio en vivo) y verás cómo self-heal lo revierte en stage.

> **IMPORTANTE:** Verifica que el tag exista. Si `v1.10.4` no existe, usa otro tag válido (o cambia a `latest` solo para laboratorio).
{: .lab-note .important .compact}

#### Tarea 5.1 — Cambio en DEV (auto-sync)

- {% include step_label.html %} Cambia el tag en overlay DEV, commit y push.

  ```bash
  sed -i 's/newTag: v1.10.3/newTag: v1.10.4/' apps/whoami/overlays/dev/kustomization.yaml
  ```
  ```bash
  git add apps/whoami/overlays/dev/kustomization.yaml
  git commit -m "Promote: dev -> whoami v1.10.4"
  git push
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica rollout en `whoami-dev` (evidencia de auto-sync).

  ```bash
  kubectl -n whoami-dev rollout status deploy/whoami | tee ../../outputs/28_dev_rollout.txt
  ```
  ```bash
  kubectl -n whoami-dev get pods -o wide | tee ../../outputs/29_dev_pods.txt
  ```
  {% include step_image.html %}

#### Tarea 5.2 — Promoción a STAGE (auto-sync)

- {% include step_label.html %} Aplica el mismo cambio en STAGE, commit y push.

  ```bash
  sed -i 's/newTag: v1.10.3/newTag: v1.10.4/' apps/whoami/overlays/stage/kustomization.yaml
  ```
  ```bash
  git add apps/whoami/overlays/stage/kustomization.yaml
  git commit -m "Promote: stage -> whoami v1.10.4"
  git push
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica réplicas (debe ser 2) y rollout.

  ```bash
  kubectl -n whoami-stage rollout status deploy/whoami | tee ../../outputs/30_stage_rollout.txt
  ```
  ```bash
  kubectl -n whoami-stage get deploy whoami -o jsonpath='{.spec.replicas}{"\n"}' | tee ../../outputs/31_stage_replicas.txt
  ```
  {% include step_image.html %}

#### Tarea 5.3 — Promoción a PROD (manual)

- {% include step_label.html %} Cambia PROD en Git, pero **no** debe desplegarse automáticamente (sync manual).

  ```bash
  sed -i 's/newTag: v1.10.3/newTag: v1.10.4/' apps/whoami/overlays/prod/kustomization.yaml
  ```
  ```bash
  git add apps/whoami/overlays/prod/kustomization.yaml
  git commit -m "Promote: prod -> whoami v1.10.4 (manual sync)"
  git push
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa `whoami-prod` y ejecuta Sync **manual** desde la UI. Si es neceario, si aparece **Synced/HEalthy** puedes continuar.

  > **Evidencia rápida por kubectl:** mientras no sincronices, es normal ver `OutOfSync`.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-prod -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/32_prod_before_sync.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Después del Sync manual (UI), valida rollout y réplicas en PROD (debe ser 3).

  ```bash
  kubectl -n whoami-prod rollout status deploy/whoami | tee ../../outputs/33_prod_rollout.txt
  ```
  ```bash
  kubectl -n whoami-prod get deploy whoami -o jsonpath='{.spec.replicas}{"\n"}' | tee ../../outputs/34_prod_replicas.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-prod -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/35_prod_after_sync.txt
  ```
  {% include step_image.html %}

#### Tarea 5.4 — Drift y self-heal (stage)

- {% include step_label.html %} Genera drift manual (cambia replicas en vivo a 5).

  > **IMPORTANTE:** Es normal que no veas el incremento de replicas ya que argo lo revierte inmediatamente.
  {: .lab-note .important .compact}

  ```bash
  kubectl -n whoami-stage scale deploy/whoami --replicas=5
  kubectl -n whoami-stage get deploy whoami -o jsonpath='{.spec.replicas}{"\n"}' | tee ../../outputs/36_stage_drift_replicas_now.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera y verifica que Argo revierta a 2 (self-heal).

  ```bash
  sleep 30
  kubectl -n whoami-stage get deploy whoami -o jsonpath='{.spec.replicas}{"\n"}' | tee ../../outputs/37_stage_selfheal_replicas.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura evidencia en Argo (describe + eventos).

  ```bash
  kubectl -n "$ARGOCD_NS" describe app whoami-stage | egrep -n "Sync Status|Health Status|Conditions|Operation|Message" -A2 -B2 | tee ../../outputs/38_argocd_stage_status_snippets.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$ARGOCD_NS" get events --sort-by=.lastTimestamp | tail -n 30 | tee ../../outputs/39_argocd_events_tail_2.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Checklist final

Harás un checklist final tipo CKAD (estado de apps, recursos, eventos).

#### Tarea 6.1

- {% include step_label.html %} Estado final de Argo CD (apps, sync/health).

  ```bash
  kubectl -n "$ARGOCD_NS" get app | egrep "whoami-bootstrap|whoami-dev|whoami-stage|whoami-prod" | tee ../../outputs/40_final_apps_list.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-dev   -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/41_final_dev_sync_health.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-stage -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/42_final_stage_sync_health.txt
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get app whoami-prod  -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}' | tee ../../outputs/43_final_prod_sync_health.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Estado final en Kubernetes (resources por entorno).

  ```bash
  kubectl -n whoami-dev   get deploy,svc,cm -o wide | tee ../../outputs/44_final_dev_resources.txt
  ```
  ```bash
  kubectl -n whoami-stage get deploy,svc,cm -o wide | tee ../../outputs/45_final_stage_resources.txt
  ```
  ```bash
  kubectl -n whoami-prod  get deploy,svc,cm -o wide | tee ../../outputs/46_final_prod_resources.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Eventos recientes (si algo quedó OutOfSync/Degraded, aquí suele estar la pista).

  ```bash
  kubectl -n "$ARGOCD_NS" get events --sort-by=.lastTimestamp | tail -n 30 | tee ../../outputs/47_final_argocd_events_tail.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza de recursos

Realiza la limipeza de los recursos creados por el laboratorio

#### Tarea 7.2 

> **IMPORTANTE:** En GitOps “pro”, lo ideal es limpiar **eliminando manifests en Git** y dejando que Argo haga prune. Para el lab, haremos una limpieza directa para dejar el clúster sin residuos.
{: .lab-note .important .compact}

- {% include step_label.html %} Borra Applications (para que Argo haga cascade delete de recursos).

  ```bash
  kubectl -n "$ARGOCD_NS" delete app whoami-dev whoami-stage whoami-prod --ignore-not-found
  kubectl -n "$ARGOCD_NS" delete app whoami-bootstrap --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Borra ApplicationSet y Project.

  ```bash
  kubectl -n "$ARGOCD_NS" delete applicationset whoami-multi-env --ignore-not-found
  kubectl -n "$ARGOCD_NS" delete appproject whoami-platform --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Borra namespaces destino (solo si ya no te sirven).

  ```bash
  kubectl delete ns whoami-dev whoami-stage whoami-prod --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que no queden residuos.

  > **NOTA:** Las apps pueden tardar un momento en eliminarse. Puedes intentar eliminarlas desde la UI o simplement avanzar y eliminar el cluster de Amazon EKS.
  {: .lab-note .info .compact}

  ```bash
  kubectl get ns | egrep "whoami-dev|whoami-stage|whoami-prod" || echo "OK: namespaces eliminados"
  ```
  ```bash
  kubectl -n "$ARGOCD_NS" get app | egrep "whoami-" || echo "OK: apps eliminadas"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si creaste el clúster solo para esta práctica, elimínalo con eksctl.

  > **NOTA:** El cluster tardara **9 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica (si lo eliminaste) que ya no existe el clúster.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
