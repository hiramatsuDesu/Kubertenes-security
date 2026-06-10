## Introducción.

Construir un sistema de seguridad en kubernetes que pueda integrarse a cualquier infraestructura. Kubernetes permite almacenar credenciales mediante Secrets. Sin embargo, estos pueden ser consultados por usuarios con permisos suficientes y, dependiendo de la aplicación, terminar expuestos en archivos temporales, variables de entorno o memoria del proceso. El objetivo de este laboratorio es reducir esa exposición utilizando Vault como gestor centralizado de secretos.  

Para ello, sin utilizar helm para transparencia pero manteniendo la simpleza y seguridad se suben los archivos yaml.  


Los componentes de esta infra son simples:  
- Vault como "baúl" externo donde almacenamos nuestros secrets (credenciales, usuario, contraseñas)
- Vault Agent Sidecar: En un entorno productivo sería recomendable utilizar Vault Agent Injector o CSI Driver. Para simplificar el laboratorio y evitar la complejidad asociada a certificados TLS y componentes adicionales, se utiliza Vault Agent Sidecar.  

De esta manera las aplicaciones que deben ser desplegadas consumen las credenciales que el sidecard referencia a vault. Esto no va a evitar que en algún punto de la ejecución las contraseñas se muestren por comando o aparezcan en algún manifiesto. Por ello es necesario complementarlo con un sistema de accesos y permisos.  


- RBAC: Control de Acceso Basado en Roles y creamos dos roles:  
- Admin  
Control total  


- Audit  
Ver recursos  
Ver consumo CPU  
Ver consumo RAM  


NO crear  
NO borrar  
NO modificar  
NO ejecutar comandos  
NO ver secretos  


## Diagrama. [https://mermaid.ai/live/edit#pako:eNqdVg1vozYY_iuWq5NayWmAQkM5bVKa9LasuWvVpD1pyzQ5YBJ2BEdgumVV__v8haGEbslFioTx87xfft7XvMCQRgQGME7pX-Ea5wxMHxb5IhP_Dx_AD-_99P5tuSR5RhgpwCgtC0byQ4hFuVzleLsGt_7stwXcNwJOCxKWecJ2vRQvzxbw9yomwH-GPhty9ozkz0lIwDAMaZmxQoOB_s2GfwzHnydfaqQGBgBHmyTrgD-OJ_MueBklbB8-vu7AsmRDihCnJFo2GCSLOvO4vxuLOtzTqB093zHh820dc4-XinUhdeQKKcJ9DymDlrB5Fen4-v1If_k6N2Hw573KyX3tXO63SiX2pUuxuV8b8W94PEB3T7hM2VFSexo-TkV4kgnEeXGtvtXV8HH-81s5Dku2Bp8JW9PoPQ0-3E1vZsbuA01J-wwFwlRPALqVp2C6iBrWoTgJk7VUmGOVNp2MJo2A72mahAnZ1920oTsO2emgwWlcpinAYUiK4qyLZTSoWSIHcJoTHPVolu5AseMdvgEbwvIk7DShxan4jfyUFTC-Bnw65ISdHZDw7Gb0cDOv87194mcvyHtzQgKVb2W-_7-l_Q7NguGKZHzI4t2Ro3L4082XWr_KyuksiUiIc9AHk-xPEjKat2elpJmjVLS2-jRGH5zGtKSnMLI6CtDdxQL6adLQe_9ZxNtXFS36bdcKqz23sa0QJFZG0AJGy3OSPX_3LBlut1xqmCU0K447kvt7IawRzRhOMpI3JcX36qrLxuFu2vtVxWWLzClNGwCZ6P5wPi61-gX4NL37OgP_geiwoDG9Xg-oHMRXgliKDXM18Rc_mmtWLsx1IXBmIbfEjJUP9VSUSzNwlLWqGYUBs1AGakHLda02tV3VvR2_rPFe_OIETPxmYa4zE7_ZasVv3pvR1_CrRl9GQciVVW6IHlsFAgVNKaDLgt9CeJmkSYQjFdK0YbLujTqzlgeRGa9LOy1dKvlxYhJSxVRPHanol2r-HnYIeqH7Uj4r8IHqnLFdmmSrQ9BhiotiTGLwzS9AnKRpcEIuYieOUMFy-o0EJzbxfeKhkFc2D06syB1g--MbqhwamhxfEC_2DNknjotxRXax7frhWzKWQ0979mOPXBmye4Etd1CR7aVHHKtF3m4rvzH3bBlqvPRDy6qo5NKzLUU1dNNYqFIokseKTPshI2Skzh6ZjkNGu0ifOy9fHZpUAKobEdWaRloTyPQlMuJESiOoloSsbNNu3aOocbmgSjaoblpUqxxVQpK1bpqrehqZHkACtt1-hAiu8iSCActLguCG5BsslvBFTNEFZGuyIQsY8MeIxPKjES6yV07b4uxXSjcVM6flag2DGKcFX5XbCDMyTjAf9DWEz16Sj8QHPgxsy5Y2YPAC_4ZBz_H9c8vxPO9qMLj0HMtBcAcDx7PO3cGV5V4ObPtq4NreK4L_SLfOuWW7jm8P3KuB4124zuu_Si4mnQ]

```
flowchart LR

%% =========================
%% Kubernetes Cluster
%% =========================
subgraph K8S["Kubernetes Cluster (security-lab)"]

    subgraph SA["Service Accounts"]
        SA_ADMIN["ServiceAccount: admin"]
        SA_AUDIT["ServiceAccount: audit"]
        SA_DB["ServiceAccount: timescaledb"]
    end

    subgraph PODS["Pods"]
        POD_ADMIN["Pod: admin-test"]
        POD_AUDIT["Pod: audit-test"]
        POD_DB["Pod: TimescaleDB"]
    end

    JWT_ADMIN["JWT admin"]
    JWT_AUDIT["JWT audit"]
    JWT_DB["JWT timescaledb"]

end

%% =========================
%% Vault
%% =========================
subgraph VAULT["Vault Server"]

    AUTH["Kubernetes Auth Method"]

    subgraph ROLES["Vault Roles"]
        ROLE_ADMIN["Role: admin"]
        ROLE_AUDIT["Role: audit"]
        ROLE_DB["Role: timescaledb"]
    end

    subgraph POLICIES["Vault Policies"]
        POL_ADMIN["Policy: admin (full access)"]
        POL_AUDIT["Policy: audit (read-only system metrics)"]
        POL_DB["Policy: timescaledb (read DB secret)"]
    end

    subgraph SECRETS["Vault KV Secrets"]
        SECRET_DB["secret/timescaledb"]
    end

end

%% =========================
%% Vault Agent Layer
%% =========================
subgraph AGENT["Vault Agent (Sidecar / Injector)"]

    AGENT_ADMIN["Agent admin"]
    AGENT_AUDIT["Agent audit"]
    AGENT_DB["Agent timescaledb"]

    FILE_ADMIN["/vault/secrets/admin"]
    FILE_AUDIT["/vault/secrets/audit"]
    FILE_DB["/vault/secrets/db.env"]

end

%% =========================
%% Applications
%% =========================
subgraph APPS["Containers"]
    APP_ADMIN["Admin App"]
    APP_AUDIT["Audit Tool"]
    DB["TimescaleDB"]
end

%% =========================
%% ========== FLOWS ==========
%% =========================

%% --- Admin flow ---
POD_ADMIN --> SA_ADMIN --> JWT_ADMIN
JWT_ADMIN --> AUTH --> ROLE_ADMIN --> POL_ADMIN --> SECRET_DB
SECRET_DB --> AGENT_ADMIN --> FILE_ADMIN --> APP_ADMIN

%% --- Audit flow ---
POD_AUDIT --> SA_AUDIT --> JWT_AUDIT
JWT_AUDIT --> AUTH --> ROLE_AUDIT --> POL_AUDIT

%% audit no consume secrets, solo observabilidad
POL_AUDIT --> FILE_AUDIT --> APP_AUDIT

%% --- DB flow ---
POD_DB --> SA_DB --> JWT_DB
JWT_DB --> AUTH --> ROLE_DB --> POL_DB --> SECRET_DB
SECRET_DB --> AGENT_DB --> FILE_DB --> DB

%% =========================
%% Styling
%% =========================
classDef k8s fill:#e3f2fd,stroke:#1e88e5,color:#0d47a1;
classDef vault fill:#f3e5f5,stroke:#8e24aa,color:#4a148c;
classDef agent fill:#e8f5e9,stroke:#43a047,color:#1b5e20;
classDef app fill:#fff3e0,stroke:#fb8c00,color:#e65100;

class SA_ADMIN,SA_AUDIT,SA_DB,POD_ADMIN,POD_AUDIT,POD_DB,JWT_ADMIN,JWT_AUDIT,JWT_DB k8s;
class AUTH,ROLE_ADMIN,ROLE_AUDIT,ROLE_DB,POL_ADMIN,POL_AUDIT,POL_DB,SECRET_DB vault;
class AGENT_ADMIN,AGENT_AUDIT,AGENT_DB,FILE_ADMIN,FILE_AUDIT,FILE_DB agent;
class APP_ADMIN,APP_AUDIT,DB app;

```


## RBAC

```
Usuario
   ↓
Rol
   ↓
Permisos
   ↓
Recursos Kubernetes

```


Componentes de RBAC
Hay cuatro objetos importantes:  
- Role  
- ClusterRole  
- RoleBinding  
- ClusterRoleBinding  


1. Role  
Un Role define permisos dentro de un namespace.  
Ejemplo:  
kind: Role  
sólo en:  
namespace: database  


2. ClusterRole  
Define permisos para todo el cluster.  
kind: ClusterRole  
Diferencia:  

```

Role
 └─ Un namespace

ClusterRole
 └─ Todo el cluster

```

3. RoleBinding  
Une:  
```

Usuario
      ↓
Role

```

Ejemplo:  
kind: RoleBinding  
```
usuario audit
       ↓
role readonly
```


4. ClusterRoleBinding  
Une:  
```
Usuario
      ↓
ClusterRole
```

Ejemplo:  
kind: ClusterRoleBinding  
```
usuario admin
       ↓
cluster-admin
```


Una vez creadas las Service Accounts, se crean los Roles o ClusterRoles correspondientes y posteriormente se asocian mediante RoleBindings o ClusterRoleBindings.

```
kubectl apply -f serviceAccount-admin.yaml
```

## Vault
Vault es una aplicación.
Cuando despliegas Vault en Kubernetes realmente estás creando:
```
Pod
 └── vault
 8200/tcp

```

Consideraciones especiales:  
- Vault statefulset:  
- Vault necesita almacenamiento persistente para conservar secretos, configuraciones, políticas y metadatos. Aunque el laboratorio utiliza una configuración simplificada, se emplea un Persistent Volume para evitar perder la información ante reinicios.  


Orden de despliegue de vault  
```

kubectl apply -f serviceAccount-vault.yaml
kubectl apply -f vault-pvc.yaml
kubectl apply -f vault-configMap.yaml
kubectl apply -f vault-service.yaml
kubectl apply -f vault-statefulset.yaml
kubectl get pods 

```

### Siguiente paso: verificar el estado de Vault
Entramos adentro del pod vault-0 
``` 
kubectl exec -it vault-0 -n security-lab -- sh
vault status

Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.19.5
Build Date         2025-05-29T09:17:06Z
Storage Type       file
HA Enabled         false
/ # 

| Estado              | Significado                    |
| ------------------- | ------------------------------ |
| Initialized = false | Nunca se creó la clave maestra |
| Initialized = true  | Ya existe la clave maestra     |
| Sealed = true       | Vault está bloqueado           |
| Sealed = false      | Vault está operativo           |


vault operator init -address=http://127.0.0.1:8200

Unseal Key 1: <redacted>
Unseal Key 2: <redacted>
Unseal Key 3: <redacted>
Unseal Key 4: <redacted>
Unseal Key 5: <redacted>

Initial Root Token: <redacted>

3 veces + 3 primeras llaves hasta que sealed = false
vault status -address=http://127.0.0.1:8200
/ # vault login -address=http://127.0.0.1:8200
introdimos el token

/ # vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/

En otra terminal (afuera del pod):
kubectl create token vault

cat ~/.minikube/ca.crt

-----BEGIN CERTIFICATE-----
certificado
-----END CERTIFICATE-----
```

## Kubernetes Auth
El método Kubernetes Auth permite que Vault confíe en los JWT emitidos por Kubernetes. Para ello se configura un token reviewer y el certificado CA del cluster.  
```
Flujo de autenticación:
Pod
 ↓
ServiceAccount
 ↓
JWT
 ↓
Vault Kubernetes Auth
 ↓
Vault Role
 ↓
Vault Policy
 ↓
Vault Token
```

volvemos adentro del pod

```
vault write auth/kubernetes/config \
  token_reviewer_jwt="<token>" \
  kubernetes_host="https://127.0.0.1:64214" \
  kubernetes_ca_cert=@/tmp/ca.crt

verificamos si vault se puede comunicar correctamente con kubernetes authenticate
/ # vault auth list
Path           Type          Accessor                    Description                Version
----           ----          --------                    -----------                -------
kubernetes/    kubernetes    auth_kubernetes_a59ac985    n/a                        n/a
token/         token         auth_token_0115aeca         token based credentials    n/a

```

La respuesta contiene un client_token emitido por Vault. Esto confirma que el JWT del Pod fue validado correctamente y que el Role asociado permitió la autenticación.

```
/ # 

  creamos la policy
  cat > audit-policy.hcl <<EOF
path "sys/health" {
  capabilities = ["read"]
}

path "sys/metrics" {
  capabilities = ["read"]
}

path "sys/seal-status" {
  capabilities = ["read"]
}
EOF

vault policy write audit audit-policy.hcl

vault policy read audit

kubectl get sa
NAME      AGE
admin     11h
audit     11h
default   12h
vault     3h14m

creamos el role en vault (audit)
vault write auth/kubernetes/role/audit \
    bound_service_account_names=audit \
    bound_service_account_namespaces=security-lab \
    policies=audit \
    ttl=1h

verificar:
vault read auth/kubernetes/role/audit
Key                                         Value
---                                         -----
alias_name_source                           serviceaccount_uid
bound_service_account_names                 [audit]
bound_service_account_namespace_selector    n/a
bound_service_account_namespaces            [security-lab]
policies                                    [audit]
token_bound_cidrs                           []
token_explicit_max_ttl                      0s
token_max_ttl                               0s
token_no_default_policy                     false
token_num_uses                              0
token_period                                0s
token_policies                              [audit]
token_ttl                                   1h
token_type                                  default
ttl                                         1h

```

creamos un pod de prueba. vault-debug.yaml

```
kubectl apply -f vault-debug.yaml

entramos a vault-debug
entramos al pod
/ # apk add curl

cat /var/run/secrets/kubernetes.io/serviceaccount/token
JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

/ # cat /var/run/secrets/kubernetes.io/serviceaccount/token
/ # certificatate
```

en otra terminal (afuera del pod de vault-debug)

```
kubectl port-forward svc/vault 8200:8200 -n security-lab

curl \
>   --request POST \
>   --data "{\"jwt\":\"$JWT\",\"role\":\"admin\"}" \
>   http://localhost:8200/v1/auth/kubernetes/login
{"request_id":"7c6f83f3-8d7b-fdb3-8909-b2ca5ee41f2a","lease_id":"","renewable":false,"lease_duration":0,"data":null,"wrap_info":null,"warnings":null,"auth":{"client_token":"hvs.token","accessor":"qWHb27FOn766whSHHpNizxfD","policies":["admin","default"],"token_policies":["admin","default"],"metadata":{"role":"admin","service_account_name":"admin","service_account_namespace":"security-lab","service_account_secret_name":"","service_account_uid":"f0029921-512c-46cb-89a9-375bbd5b1fcd"},"lease_duration":3600,"renewable":true,"entity_id":"bd38004d-0dca-e315-6adf-5fef66c3e04c","token_type":"service","orphan":true,"mfa_requirement":null,"num_uses":0},"mount_type":""}

```
Esto significa: logueado


## TimescaleDB
Cuando TimescaleDB arranca, la contraseña termina existiendo en memoria del proceso y en el filesystem temporal montado por Vault.  

Por eso la separación:  
```
admin -> puede entrar a pods
audit -> no puede entrar a pods
```

## Vault Agent Sidecar
El Vault Agent realiza automáticamente la autenticación contra Vault utilizando el JWT asociado al ServiceAccount del Pod. Posteriormente consulta los secretos autorizados por la Policy y los renderiza como archivos dentro de un volumen compartido.

```

Vault
 │
 ▼
Vault Agent
 │
 ▼
Archivo
 │
 ▼
Aplicación


security-lab
│
├── vault-0
│
└── admin-test
     │
     ├── app
     └── vault-agent

     ------------
```
Así:
```
TimescaleDB Pod
│
├── app (postgres/timescaledb)
└── vault-agent (sidecar)
        ↓
/vault/secrets/postgres
        ↓
DB arranca consumiendo credenciales obtenidas dinámicamente desde Vault.
```


```
kubectl exec -it vault-0 -n security-lab -- sh
dentro del pod vault-0
vault kv put secret/timescaledb \
  username=ts_admin \
  password=ts_pass_123
verificar

/ # vault kv get secret/timescaledb

policy para DB

cat > /tmp/timescale-policy.hcl <<EOF
path "secret/data/timescaledb" {
  capabilities = ["read"]
}
EOF

role Kubernetes
vault write auth/kubernetes/role/timescaledb \
  bound_service_account_names=timescaledb \
  bound_service_account_namespaces=security-lab \
  policies=timescaledb \
  ttl=1h
```

## ServiceAccount

fuera del pod de vault-0
```
kubectl create sa timescaledb -n security-lab
```
Desplegar en este orden:

```
kubectl apply -f vault-agent-timescale.yaml
kubectl apply -f vault-agent-timescale-template.yaml
kubectl apply -f timescaledb.yaml

kubectl logs -f timescaledb -c vault-agent -n security-lab
kubectl exec -it timescaledb -n security-lab -- sh
cat /vault/secrets/db.env

POSTGRES_USER=ts_admin
POSTGRES_PASSWORD=ts_pass_123

```
- Esto confirma que el Vault Agent está realizando automáticamente el mismo flujo que anteriormente ejecutábamos manualmente:

```
JWT
  ↓
auth/kubernetes/login
  ↓
Vault Token
  ↓
vault kv get
  ↓
Archivo renderizado
```


- Verificamos:
```
kubectl get pods -n security-lab
kubectl logs timescaledb -c vault-agent -n security-lab
```

## Oportunidades de mejora
- Migrar de Vault Agent Sidecar a Vault CSI Driver.
- Habilitar TLS para Vault.
- Implementar renovación automática de credenciales.
- Incorporar MinIO utilizando el mismo patrón de autenticación.
- Utilizar credenciales dinámicas para bases de datos.
- Configurar Vault en modo HA.
- Integrar OIDC para autenticación de usuarios.


