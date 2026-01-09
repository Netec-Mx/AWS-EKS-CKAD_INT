---
layout: lab
title: "Práctica 4: Configuración de TLS en un ALB de EKS"
permalink: /lab4/lab4/
images_base: /labs/lab4/img
duration: "55 minutos"
objective:
  - "Configurar terminación TLS (HTTPS) en un Application Load Balancer (ALB) creado desde Amazon EKS mediante AWS Load Balancer Controller, usando un certificado de AWS Certificate Manager (ACM), habilitando redirect HTTP/HTTPS y aplicando una política TLS recomendada; finalmente validar desde Kubernetes (Ingress/Service/Pods) y desde AWS (listeners/SSL policy/certificado)."
prerequisites:
  - "Amazon EKS accesible con kubectl"
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, eksctl, helm, openssl (incluido típicamente en Git Bash)."
  - "Permisos AWS: IAM (IRSA), ACM (import/describe), ELBv2 (ALB/listeners/target groups)."
introduction:
  - "En EKS, un ALB se provisiona declarativamente con un objeto Ingress. El AWS Load Balancer Controller traduce ese Ingress (y sus annotations) en recursos ELBv2 (ALB, listeners, target groups). Para HTTPS, el ALB requiere un certificado de ACM y configuraciones explícitas para listener 443, redirect 80 a 443 y una política TLS. En esta práctica lo harás end-to-end y validarás en Kubernetes y en AWS, usando un certificado autofirmado importado a ACM (sin necesidad de dominio propio)."
slug: lab4
lab_number: 4
final_result: >
  Al finalizar tendrás una aplicación en EKS publicada por un ALB con HTTPS (certificado en ACM importado/autofirmado), redirect automático de HTTP a HTTPS y una SSL Policy moderna aplicada. Además, habrás validado la configuración tanto desde Kubernetes (Ingress/Service/Pods/Events) como desde AWS (listeners, certificado adjunto y política TLS).
notes:
  - "Este laboratorio asume que puedes crear recursos en AWS. Evita ejecutar esto en producción sin controles (WAF, restricciones por CIDR, observabilidad y cambios aprobados)."
  - "Para evitar depender de un dominio (Route 53 / DNS público), este lab usa un certificado **autofirmado** importado a ACM. Los navegadores mostrarán advertencia; para validar de forma limpia usa `curl --cacert outputs/alb.crt https://<ALB_DNS>` o `curl -k` (solo laboratorio)."
  - "Los certificados importados en ACM **no** se renuevan automáticamente. En ambientes reales, usa certificados públicos de ACM con DNS validation y/o automatiza el ciclo de vida."
  - "El ALB tiene costo por hora + LCU. En laboratorio, elimina el Ingress/ALB al finalizar para evitar cargos."
  - "CKAD: aquí practicas objetos clave (Namespace/Deployment/Service/Ingress), troubleshooting con describe/events y diseño declarativo (controller reconcilia)."
references:
  - text: "AWS EKS User Guide - Install AWS Load Balancer Controller with Helm"
    url: https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html
  - text: "AWS EKS User Guide - Install AWS Load Balancer Controller with manifests (incluye IAM policy por versión)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html
  - text: "AWS Load Balancer Controller - Ingress annotations (TLS/ssl-redirect/ssl-policy/listen-ports)"
    url: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
  - text: "AWS Load Balancer Controller - SSL Redirect task"
    url: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/tasks/ssl_redirect/
  - text: "AWS Certificate Manager - Import a certificate (CLI/API)"
    url: https://docs.aws.amazon.com/acm/latest/userguide/import-certificate-api-cli.html
  - text: "AWS CLI - acm import-certificate"
    url: https://docs.aws.amazon.com/cli/latest/reference/acm/import-certificate.html
  - text: "ELB - Security policies for your Application Load Balancer (SSL/TLS policies)"
    url: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html
  - text: "Elastic Load Balancing - Application Load Balancer pricing"
    url: https://aws.amazon.com/elasticloadbalancing/pricing/
prev: /lab3/lab3/
next: /lab5/lab5/
---

---

### Tarea 1. Preparar la carpeta de la práctica y verificar contexto

Crearás la carpeta `lab04`, abrirás la terminal GitBash en VS Code, definirás variables reutilizables (región/cluster) y verificarás que `aws` y `kubectl` apuntan al lugar correcto antes de tocar TLS.

> **IMPORTANTE:** Para simplificar y no depender de un dominio propio, este laboratorio usará el hostname del ALB (`<algo>.<región>.elb.amazonaws.com`) y un certificado **autofirmado** importado a ACM.
{: .lab-note .important .compact}

> **NOTA (CKAD):** La verificación de contexto (`kubectl config current-context`, `kubectl get nodes`, `kubectl describe`, `kubectl get events`) es una habilidad constante bajo presión.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo asignado al curso con el usuario que tenga permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code**.

- {% include step_label.html %} En VS Code, abre la terminal (ícono de terminal en la parte superior derecha).

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la raíz de tu repositorio/labs (ej. `labs-eks-ckad`).

  > **Nota:** Si vienes de otro lab, usa `cd ..` hasta llegar a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio del lab y su estructura.

  ```bash
  mkdir lab04 && cd lab04
  mkdir -p manifests scripts outputs notes
  ```

- {% include step_label.html %} Confirma la estructura de carpetas creada.

  > **Nota.** Dentro de `lab04` deben existir `manifests/ scripts/ outputs/ notes/`.
  {: .lab-note .info .compact}

  ```bash
  pwd
  find . -maxdepth 2 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define las variables del laboratorio.

  ```bash
  export AWS_REGION="us-west-2"
  export NAMESPACE="tls-lab"
  export APP_NAME="whoami"
  ```

- {% include step_label.html %} Confirma que las variables quedaron cargadas.

  > **Nota.** Verifica que tu shell tiene los valores esperados para evitar errores por variables vacías.
  {: .lab-note .info .compact}

  ```bash
  echo "AWS_REGION=$AWS_REGION"
  echo "NAMESPACE=$NAMESPACE"
  echo "APP_NAME=$APP_NAME"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que `openssl` esté disponible (lo usaremos para generar el certificado).

  > **Nota.** La version puede variar pero no afectara la realización de la practica.
  {: .lab-note .info .compact}

  ```bash
  openssl version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que `openssl` responde.

  > **Nota.** Esto permite saber que podrás generar el certificado y la llave sin instalar herramientas adicionales. Es normal que aparezca el mismo resultado de la imagen del paso anterior.
  {: .lab-note .info .compact}

  ```bash
  openssl version | sed -n '1,1p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica la región de AWS a trabajar.

  > **Nota.** La región debe ser **Oregón (us-west-2)**
  {: .lab-note .info .compact}

  ```bash
  aws configure get region || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que tienes credenciales y cuenta correctas.

  > **Nota.** que AWS CLI tiene sesión y permisos (si falla aquí, todo lo demás fallará).
  {: .lab-note .info .compact}

  ```bash
  aws sts get-caller-identity --query "{Account:Account,Arn:Arn}" --output table
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. (Opcional) Crear un clúster EKS para esta práctica

Crearás un clúster EKS con Managed Node Group usando `eksctl`. **Omite esta tarea** si ya tienes clúster y `kubectl` conecta correctamente.

> **IMPORTANTE:** Crear un clúster puede tardar más de lo planeado y generar costos. Úsalo solo si no cuentas con uno de laboratorio.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Recomendado) Revisa versiones soportadas en tu región antes de elegir `K8S_VERSION`.

  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de que la versión elegida aparece en la lista, usaremos la version **`1.33`**.

- {% include step_label.html %} Define las variables para la creación del clúster 

  ```bash
  export CLUSTER_NAME="eks-tls-alb-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Confirma las variables del clúster.

  > **Nota.** Verifica que `eksctl` usará el nombre/versión correctos.
  {: .lab-note .info .compact}

  ```bash
  echo "CLUSTER_NAME=$CLUSTER_NAME"
  echo "NODEGROUP_NAME=$NODEGROUP_NAME"
  echo "K8S_VERSION=$K8S_VERSION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el clúster con un Managed Node Group.

  > **NOTA:** `eksctl` creará VPC/subnets/roles/SG y el clúster listo para workloads.
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

  > **Nota.** El comando verifica que el control plane está listo.
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista los nodegroups creados.

  > **Nota.** El comando verifica que tienes la capacidad de cómputo asociada.
  {: .lab-note .info .compact}

  ```bash
  aws eks list-nodegroups --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura kubeconfig y valida la conectividad al cluster.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  ```bash
  kubectl cluster-info
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que los Nodos muestren `Ready`.

  > **Nota.** El data plane está listo para desplegar la app.
  {: .lab-note .info .compact}

  ```bash
  kubectl get nodes
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

### Tarea 3. Instalar/validar AWS Load Balancer Controller (IRSA + Helm)

Instalarás (o confirmarás) el AWS Load Balancer Controller, que es el componente que **crea el ALB y configura listeners/certificados** a partir de tu Ingress.

> **NOTA (CKAD):** Es el patrón “declaras YAML → controller reconcilia”. Cuando algo falla, el diagnóstico vive en `kubectl describe ingress`, `kubectl get events` y logs del controller.
{: .lab-note .info .compact}

#### Tarea 3.1

- {% include step_label.html %} Verifica si ya existe el deployment del controller. Si existe, confirma que está disponible.

  > **Nota.** El controller está listo para reconciliar Ingress a ALB.
  - Si existe y está `AVAILABLE`, **salta** a “Tarea 4”.
  - Si **no existe**, continúa con instalación.
  {: .lab-note .info .compact}

  ```bash
  kubectl get deployment -n kube-system aws-load-balancer-controller -o wide
  ```
  {% include step_image.html %}

#### Tarea 3.2 (instalación con IRSA)

- {% include step_label.html %} Asegura que tu clúster tiene OIDC provider

  ```bash
  eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --approve
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el OIDC quedó asociado.

  > **Nota.** Es un requisito base para IRSA (si falta, el SA no podrá asumir rol).
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.identity.oidc.issuer" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Descarga el IAM policy oficial del controller.

  > **IMPORTANTE:** El comando se ejecura dentro del directorio **lab04**
  {: .lab-note .important .compact}

  ```bash
  curl -o iam_policy.json \
    https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el archivo de policy existe.

  > **Nota.** El comando verifica que `curl` descargó correctamente el JSON. Solo debe mostrar las primera 10 lineas del archivo.
  {: .lab-note .info .compact}

  ```bash
  ls -la iam_policy.json
  head -n 10 iam_policy.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la policy en IAM (si ya existe, el comando fallará; en ese caso, reutiliza el ARN existente).

  ```bash
  aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que la policy existe (si el create falló por **“Already exists”**, usa esta validación para ubicarla).

  > **Nota.** Identifica que tienes un ARN utilizable para IRSA.
  {: .lab-note .info .compact}

  ```bash
  aws iam list-policies --scope Local \
    --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].Arn | [0]" \
    --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén tu `AWS_ACCOUNT_ID` y crea el ServiceAccount con IRSA usando `eksctl`.

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  echo $AWS_ACCOUNT_ID
  ```
  ```bash
  eksctl create iamserviceaccount \
    --cluster="$CLUSTER_NAME" \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn="arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy" \
    --override-existing-serviceaccounts \
    --region "$AWS_REGION" \
    --approve
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el ServiceAccount y su anotación IRSA.

  > **Nota.** Verifica que el SA tiene `eks.amazonaws.com/role-arn`.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n kube-system get sa aws-load-balancer-controller -o yaml | sed -n '1,120p'
  ```
  {% include step_image.html %}

#### Tarea 3.3 (instalación con Helm)

- {% include step_label.html %} Primero realiza la instalación de **Helm** con la siguiente serie de comandos dentro de la terminal.

  > **IMPORTANTE:** El comando se ejecuta desde el directorio **lab04**
  {: .lab-note .important .compact}

  ```bash
  cd ~
  HELM_VER="v3.19.4"
  ARCH="amd64"

  curl -LO "https://get.helm.sh/helm-${HELM_VER}-windows-${ARCH}.zip"
  unzip -o "helm-${HELM_VER}-windows-${ARCH}.zip"

  mkdir -p ~/bin
  mv "windows-${ARCH}/helm.exe" ~/bin/helm.exe

  # PATH para esta sesión
  export PATH="$HOME/bin:$PATH"

  # Persistir PATH (para próximas sesiones)
  grep -q 'export PATH="$HOME/bin:$PATH"' ~/.bashrc 2>/dev/null || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc

  cd Desktop/labs-eks-ckad/lab04/
  helm version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Agrega el repo de charts e instala el controller.

  > **Nota:** En ambientes donde ya existe una instalación previa, usa `helm upgrade --install ...`.
  {: .lab-note .info .compact}

  ```bash
  helm repo add eks https://aws.github.io/eks-charts
  helm repo update
  ```
  ```bash
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName="$CLUSTER_NAME" \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera el rollout del controller.

  > **Nota.** Verifica que el pod está corriendo (si falla, revisa los logs).
  {: .lab-note .info .compact}

  ```bash
  kubectl -n kube-system rollout status deploy/aws-load-balancer-controller
  ```
  ```bash
  kubectl -n kube-system get pods -l app.kubernetes.io/name=aws-load-balancer-controller -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa los logs recientes (permisos/IRSA).

  > **Nota.** Veroifica que no hay errores de **“AccessDenied”** o **“assume role”**.
  {: .lab-note .info .compact}

  ```bash
  kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=120
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

### Tarea 4. Crear certificado autofirmado e importarlo a ACM (sin dominio)

Generarás un certificado **autofirmado** con `openssl` y lo importarás a ACM. Este `CERT_ARN` será el que adjuntes al listener HTTPS del ALB.

> **IMPORTANTE:** El certificado **debe** estar en la misma región que el ALB (la región de tu clúster/control plane).
{: .lab-note .important .compact}

> **NOTA:** Para evitar depender del hostname exacto del ALB (que aún no existe), usaremos un SAN wildcard del estilo `*.${AWS_REGION}.elb.amazonaws.com`, que normalmente cubre el DNSName del ALB.
{: .lab-note .info .compact}

#### Tarea 4.1 (generar cert y key con SAN)

- {% include step_label.html %} Define el nombre (CN/SAN) para el certificado.

  ```bash
  export CERT_DOMAIN="*.${AWS_REGION}.elb.amazonaws.com"
  ```

- {% include step_label.html %} Confirma el dominio del certificado.

  > **Nota.** Verifica que el CN/SAN será consistente con el DNS del ALB en tu región.
  {: .lab-note .info .compact}

  ```bash
  echo "CERT_DOMAIN=$CERT_DOMAIN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un archivo de configuración OpenSSL con SAN.

  ```bash
  cat > scripts/openssl-san.cnf <<EOF
  [ req ]
  default_bits       = 2048
  prompt             = no
  default_md         = sha256
  req_extensions     = req_ext
  distinguished_name = dn

  [ dn ]
  CN = ${CERT_DOMAIN}

  [ req_ext ]
  subjectAltName = @alt_names

  [ alt_names ]
  DNS.1 = ${CERT_DOMAIN}
  EOF
  ```

- {% include step_label.html %} Verifica que el archivo de configuración existe.

  > **Nota.** `openssl` tendrá SAN correcto (evita cert sin SAN).
  {: .lab-note .info .compact}

  ```bash
  ls -la scripts/openssl-san.cnf
  sed -n '1,60p' scripts/openssl-san.cnf
  ```
  {% include step_image.html %}

- {% include step_label.html %} Genera la llave privada y el certificado (30 días).

  ```bash
  openssl req -x509 -nodes -days 30 -newkey rsa:2048 \
    -keyout outputs/alb.key \
    -out outputs/alb.crt \
    -config scripts/openssl-san.cnf \
    -extensions req_ext
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que se generaron los archivos `.crt` y `.key`.

  > **Nota.** Identifica que ya tienes material para importar a ACM.
  {: .lab-note .info .compact}

  ```bash
  ls -la outputs/alb.crt outputs/alb.key
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona el CN/SAN/fechas del certificado.

  > **Nota.** CN y SAN deben de coincidir con `CERT_DOMAIN` y el cert no debe estar vencido.
  {: .lab-note .info .compact}

  ```bash
  openssl x509 -in outputs/alb.crt -noout -subject -issuer -dates -ext subjectAltName
  ```
  {% include step_image.html %}

#### Tarea 4.2 (importar a ACM)

- {% include step_label.html %} Importa el certificado a ACM y guarda el ARN.

  ```bash
  CERT_ARN="$(aws acm import-certificate \
    --region "$AWS_REGION" \
    --certificate fileb://outputs/alb.crt \
    --private-key fileb://outputs/alb.key \
    --query CertificateArn --output text)"
  ```
  ```bash
  echo "CERT_ARN=$CERT_ARN" | tee outputs/cert_arn.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el ARN se guardó.

  > **Nota.** Verifica que no quedó vacío y podrás referenciarlo en el Ingress.
  {: .lab-note .info .compact}

  ```bash
  cat outputs/cert_arn.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el certificado en el servicio de AWS ACM.

  > **Nota.** Verifica que ACM lo reconoce como `IMPORTED` (y estado operativo).
  {: .lab-note .info .compact}

  ```bash
  aws acm describe-certificate \
    --region "$AWS_REGION" \
    --certificate-arn "$CERT_ARN" \
    --query "Certificate.{Status:Status,Type:Type,NotAfter:NotAfter,DomainName:DomainName}" \
    --output table
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

### Tarea 5. Desplegar aplicación demo (Deployment + Service)

Desplegarás una app sencilla (`traefik/whoami`) con 2 réplicas y probes, expuesta por un Service `ClusterIP`. El ALB apuntará a este Service mediante el Ingress.

> **NOTA (CKAD):** Aquí practicas recursos core: Deployment, Service, selectors, probes, rollout y port-forward.
{: .lab-note .info .compact}

#### Tarea 5.1

- {% include step_label.html %} Crea el namespace del laboratorio.

  ```bash
  kubectl create ns "$NAMESPACE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el namespace existe.

  ```bash
  kubectl get ns "$NAMESPACE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el manifiesto de la app en `manifests/app.yaml`.

  ```bash
  cat > manifests/app.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ${APP_NAME}
    namespace: ${NAMESPACE}
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: ${APP_NAME}
    template:
      metadata:
        labels:
          app: ${APP_NAME}
      spec:
        containers:
        - name: ${APP_NAME}
          image: traefik/whoami:v1.10.3
          ports:
          - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: ${APP_NAME}
    namespace: ${NAMESPACE}
  spec:
    selector:
      app: ${APP_NAME}
    ports:
    - name: http
      port: 80
      targetPort: 80
    type: ClusterIP
  EOF
  ```

- {% include step_label.html %} Previsualiza el manifiesto generado.

  ```bash
  sed -n '1,200p' manifests/app.yaml
  ```

- {% include step_label.html %} Aplica los manifiestos.

  ```bash
  kubectl apply -f manifests/app.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera el rollout del deployment.

  > **Nota.** Verifica que los pods llegan a `Ready`.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NAMESPACE" rollout status deploy/"$APP_NAME"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista los pods y el service.

  ```bash
  kubectl -n "$NAMESPACE" get pods -o wide
  ```
  ```bash
  kubectl -n "$NAMESPACE" get svc "$APP_NAME" -o wide
  ```
  ```bash
  kubectl -n "$NAMESPACE" get endpoints "$APP_NAME" -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba local en la terminal Bash con port-forward. Abre una **segunda terminal** para ejcutar el comando.

  ```bash
  kubectl -n "$NAMESPACE" port-forward svc/"$APP_NAME" 8080:80
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre una **tercera terminal** para ejecutar el siguiente comando CURL y confirma respuesta del `whoami`.

  > **Nota.** Verifica que la app responde antes de poner ALB enfrente.

  ```bash
  curl -s http://localhost:8080 | head -n 20
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

---

### Tarea 6. Crear Ingress ALB con TLS, redirect HTTP→HTTPS y política TLS

Crearás un Ingress que provoque la creación de un ALB con listeners 80/443, adjunte tu certificado de ACM (importado), fuerce redirect a HTTPS y aplique una SSL policy. Luego obtendrás el hostname del ALB para usarlo directamente en las pruebas (sin DNS propio).

#### Tarea 6.1 (crear manifiesto del Ingress)

- {% include step_label.html %} Ahora regresa a la **primera terminal** y rompe el proceso **`CTRL+C`**. Crea `manifests/ingress.yaml` con annotations de ALB.

  > **NOTA:** `alb.ingress.kubernetes.io/ssl-policy` define ciphers/TLS versions. Ajusta a una policy que tu organización requiera.
  {: .lab-note .info .compact}

  ```bash
  cat > manifests/ingress.yaml <<EOF
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ${APP_NAME}-alb
    namespace: ${NAMESPACE}
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
      alb.ingress.kubernetes.io/certificate-arn: ${CERT_ARN}
      alb.ingress.kubernetes.io/ssl-redirect: "443"
      alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
      alb.ingress.kubernetes.io/healthcheck-path: /
  spec:
    ingressClassName: alb
    rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: ${APP_NAME}
              port:
                number: 80
  EOF
  ```

- {% include step_label.html %} Ejecuta el `dry-run` del manifiesto antes de aplicarlo.

  > **Nota.** Recuerda siempre verificar que el YAML es válido para el cliente `kubectl` y no tiene errores de sintaxis.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply --dry-run=client -f manifests/ingress.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora si aplica el Ingress.

  ```bash
  kubectl apply -f manifests/ingress.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona el Ingress recién creado.

  ```bash
  kubectl -n "$NAMESPACE" get ingress "${APP_NAME}-alb" -o yaml | sed -n '1,200p'
  ```
  {% include step_image.html %}

#### Tarea 6.2 (esperar ALB y obtener hostname)

- {% include step_label.html %} Espera a que el controller llene el hostname del ALB.

  ```bash
  kubectl -n "$NAMESPACE" get ingress "${APP_NAME}-alb" -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa los eventos del namespace por si el hostname tarda en aparecer.

  > **Nota.** Si hay problemas de permisos, SG, subnets o annotations, normalmente aparecen aquí.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NAMESPACE" get events --sort-by=.metadata.creationTimestamp | tail -n 30
  ```
  {% include step_image.html %}

- {% include step_label.html %} Extrae el hostname y guárdalo en una variable.

  ```bash
  export ALB_DNS="$(kubectl -n "$NAMESPACE" get ingress "${APP_NAME}-alb" -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
  echo "ALB_DNS=$ALB_DNS" | tee outputs/alb_dns.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el archivo donde se guardo `ALB_DNS` no está vacío.

  > **Nota.** Verifica que el ALB fue creado y Kubernetes ya conoce su endpoint.
  {: .lab-note .info .compact}

  ```bash
  cat outputs/alb_dns.txt
  ```
  ```bash
  test -n "$ALB_DNS" && echo "OK: ALB_DNS definido" || echo "ERROR: ALB_DNS vacío"
  ```
  {% include step_image.html %}

#### Tarea 6.3 (sin DNS propio)

- {% include step_label.html %} En esta versión del lab **no necesitas** crear registros DNS. Usarás directamente `https://$ALB_DNS` para probar.

  > **Opcional (Bonus):** Si cuentas con dominio y DNS, puedes crear un CNAME/ALIAS hacia `$ALB_DNS` y probar con tu FQDN.
  {: .lab-note .info .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}

---

### Tarea 7. Pruebas HTTP/HTTPS y validación desde AWS (listeners / SSL policy / cert)

Validarás el redirect HTTP a HTTPS y que el certificado y la SSL Policy estén aplicados. También confirmarás, desde AWS, que el ALB tiene listeners 80/443 y el certificado ACM adjunto.

#### Tarea 7.1 (pruebas desde cliente)

- {% include step_label.html %} Prueba que HTTP redirige a HTTPS usando el hostname del ALB.

  ```bash
  curl -I "http://${ALB_DNS}" | sed -n '1,20p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el código 301/302 y el header `Location`.

  > **Nota.** Este comando es mas especifico, genera la misma salida del anterior pero filtrada, solo para reconfirmar.
  {: .lab-note .info .compact}

  ```bash
  curl -I "http://${ALB_DNS}" | egrep -i 'HTTP/|location:'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba HTTPS (certificado autofirmado) y valida la respuesta.

  > **Nota.** **Opción A (rápida, laboratorio):** Omite validación del certificado.
  {: .lab-note .info .compact}

  ```bash
  curl -sk "https://${ALB_DNS}" | head -n 20
  ```
  {% include step_image.html %}

  > **Nota.** **(Opción B):** Valida HTTPS confiando explícitamente en tu cert self-signed.
  {: .lab-note .info .compact}

  > **Nota.** Verifica que TLS funciona y que el certificado presentado puede ser confiado (para pruebas sin `-k`).
  {: .lab-note .info .compact}

  ```bash
  curl --cacert outputs/alb.crt "https://${ALB_DNS}" | head -n 20
  ```
  {% include step_image.html %}

- {% include step_label.html %} **Validación con debug TLS:** inspecciona el certificado presentado por el ALB.

  > **Nota.** CN/SAN/fechas del cert servido en el listener 443.
  {: .lab-note .info .compact}

  ```bash
  openssl s_client -connect ${ALB_DNS}:443 -servername ${ALB_DNS} < /dev/null 2>/dev/null \
    | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si algo falla, revisa el Ingress y eventos (diagnóstico).

  ```bash
  kubectl -n "$NAMESPACE" describe ingress "${APP_NAME}-alb" | sed -n '1,220p'
  ```
  ```bash
  kubectl -n "$NAMESPACE" get events --sort-by=.metadata.creationTimestamp | tail -n 10
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que no hay errores de reconciliación evidentes.

  > **Nota.** Los mensajes tipo `failed build model`, `AccessDenied`, `subnet`/`SG`/
  `target` pueden indicar problemas.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NAMESPACE" describe ingress "${APP_NAME}-alb" | egrep -i 'error|failed|denied' || echo "OK: no se detectaron errores obvios"
  ```
  {% include step_image.html %}

#### Tarea 7.2 (validación en AWS con CLI)

- {% include step_label.html %} Obtén el ARN del ALB a partir del DNSName.

  ```bash
  export ALB_ARN="$(aws elbv2 describe-load-balancers --region "$AWS_REGION" \
    --query "LoadBalancers[?DNSName=='${ALB_DNS}'].LoadBalancerArn | [0]" \
    --output text)"
  ```
  ```bash
  echo "ALB_ARN=$ALB_ARN" | tee outputs/alb_arn.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que `ALB_ARN` no está vacío.

  > **Nota.** Verifica que AWS CLI encontró el ALB por su DNSName.
  {: .lab-note .info .compact}

  ```bash
  cat outputs/alb_arn.txt
  test -n "$ALB_ARN" && echo "OK: ALB_ARN definido" || echo "ERROR: ALB_ARN vacío"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista los listeners y confirma puertos los 80/443.

  ```bash
  aws elbv2 describe-listeners --region "$AWS_REGION" \
    --load-balancer-arn "$ALB_ARN" \
    --query "Listeners[].{Port:Port,Protocol:Protocol,SslPolicy:SslPolicy,Certificates:Certificates}"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegura que existe listener 443 con `SslPolicy` y `Certificates`.

  > **Nota.** Verifica que el listener HTTPS está activo, con policy aplicada y cert adjunto.
  {: .lab-note .info .compact}

  ```bash
  aws elbv2 describe-listeners --region "$AWS_REGION" --load-balancer-arn "$ALB_ARN" \
    --query "Listeners[?Port==\`443\`].{Port:Port,Protocol:Protocol,SslPolicy:SslPolicy,CertArn:Certificates[0].CertificateArn}" \
    --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que tu SSL policy existe.

  ```bash
  aws elbv2 describe-ssl-policies --region "$AWS_REGION" \
    --query "SslPolicies[?Name=='ELBSecurityPolicy-TLS13-1-2-2021-06'].Name" \
    --output text
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r7 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r7 %}

---

### Tarea 8. Limpieza del laboratorio (evitar costos)

Eliminarás el Ingress (lo que desencadena el borrado del ALB), el namespace y, opcionalmente, el certificado ACM y el controller si fue instalado solo para este lab.

> **IMPORTANTE:** El ALB cuesta. Si dejas el Ingress, dejarás el ALB corriendo.
{: .lab-note .important .compact}

#### Tarea 8.1

- {% include step_label.html %} Elimina el Ingress y espera a que el controller borre el ALB.

  ```bash
  kubectl delete -f manifests/ingress.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa los eventos de borrado.

  > **Nota.** Verifica que el controller inició la reconciliación de delete (ALB empezará a eliminarse). **Puede tardar unos 2 minutos en iniciar el borrado**
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NAMESPACE" get events --sort-by=.metadata.creationTimestamp | tail -n 30
  ```

- {% include step_label.html %} Elimina la app y el namespace.

  ```bash
  kubectl delete -f manifests/app.yaml
  ```
  ```bash
  kubectl delete ns "$NAMESPACE"
  ```
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el namespace ya no existe.

  ```bash
  kubectl get ns | grep -E "^${NAMESPACE}\b" && echo "ERROR: namespace aún existe" || echo "OK: namespace eliminado"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el certificado ACM creado para la practica

  ```bash
  aws acm delete-certificate --region "$AWS_REGION" --certificate-arn "$CERT_ARN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegura que el cert ya no aparece.

  ```bash
  aws acm list-certificates --region "$AWS_REGION" \
    --query "CertificateSummaryList[?CertificateArn=='${CERT_ARN}']" --output json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si instalaste el controller solo para este lab, desinstálalo.

  ```bash
  helm uninstall aws-load-balancer-controller -n kube-system || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que ya no existe el deployment.

  ```bash
  kubectl -n kube-system get deploy aws-load-balancer-controller && echo "WARN: aún existe" || echo "OK: ya no existe"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **7 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r8 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r8 %}