# eBPF POC - Kubernetes Manifests

Manifiestos para desplegar aplicaciones Java 8 y Java 17 con PostgreSQL en Kubernetes.

## Arquitectura
```
Usuario → Java 8 Gateway → Java 17 Service → PostgreSQL
```

## Despliegue

### Orden de aplicación
```bash
# 1. Crear namespace
kubectl apply -f namespace/

# 2. Desplegar base de datos
kubectl apply -f database/

# 3. Esperar a que PostgreSQL esté listo
kubectl wait --for=condition=ready pod -l app=postgres -n ebpf-poc --timeout=120s

# 4. Desplegar Java 17 Service
kubectl apply -f java17-service/

# 5. Esperar a que Java 17 esté listo
kubectl wait --for=condition=ready pod -l app=java17-service -n ebpf-poc --timeout=120s

# 6. Desplegar Java 8 Gateway
kubectl apply -f java8-gateway/

# 7. (Opcional) Desplegar Ingress
kubectl apply -f ingress/
```

### Despliegue completo (un solo comando)
```bash
kubectl apply -f namespace/ && \
kubectl apply -f database/ && \
kubectl wait --for=condition=ready pod -l app=postgres -n ebpf-poc --timeout=120s && \
kubectl apply -f java17-service/ && \
kubectl wait --for=condition=ready pod -l app=java17-service -n ebpf-poc --timeout=120s && \
kubectl apply -f java8-gateway/
```

## Verificación
```bash
# Ver todos los recursos
kubectl get all -n ebpf-poc

# Ver pods
kubectl get pods -n ebpf-poc

# Ver servicios
kubectl get svc -n ebpf-poc

# Logs Java 8 Gateway
kubectl logs -f -l app=java8-gateway -n ebpf-poc

# Logs Java 17 Service
kubectl logs -f -l app=java17-service -n ebpf-poc

# Logs PostgreSQL
kubectl logs -f -l app=postgres -n ebpf-poc
```

## Testing

### Acceso a la App

#### En minikube


**Paso 1: Obtener la URL del servicio**
```bash
minikube service java8-gateway -n ebpf-poc --url
# Output ejemplo: http://192.168.49.2:30547
```

Esta URL combina:
- **IP del nodo**: 192.168.49.2 (IP interna de Minikube)
- **NodePort**: 30547 (puerto asignado automáticamente en rango 30000-32767)

**Paso 2: Guardar la URL en una variable (opcional)**
```bash
export GATEWAY_URL=$(minikube service java8-gateway -n ebpf-poc --url)
echo $GATEWAY_URL
```

**Alternativa: Usar minikube tunnel**

Si prefieres usar LoadBalancer real (en una terminal separada):

```bash
# En una terminal, dejar corriendo:
sudo minikube tunnel

# En otra terminal, obtener la EXTERNAL-IP
kubectl get svc java8-gateway -n ebpf-poc
# Usar la IP mostrada en EXTERNAL-IP (usualmente 10.96.x.x)

# Ahora puedes usar puerto 80 directamente
export GATEWAY_URL="http://10.96.123.45"  # Reemplaza con tu EXTERNAL-IP
```

#### En cluster kubeadm

```bash
# Obtener NodePort asignado
kubectl get svc java8-gateway -n ebpf-poc

# Output ejemplo:
# NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# java8-gateway   LoadBalancer   10.96.123.45         80:30547/TCP   5m

# Usar cualquier IP de nodo del cluster + NodePort
export GATEWAY_URL="http://:30547"

# Para obtener IPs de los nodos:
kubectl get nodes -o wide
```

### Testing de Endpoints

Una vez tengas `$GATEWAY_URL` configurada:

#### Health Check
```bash
curl $GATEWAY_URL/api/health

# Respuesta esperada:
# {"status":"UP","service":"java8-gateway"}
```

#### Crear Usuario
```bash
curl -X POST $GATEWAY_URL/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Smith","email":"alice@example.com"}'

# Respuesta esperada (con ID generado):
# {
#   "id": 4,
#   "name": "Alice Smith",
#   "email": "alice@example.com",
#   "createdAt": "2024-12-16T20:30:45.123",
#   "backend": "java17",
#   "gateway": "java8",
#   "message": "User created successfully"
# }
```

#### Obtener Usuario
```bash
# Obtener usuario con ID 1 (datos precargados)
curl $GATEWAY_URL/api/users/1

# Respuesta esperada:
# {
#   "id": 1,
#   "name": "John Doe",
#   "email": "john@example.com",
#   "createdAt": "2024-12-16T19:45:12.456",
#   "backend": "java17",
#   "gateway": "java8"
# }
```

#### Obtener Órdenes de un Usuario
```bash
curl $GATEWAY_URL/api/orders/1

# Respuesta esperada:
# {
#   "userId": 1,
#   "orders": [
#     {
#       "id": 1,
#       "userId": 1,
#       "product": "Laptop",
#       "amount": 999.99,
#       "status": "completed",
#       "createdAt": "2024-12-16T19:45:15.789"
#     },
#     {
#       "id": 2,
#       "userId": 1,
#       "product": "Mouse",
#       "amount": 29.99,
#       "status": "pending",
#       "createdAt": "2024-12-16T19:45:16.123"
#     }
#   ],
#   "count": 2,
#   "backend": "java17",
#   "gateway": "java8"
# }
```

#### Provocar Errores Aleatorios

Los servicios tienen errores aleatorios inyectados (10-15% de probabilidad). Simplemente ejecuta múltiples requests:
```bash
# Hacer 20 requests para ver algunos errores
for i in {1..20}; do
  echo "Request $i:"
  curl -s $GATEWAY_URL/api/users/1 | jq -r '.error // "OK"'
  sleep 0.5
done

# Verás respuestas como:
# Request 1: OK
# Request 2: OK
# Request 3: Simulated random error in gateway
# Request 4: OK
# Request 5: Simulated database connection error
# ...
```

#### Script de Testing Automatizado
```bash
# Ejecutar el script de pruebas incluido
./test.sh

# Esto ejecutará todas las pruebas anteriores automáticamente
```

## Limpieza
```bash
# Eliminar todo
kubectl delete namespace ebpf-poc

# O eliminar por partes
kubectl delete -f java8-gateway/
kubectl delete -f java17-service/
kubectl delete -f database/
kubectl delete -f namespace/
```

## Recursos

- **Java 8 Gateway**: 2 réplicas, 384Mi-768Mi RAM
- **Java 17 Service**: 2 réplicas, 512Mi-1Gi RAM
- **PostgreSQL**: 1 réplica, 256Mi-512Mi RAM, 1Gi storage

## Notas

- Las contraseñas están en base64 en los secrets (sé que esto está mal en producción)
- El storage usa `storageClassName: standard` (esto lo voy a dejar así, aunque no sé si esté diferente en prod)
- Los health checks esperan 30-60s para dar tiempo a la JVM de iniciar
- Los logs siguen el formato: `Fecha|HoraInicio|HoraFin|Latencia|Endpoint|StatusCode`
