# 🚀 Guia Completo: Deploy de Aplicação Java Multi-Tier no Kubernetes

## 📋 Pré-requisitos

- Cluster Kubernetes configurado (EKS, GKE, AKS ou local com Minikube/Kind)
- kubectl instalado e configurado
- NGINX Ingress Controller instalado no cluster
- Conhecimento básico de Kubernetes e Docker

## 🏗️ Arquitetura do Projeto

```
┌─────────────────┐
│  NGINX Ingress  │
└────────┬────────┘
         │
    ┌────▼────┐
    │ vproapp │ (Spring Boot)
    └────┬────┘
         │
    ┌────┼────────────────┐
    │    │                │
┌───▼──┐ ┌──▼────┐ ┌─────▼─────┐
│MySQL │ │RabbitMQ│ │Memcached  │
└──────┘ └────────┘ └───────────┘
```

---

## 📁 Estrutura do Projeto

```
vprokube/
├── kubedefs/
│   ├── secret.yaml          # Credenciais
│   ├── dbpvc.yaml          # Persistência MySQL
│   ├── dbdeploy.yaml       # Deployment MySQL
│   ├── dbservice.yaml      # Service MySQL
│   ├── rmqdeploy.yaml      # Deployment RabbitMQ
│   ├── rmqservice.yaml     # Service RabbitMQ
│   ├── mcdep.yaml          # Deployment Memcached
│   ├── mcservice.yaml      # Service Memcached
│   ├── appdeploy.yaml      # Deployment Aplicação
│   ├── appservice.yaml     # Service Aplicação
│   └── appingress.yaml     # Ingress
└── README.md
```

---

## 🔧 Passo a Passo

### **Passo 1: Criar o Secret para Credenciais**

Crie o arquivo `kubedefs/secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # echo -n 'vprodbpass' | base64
  db-pass: dnByb2RicGFzcw==
  # echo -n 'guest' | base64
  rmq-pass: Z3Vlc3Q=
  # echo -n 'guest' | base64
  rmq-user: Z3Vlc3Q=
```

**Aplicar:**
```bash
kubectl apply -f kubedefs/secret.yaml
```

---

### **Passo 2: Configurar MySQL com Persistência**

**2.1 - Criar PersistentVolumeClaim** (`kubedefs/dbpvc.yaml`):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pv-claim
  labels:
    app: vprodb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: default
```

**2.2 - Criar Deployment MySQL** (`kubedefs/dbdeploy.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vprodb
  labels:
    app: vprodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vprodb
  template:
    metadata:
      labels:
        app: vprodb
    spec:
      containers:
      - name: vprodb
        image: kubeivan/vprofiledb
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: vpro-db-data
        ports:
        - name: vprodb-port
          containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db-pass
      volumes:
        - name: vpro-db-data
          persistentVolumeClaim:
           claimName: db-pv-claim
      initContainers:
      - name: busybox
        image: busybox
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: vpro-db-data
        args: ["rm", "-rf", "/var/lib/mysql/lost+found"]
```

**2.3 - Criar Service MySQL** (`kubedefs/dbservice.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vprodb
spec:
  type: ClusterIP
  selector:
    app: vprodb
  ports:
  - port: 3306
    targetPort: vprodb-port
    protocol: TCP
```

**Aplicar:**
```bash
kubectl apply -f kubedefs/dbpvc.yaml
kubectl apply -f kubedefs/dbdeploy.yaml
kubectl apply -f kubedefs/dbservice.yaml
```

---

### **Passo 3: Configurar RabbitMQ**

**3.1 - Criar Deployment RabbitMQ** (`kubedefs/rmqdeploy.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vprormq
  labels:
    app: vprormq
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vprormq
  template:
    metadata:
      labels:
        app: vprormq
    spec:
      containers:
      - name: vprormq
        image: rabbitmq
        ports:
        - name: vprormq-port
          containerPort: 5672
        env:
        - name: RABBITMQ_DEFAULT_USER
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: rmq-user
        - name: RABBITMQ_DEFAULT_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: rmq-pass
```

**3.2 - Criar Service RabbitMQ** (`kubedefs/rmqservice.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vpromq01
spec:
  type: ClusterIP
  selector:
    app: vprormq
  ports:
  - port: 5672
    targetPort: vprormq-port
    protocol: TCP
```

**Aplicar:**
```bash
kubectl apply -f kubedefs/rmqdeploy.yaml
kubectl apply -f kubedefs/rmqservice.yaml
```

---

### **Passo 4: Configurar Memcached**

**4.1 - Criar Deployment Memcached** (`kubedefs/mcdep.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpromc
  labels:
    app: vpromc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vpromc
  template:
    metadata:
      labels:
        app: vpromc
    spec:
      containers:
      - name: vpromc
        image: memcached
        ports:
        - name: vpromc-port
          containerPort: 11211
```

**4.2 - Criar Service Memcached** (`kubedefs/mcservice.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vprocache01
spec:
  type: ClusterIP
  selector:
    app: vpromc
  ports:
  - port: 11211
    targetPort: vpromc-port
    protocol: TCP
```

**Aplicar:**
```bash
kubectl apply -f kubedefs/mcdep.yaml
kubectl apply -f kubedefs/mcservice.yaml
```

---

### **Passo 5: Configurar Aplicação Java**

**5.1 - Criar Deployment da Aplicação** (`kubedefs/appdeploy.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vproapp
  labels:
    app: vproapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vproapp
  template:
    metadata:
      labels:
        app: vproapp
    spec:
      containers:
      - name: vproapp
        image: kubeivan/vprofileapp
        ports:
        - name: vproapp-port
          containerPort: 8080
        env:
        - name: DB_HOST
          value: vprodb
        - name: DB_PORT
          value: "3306"
        - name: DB_USER
          value: root
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db-pass
        - name: RABBITMQ_HOST
          value: vpromq01
        - name: RABBITMQ_PORT
          value: "5672"
        - name: RABBITMQ_USER
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: rmq-user
        - name: RABBITMQ_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: rmq-pass
        - name: MEMCACHED_HOST
          value: vprocache01
        - name: MEMCACHED_PORT
          value: "11211"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

**5.2 - Criar Service da Aplicação** (`kubedefs/appservice.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vproapp-service
spec:
  type: ClusterIP
  selector:
    app: vproapp
  ports:
  - port: 8080
    targetPort: vproapp-port
    protocol: TCP
```

**Aplicar:**
```bash
kubectl apply -f kubedefs/appdeploy.yaml
kubectl apply -f kubedefs/appservice.yaml
```

---

### **Passo 6: Configurar Ingress**

Crie o arquivo `kubedefs/appingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vproapp-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: vproapp.seudominio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vproapp-service
            port:
              number: 8080
```

**Aplicar:**
```bash
kubectl apply -f kubedefs/appingress.yaml
```

---

## ✅ Verificação e Testes

### **1. Verificar todos os pods:**
```bash
kubectl get pods
```

Todos devem estar com status `Running`.

### **2. Verificar services:**
```bash
kubectl get svc
```

### **3. Verificar ingress:**
```bash
kubectl get ingress
```

### **4. Ver logs da aplicação:**
```bash
kubectl logs -l app=vproapp
```

Procure por mensagens de sucesso como "Started" sem erros de autenticação.

### **5. Testar acesso:**
```bash
curl http://vproapp.seudominio.com/
```

Deve retornar status 200 com a página de login.

---

## 🐛 Troubleshooting

### **Problema: Aplicação retorna 404**

**Causa:** Aplicação não inicializou devido a erro de conexão.

**Solução:**
```bash
# Ver logs
kubectl logs -l app=vproapp --tail=100

# Se houver erro de autenticação RabbitMQ, recriar pods
kubectl delete pod -l app=vprormq
kubectl delete pod -l app=vproapp
```

### **Problema: Erro de autenticação RabbitMQ**

**Causa:** Secret não foi aplicado ou pods não foram recriados.

**Solução:**
```bash
# Verificar secret
kubectl get secret app-secret -o jsonpath='{.data.rmq-user}' | base64 -d

# Deve retornar: guest

# Se não, aplicar novamente
kubectl apply -f kubedefs/secret.yaml
kubectl delete pod -l app=vprormq
kubectl delete pod -l app=vproapp
```

### **Problema: MySQL não inicia**

**Causa:** Problema com PVC ou dados corrompidos.

**Solução:**
```bash
# Ver logs
kubectl logs -l app=vprodb

# Recriar PVC (ATENÇÃO: apaga dados)
kubectl delete -f kubedefs/dbdeploy.yaml
kubectl delete -f kubedefs/dbpvc.yaml
kubectl apply -f kubedefs/dbpvc.yaml
kubectl apply -f kubedefs/dbdeploy.yaml
```

---

## 🔄 Deploy Completo (Ordem Correta)

Execute na ordem:

```bash
# 1. Secret
kubectl apply -f kubedefs/secret.yaml

# 2. Banco de Dados
kubectl apply -f kubedefs/dbpvc.yaml
kubectl apply -f kubedefs/dbdeploy.yaml
kubectl apply -f kubedefs/dbservice.yaml

# 3. RabbitMQ
kubectl apply -f kubedefs/rmqdeploy.yaml
kubectl apply -f kubedefs/rmqservice.yaml

# 4. Memcached
kubectl apply -f kubedefs/mcdep.yaml
kubectl apply -f kubedefs/mcservice.yaml

# 5. Aguardar serviços ficarem prontos
kubectl wait --for=condition=ready pod -l app=vprodb --timeout=120s
kubectl wait --for=condition=ready pod -l app=vprormq --timeout=120s
kubectl wait --for=condition=ready pod -l app=vpromc --timeout=120s

# 6. Aplicação
kubectl apply -f kubedefs/appdeploy.yaml
kubectl apply -f kubedefs/appservice.yaml

# 7. Ingress
kubectl apply -f kubedefs/appingress.yaml

# 8. Verificar
kubectl get pods
kubectl get svc
kubectl get ingress
```

---

## 🧹 Limpeza (Remover tudo)

```bash
kubectl delete -f kubedefs/
```

---

## 📚 Conceitos Aprendidos

✅ **Deployments:** Gerenciamento de pods e replicação  
✅ **Services:** Comunicação entre pods via DNS interno  
✅ **Secrets:** Armazenamento seguro de credenciais  
✅ **PersistentVolumeClaim:** Persistência de dados  
✅ **Ingress:** Exposição de aplicações para internet  
✅ **Resource Limits:** Otimização de recursos  
✅ **InitContainers:** Preparação de ambiente antes do container principal  
✅ **Environment Variables:** Configuração de aplicações  

---

## 🎯 Próximos Passos

- Implementar Horizontal Pod Autoscaler (HPA)
- Adicionar Liveness e Readiness Probes
- Configurar Monitoring com Prometheus/Grafana
- Implementar CI/CD com GitOps (ArgoCD/Flux)
- Adicionar Network Policies para segurança
- Configurar backup automático do MySQL

---

## 📞 Suporte

Se encontrar problemas:
1. Verifique os logs: `kubectl logs <pod-name>`
2. Descreva o pod: `kubectl describe pod <pod-name>`
3. Verifique eventos: `kubectl get events --sort-by='.lastTimestamp'`

---

**Autor:** Evandro  
**Data:** Fevereiro 2026  
**Versão:** 1.0
