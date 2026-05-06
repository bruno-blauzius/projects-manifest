# Estratégias de Atualização de Deployments

## 📋 Objetivo
Sincronizar as imagens dos dois Deployments (massivo e pontual) de forma centralizada através do Jenkins.

---

## 🎯 Estratégia 1: Kustomize Patch (Recomendado para Kustomize)

### Descrição
Usar um overlay do Kustomize que patcha a imagem em ambos os Deployments simultaneamente.

### Estrutura de Arquivos
```
settlement-remuneration-unit/
├── kustomization.yaml (base)
├── deployment.yaml
├── deployment-pontual.yaml
├── secret.yaml
├── triggerauthentication.yaml
├── keda-scaledobject.yaml
├── keda-scaledobject-pontual.yaml
├── serviceaccount.yaml
└── overlays/
    └── prod/
        ├── kustomization.yaml
        └── image-patch.yaml
```

### Implementação

**`overlays/prod/image-patch.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: settlement-remuneration-unit-app
spec:
  template:
    spec:
      containers:
      - name: settlement-remuneration-unit
        image: bruno01/settlement-unit-remuneration:TAG_PLACEHOLDER
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: settlement-remuneration-unit-pontual
spec:
  template:
    spec:
      containers:
      - name: settlement-remuneration-unit-pontual
        image: bruno01/settlement-unit-remuneration:TAG_PLACEHOLDER
```

**`overlays/prod/kustomization.yaml`**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../settlement-remuneration-unit

patchesStrategicMerge:
- image-patch.yaml
```

### Jenkins Pipeline
```groovy
pipeline {
    agent any

    stages {
        stage('Build Image') {
            steps {
                script {
                    sh '''
                        docker build -t bruno01/settlement-unit-remuneration:${BUILD_NUMBER} .
                        docker push bruno01/settlement-unit-remuneration:${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Update Manifests') {
            steps {
                script {
                    sh '''
                        sed -i "s/TAG_PLACEHOLDER/${BUILD_NUMBER}/g" \
                            settlement-remuneration-unit/overlays/prod/image-patch.yaml
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh 'kubectl apply -k settlement-remuneration-unit/overlays/prod/'
                }
            }
        }

        stage('Commit Changes') {
            steps {
                script {
                    sh '''
                        git config user.email "jenkins@company.com"
                        git config user.name "Jenkins CI"
                        git add settlement-remuneration-unit/overlays/prod/image-patch.yaml
                        git commit -m "chore: update image to ${BUILD_NUMBER}"
                        git push origin main
                    '''
                }
            }
        }
    }
}
```

### ✅ Vantagens
- Atualiza ambos deployments em um único patch
- Mantém base intacta
- Fácil de manter múltiplos ambientes
- Versão no Git

### ❌ Desvantagens
- Precisa de sed/regex para substituir TAG
- Patch é modificado a cada deploy

---

## 🎯 Estratégia 2: kustomize set image

### Descrição
Usar comando `kustomize edit set image` para atualizar a imagem dinamicamente.

### Implementação

**Atualizar `kustomization.yaml` base com referência de imagem**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev

resources:
- serviceaccount.yaml
- deployment.yaml
- deployment-pontual.yaml
- secret.yaml
- triggerauthentication.yaml
- keda-scaledobject.yaml
- keda-scaledobject-pontual.yaml

images:
- name: settlement-unit-remuneration
  newName: bruno01/settlement-unit-remuneration
  newTag: latest
```

**Ambos Deployments devem referenciar a mesma imagem com o mesmo nome**:
```yaml
# Em deployment.yaml
containers:
- name: settlement-remuneration-unit
  image: settlement-unit-remuneration:latest

# Em deployment-pontual.yaml
containers:
- name: settlement-remuneration-unit-pontual
  image: settlement-unit-remuneration:latest
```

### Jenkins Pipeline
```groovy
pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "bruno01/settlement-unit-remuneration"
    }

    stages {
        stage('Build & Push Image') {
            steps {
                script {
                    sh '''
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Update Kustomization') {
            steps {
                script {
                    sh '''
                        cd settlement-remuneration-unit
                        kustomize edit set image settlement-unit-remuneration=${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh 'kubectl apply -k settlement-remuneration-unit/'
                }
            }
        }

        stage('Commit Changes') {
            steps {
                script {
                    sh '''
                        git config user.email "jenkins@company.com"
                        git config user.name "Jenkins CI"
                        git add settlement-remuneration-unit/kustomization.yaml
                        git commit -m "chore: update image to ${IMAGE_TAG}"
                        git push origin main
                    '''
                }
            }
        }
    }
}
```

### ✅ Vantagens
- Muito simples
- Um comando apenas
- Atualiza kustomization.yaml automaticamente
- Fácil de ler e debugar

### ❌ Desvantagens
- Modifica kustomization.yaml (precisa estar versionado no Git)
- Ambos containers precisam ter o mesmo nome de imagem
- Menos controle fino

---

## 🎯 Estratégia 3: Script Shell Customizado

### Descrição
Usar script shell para atualizar as imagens em ambos os arquivos YAML diretamente.

### Implementação

**`scripts/update-images.sh`**:
```bash
#!/bin/bash

set -e

IMAGE_TAG=$1
REGISTRY="bruno01"
APP_NAME="settlement-unit-remuneration"
IMAGE="${REGISTRY}/${APP_NAME}:${IMAGE_TAG}"

if [ -z "$IMAGE_TAG" ]; then
    echo "❌ Uso: $0 <image-tag>"
    exit 1
fi

echo "🔄 Atualizando imagens para: $IMAGE"

# Instalar yq se não existir (alternativa: usar sed)
if ! command -v yq &> /dev/null; then
    echo "⚠️  yq não encontrado. Instalando..."
    wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
    chmod +x /usr/local/bin/yq
fi

# Atualiza deployment.yaml (massivo)
echo "📝 Atualizando deployment.yaml..."
yq -i ".spec.template.spec.containers[0].image = \"$IMAGE\"" \
    settlement-remuneration-unit/deployment.yaml

# Atualiza deployment-pontual.yaml
echo "📝 Atualizando deployment-pontual.yaml..."
yq -i ".spec.template.spec.containers[0].image = \"$IMAGE\"" \
    settlement-remuneration-unit/deployment-pontual.yaml

echo "✅ Imagens atualizadas com sucesso!"
```

### Jenkins Pipeline
```groovy
pipeline {
    agent any

    stages {
        stage('Build & Push Image') {
            steps {
                script {
                    sh '''
                        docker build -t bruno01/settlement-unit-remuneration:${BUILD_NUMBER} .
                        docker push bruno01/settlement-unit-remuneration:${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Update Manifests') {
            steps {
                script {
                    sh '''
                        chmod +x scripts/update-images.sh
                        scripts/update-images.sh ${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Verify Changes') {
            steps {
                script {
                    sh '''
                        echo "=== deployment.yaml ==="
                        grep "image:" settlement-remuneration-unit/deployment.yaml | head -1
                        echo "=== deployment-pontual.yaml ==="
                        grep "image:" settlement-remuneration-unit/deployment-pontual.yaml | head -1
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh 'kubectl apply -k settlement-remuneration-unit/'
                }
            }
        }

        stage('Commit Changes') {
            steps {
                script {
                    sh '''
                        git config user.email "jenkins@company.com"
                        git config user.name "Jenkins CI"
                        git add settlement-remuneration-unit/deployment*.yaml
                        git commit -m "chore: update images to ${BUILD_NUMBER}"
                        git push origin main
                    '''
                }
            }
        }
    }
}
```

### ✅ Vantagens
- Máximo controle
- Script reutilizável
- Fácil debugar
- Pode fazer validações antes de atualizar

### ❌ Desvantagens
- Mais complexo de manter
- Dependência de `yq` ou `sed`
- Menos elegante que Kustomize

---

## 🎯 Estratégia 4: Helm Chart (RECOMENDADO) ⭐

### Descrição
Converter para Helm Chart - a solução mais robusta e profissional.

### Estrutura de Arquivos
```
settlement-remuneration-unit/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── deployment-pontual.yaml
│   ├── secret.yaml
│   ├── triggerauthentication.yaml
│   ├── keda-scaledobject.yaml
│   └── keda-scaledobject-pontual.yaml
└── scripts/
    └── update-image.sh
```

### Implementação

**`Chart.yaml`**:
```yaml
apiVersion: v2
name: settlement-remuneration-unit
description: Kafka consumer with pontual and massivo processing
type: application
version: 1.0.0
appVersion: "1.0"
```

**`values.yaml`**:
```yaml
# ========== IMAGEM (ÚNICA FONTE DA VERDADE) ==========
image:
  registry: bruno01
  repository: settlement-unit-remuneration
  tag: "02"
  pullPolicy: Always

# ========== DEPLOYMENT MASSIVO ==========
deployment:
  name: settlement-remuneration-unit-app
  replicaCount: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "500m"
  keda:
    minReplicas: 1
    maxReplicas: 10
    lagThreshold: "100"
    pollingInterval: 60
    cooldownPeriod: 300

# ========== DEPLOYMENT PONTUAL ==========
deploymentPontual:
  name: settlement-remuneration-unit-pontual
  replicaCount: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "512Mi"
      cpu: "1000m"
  keda:
    minReplicas: 2
    maxReplicas: 5
    lagThreshold: "20"
    pollingInterval: 30
    cooldownPeriod: 120

# ========== KAFKA ==========
kafka:
  topicMassivo: settlement-remuneration-unit-process-massivo
  topicPontual: settlement-remuneration-unit-process-pontual
  groupMassivo: settlement-remuneration-unit-massivo-group
  groupPontual: settlement-remuneration-unit-pontual-group
```

**`templates/deployment.yaml`** (exemplo):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.deployment.name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: settlement-remuneration-unit-app
    component: massivo-processor
spec:
  strategy:
    {{- toYaml .Values.deployment.strategy | nindent 4 }}
  selector:
    matchLabels:
      app: settlement-remuneration-unit-app
      workload-type: massivo
  template:
    metadata:
      labels:
        app: settlement-remuneration-unit-app
        workload-type: massivo
    spec:
      serviceAccountName: settlement-remuneration-unit-app
      containers:
      - name: settlement-remuneration-unit
        image: "{{ .Values.image.registry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        env:
        - name: KAFKA_TOPIC
          value: {{ .Values.kafka.topicMassivo }}
        - name: KAFKA_GROUP_ID
          value: {{ .Values.kafka.groupMassivo }}
        resources:
          {{- toYaml .Values.deployment.resources | nindent 10 }}
```

### Jenkins Pipeline
```groovy
pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        HELM_CHART = "settlement-remuneration-unit"
        NAMESPACE = "dev"
    }

    stages {
        stage('Build & Push Image') {
            steps {
                script {
                    sh '''
                        docker build -t bruno01/settlement-unit-remuneration:${IMAGE_TAG} .
                        docker push bruno01/settlement-unit-remuneration:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                script {
                    sh '''
                        # Simples: substitui a tag em UM ÚNICO lugar
                        sed -i "s/tag: .*/tag: \"${IMAGE_TAG}\"/" \
                            ${HELM_CHART}/values.yaml
                    '''
                }
            }
        }

        stage('Helm Lint') {
            steps {
                script {
                    sh 'helm lint ${HELM_CHART}/'
                }
            }
        }

        stage('Helm Dry-Run') {
            steps {
                script {
                    sh '''
                        helm upgrade --install ${HELM_CHART} \
                            ${HELM_CHART}/ \
                            -n ${NAMESPACE} \
                            --dry-run \
                            --debug
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh '''
                        helm upgrade --install ${HELM_CHART} \
                            ${HELM_CHART}/ \
                            -n ${NAMESPACE} \
                            -f ${HELM_CHART}/values.yaml \
                            --wait \
                            --timeout 5m
                    '''
                }
            }
        }

        stage('Rollout Status') {
            steps {
                script {
                    sh '''
                        kubectl rollout status deployment/settlement-remuneration-unit-app \
                            -n ${NAMESPACE} --timeout=3m
                        kubectl rollout status deployment/settlement-remuneration-unit-pontual \
                            -n ${NAMESPACE} --timeout=3m
                    '''
                }
            }
        }

        stage('Commit Changes') {
            steps {
                script {
                    sh '''
                        git config user.email "jenkins@company.com"
                        git config user.name "Jenkins CI"
                        git add ${HELM_CHART}/values.yaml
                        git commit -m "chore: update image to ${IMAGE_TAG}"
                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        success {
            script {
                sh '''
                    echo "✅ Deploy realizado com sucesso!"
                    kubectl get deployment -n ${NAMESPACE} | grep settlement
                    helm history ${HELM_CHART} -n ${NAMESPACE}
                '''
            }
        }
        failure {
            script {
                sh '''
                    echo "❌ Deploy falhou. Rollback automático em 5m..."
                    helm rollback ${HELM_CHART} -n ${NAMESPACE}
                '''
            }
        }
    }
}
```

### ✅ Vantagens
- **MELHOR PRÁTICA** em Kubernetes
- Imagem atualizada em UM ÚNICO LUGAR (values.yaml)
- Ambos deployments sincronizados automaticamente
- Rollback nativo: `helm rollback <release> <revision>`
- Histórico de deploys: `helm history`
- Suporta múltiplos ambientes (values-dev.yaml, values-prod.yaml)
- Validação integrada (helm lint)
- Dry-run antes de aplicar
- Integração melhor com CI/CD

### ❌ Desvantagens
- Aprendizado inicial (curva de Helm)
- Migração de YAML atual
- Complexidade adicional

---

## 📊 Comparativo Geral

| Critério | Kustomize Patch | Set Image | Script Shell | **Helm** |
|----------|-----------------|-----------|--------------|----------|
| **Sincronização** | ✅ Ótima | ✅ Ótima | ✅ Ótima | ✅✅ Melhor |
| **Simplicidade** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Manutenção** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **Múltiplos ambientes** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Rollback** | Manual | Manual | Manual | ✅ Automático |
| **Validação** | Não | Não | Opcional | ✅ Nativa |
| **Profissionalismo** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

---

## 🎯 Recomendação Final

### Se você quer **rápido e funcional agora**:
→ **Estratégia 2: kustomize set image**

### Se você quer **profissional e escalável**:
→ **Estratégia 4: Helm Chart** ⭐⭐⭐

---

## 🔧 Próximos Passos

1. Escolha a estratégia
2. Implemente no Jenkins
3. Teste em dev
4. Faça rollback de teste
5. Deploy em produção

Quer que eu implemente qualquer uma dessas estratégias?
