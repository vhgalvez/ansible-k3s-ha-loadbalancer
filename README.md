# 🧰 HAProxy + Keepalived Deployment (Ansible)

---

## 🧱 Overview: What Are You Building?

You are building a high-availability Kubernetes environment on KVM virtual machines hosted on an HP ProLiant physical server. The setup includes K3s, HAProxy + Keepalived, Traefik as an internal Ingress, distributed storage with Longhorn + NFS, all secured with WireGuard VPN and nftables.

---

## 🌐 Network Architecture and External Access

```plaintext
[Public Users]
       │
       ▼
+-------------------+
| Cloudflare CDN    | ◄── Proxy + HTTPS + WAF
| (example.com)     |
+-------------------+
       │
       ▼
+----------------------------+
| VPS with Public IP         |
| WireGuard Gateway          |
| Tunnel: 10.17.0.1          |
+----------------------------+
       │
       ▼
+-----------------------------+
| WireGuard Server LAN       |
| NAT + VPN (192.168.0.30)   |
+-----------------------------+
       │
       ▼
Internal Traffic Redirected Based on Type
```

### 🎯 Traffic Segmentation in Production

| Traffic Type         | Assigned VIP | Function                                                |
| -------------------- | ------------ | ------------------------------------------------------ |
| Kubernetes API (6443) | 10.17.5.10   | Ensures stability for kubectl, etcd, control-plane     |
| Ingress HTTP/HTTPS    | 10.17.5.30   | Redirects traffic to internal services via Traefik     |

These virtual IPs (VIPs) are managed by HAProxy + Keepalived and automatically switch between nodes.

---

## 🧠 High-Availability Load Balancers

| Node          | IP           | Function                |
| ------------- | ------------ | ---------------------- |
| k8s-api-lb    | 192.168.0.30 | Main VIP node with br0 bridge |
| loadbalancer1 | 10.17.3.12   | HAProxy backup         |
| loadbalancer2 | 10.17.3.13   | HAProxy backup         |

The three nodes have HAProxy + Keepalived installed. The VIPs 10.17.5.10 (API) and 10.17.5.30 (Ingress) are floating, with only one node maintaining them at a time based on priority.

---

## ☸️ Kubernetes Cluster (K3s HA)

| Hostname | IP           | Role                  |
| -------- | ------------ | -------------------- |
| master1  | 10.17.4.21   | etcd + API           |
| master2  | 10.17.4.22   | etcd                 |
| master3  | 10.17.4.23   | etcd                 |
| worker1  | 10.17.4.24   | Application node     |
| worker2  | 10.17.4.25   | Application node     |
| worker3  | 10.17.4.26   | Application node     |

All nodes use Flatcar Container Linux. The K3s cluster operates in etcd HA mode. It is installed with `--tls-san 10.17.5.10` to allow `kubectl` access via the VIP.

---

## 🚪 Ingress Controller (Traefik)

| Type    | Deployment                       |
| ------- | -------------------------------- |
| Traefik | Internal deployment in Kubernetes |

Access is through the VIP `10.17.5.30` managed by HAProxy. Traefik communicates with pods via ClusterIP and is not directly exposed.

---

## 💾 Persistent Storage

| Node     | IP           | Role            |
| -------- | ------------ | -------------- |
| storage1 | 10.17.4.27   | NFS + Longhorn |

**Longhorn (RWO):**

- Microservices
- Prometheus
- Grafana
- ELK

**NFS (RWX):**

- PostgreSQL → `/srv/nfs/postgresql`
- Shared data → `/srv/nfs/shared`

---

## 🔐 Security

| Component    | Details                                |
| ------------- | --------------------------------------- |
| WireGuard     | Secure remote access from the VPS       |
| nftables      | Strict firewall on the physical server |
| Cloudflare    | HTTPS, WAF, DDoS protection            |
| Authentication | basicAuth for internal dashboards     |
| DNS/NTP       | intra-cluster (`10.17.3.11`)           |

---

## 🧠 Automation and CI/CD

| Tool           | Function                                |
| -------------- | -------------------------------------- |
| Terraform      | VM and network provisioning            |
| Ansible        | Installation and configuration (100% IaC) |
| Jenkins + ArgoCD | Internal CI/CD                        |

---

## 🖥 Machine Table

| Hostname      | IP           | Function                     |
| ------------- | ------------ | --------------------------- |
| master1       | 10.17.4.21   | K3s Master + etcd           |
| master2       | 10.17.4.22   | K3s Master + etcd           |
| master3       | 10.17.4.23   | K3s Master + etcd           |
| worker1       | 10.17.4.24   | Application node            |
| worker2       | 10.17.4.25   | Application node            |
| worker3       | 10.17.4.26   | Application node            |
| storage1      | 10.17.4.27   | Longhorn + NFS              |
| k8s-api-lb    | 192.168.0.30 | HAProxy + Keepalived (VIPs) |
| loadbalancer1 | 10.17.3.12   | HAProxy (backup)            |
| loadbalancer2 | 10.17.3.13   | HAProxy (backup)            |
| postgresql1   | 10.17.3.14   | Centralized PostgreSQL      |
| infra-cluster | 10.17.3.11   | CoreDNS + Chrony            |

---

## ✅ Advantages of This Architecture

- True high availability with multiple separated VIPs.
- Internally controlled ingress with Traefik.
- Robust security via VPN, nftables, and HTTPS.
- Full automation (Terraform + Ansible).
- Distributed and fault-tolerant storage.
- Modular design for growth without redesign.

---

# 🧰 Documentation: Bootstrap of K3s Cluster with HAProxy + Keepalived + VIPs

## 📄 Objective

Allow the `master1` node to bootstrap the K3s cluster without requiring HAProxy or Keepalived to be active and functional beforehand. This resolves the classic cyclic dependency problem ("the egg or the chicken") when using a VIP (virtual IP) as the entry point to the cluster.

---

## 🏛️ Architecture

- **API Server VIP**: `10.17.5.10`
- **Ingress VIP**: `10.17.5.30`
- **Masters**:
  - `10.17.4.21` (bootstrap)
  - `10.17.4.22`
  - `10.17.4.23`
- **Workers**:
  - `10.17.4.24`, `10.17.4.25`, `10.17.4.26`, `10.17.4.27`
- **Load Balancers**:
  - `10.17.3.12`, `10.17.3.13`, `192.168.0.30`

---


k8s-api-lb 192.168.0.30 se crea un adaptador de puente `br0` para que los nodos puedan comunicarse entre sí y con el mundo exterior.

## 🔄 Orden de inicio esperado

1. El nodo `master1` se inicializa con su IP real (`10.17.4.21`).
2. Se levanta el `k3s-server` y el `etcd` en `master1`.
3. Los otros masters se unen usando `https://10.17.4.21:6443` (no la VIP).
4. Una vez el clúster está operativo:

   * Se configura la VIP (`10.17.5.10`) con Keepalived.
   * Se habilita HAProxy en los nodos `haproxy_keepalived`.
5. HAProxy redirige el tráfico de `10.17.5.10:6443` hacia los masters disponibles.
6. El `kubeconfig` puede comenzar a usar la VIP como endpoint oficial.

---

## ✅ Configuración correcta para romper el ciclo

### 1. **`master1` usa su IP real para bootstrap**

* El script de Ansible no apunta a la VIP (`10.17.5.10`) para levantar el nodo inicial.
* Esto permite iniciar el API Server antes que HAProxy.

### 2. **HAProxy permite `bind` en IPs no locales**

```yaml
- name: Habilitar net.ipv4.ip_nonlocal_bind
  ansible.posix.sysctl:
    name: net.ipv4.ip_nonlocal_bind
    value: "1"
    sysctl_file: /etc/sysctl.d/99-haproxy-nonlocal-bind.conf
    reload: yes
    state: present
```

Esto evita errores de HAProxy como `Cannot bind to VIP`, ya que permite iniciar el proceso sin que la IP esté asignada localmente a la interfaz.

### 3. **Keepalived no requiere HAProxy para iniciar**

```ini
# override.conf
[Unit]
After=haproxy.service
# NO incluye Requires=haproxy.service
```

Esto asegura que Keepalived pueda arrancar independientemente de HAProxy. La relación es suave, no bloqueante.

### 4. **VIP solo se usa después de la estabilización**

* El uso de la VIP para el `kubeconfig` solo se hace después de validar que el balanceador HAProxy esté activo.

### 5. **Configuración de HAProxy**

El `haproxy.cfg` está correctamente estructurado para enrutar tráfico TCP en el puerto 6443 hacia los masters:

```haproxy
frontend kubernetes_api
    bind 10.17.5.10:6443
    mode tcp
    option tcplog
    default_backend kubernetes_masters

backend kubernetes_masters
    mode tcp
    balance roundrobin
    option tcp-check
    tcp-check connect port 6443
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server master-1 10.17.4.21:6443 check
    server master-2 10.17.4.22:6443 check
    server master-3 10.17.4.23:6443 check
```

---

## 🔧 Validaciones adicionales

* Comando para verificar si HAProxy permite bind:

```bash
sysctl net.ipv4.ip_nonlocal_bind
```

* Verificar configuración de HAProxy:

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
```

* Verificar si está corriendo:

```bash
sudo systemctl status haproxy
```

---

## 🧪 Estado de los Balanceadores tras el Playbook `install_haproxy_keepalived.yml`

Este es el estado esperado de los nodos balanceadores una vez finaliza la instalación automática con Ansible. Todos los nodos tienen HAProxy + Keepalived configurados, y las VIPs se asignan automáticamente por prioridad.

| Hostname        | IP           | Rol                                 | Keepalived         | HAProxy             | VIPs Activas                      |
|-----------------|--------------|--------------------------------------|--------------------|---------------------|------------------------------------|
| `k8s-api-lb`    | `192.168.0.30` | Nodo principal de VIPs (`priority=100`) | ✅ Activo (MASTER)  | ✅ Activo y corriendo | ✅ `10.17.5.10` y `10.17.5.30`      |
| `loadbalancer1` | `10.17.3.12` | Respaldo 1 (`priority=120`)         | ✅ Activo (BACKUP)  | ✅ Activo (en espera) | ❌ (asumirá VIPs si el principal cae) |
| `loadbalancer2` | `10.17.3.13` | Respaldo 2 (`priority=110`)         | ✅ Activo (BACKUP)  | ✅ Activo (en espera) | ❌ (asumirá VIPs si los anteriores caen) |

---

### ⚙️ Detalles técnicos

- Todos los nodos tienen `HAProxy` habilitado y en ejecución (`enabled + running`).
- Todos usan `ip_nonlocal_bind=1` para permitir el arranque sin poseer la VIP localmente.
- Keepalived gestiona la flotación de las siguientes IPs virtuales:
  - `10.17.5.10` → Kubernetes API (`6443`)
  - `10.17.5.30` → Ingress HTTP/HTTPS (`80` y `443`)
- El nodo que obtiene las VIPs es determinado por el archivo `host_vars/<ip>.yml` con sus prioridades:
  - `keepalived_priority_api`
  - `keepalived_priority_ingress`
- Si el nodo principal falla, el siguiente en prioridad **asume automáticamente las VIPs** y el tráfico continúa sin interrupciones.

---


## 🎯 Conclusión

Con esta configuración:

* El nodo `master1` puede iniciar el clúster sin la VIP.
* Los nodos balanceadores y Keepalived pueden arrancar sin romper dependencias.
* HAProxy puede arrancar incluso si no posee la VIP.

👍 Estás aplicando correctamente un patrón de alta disponibilidad tolerante a fallos y circularidades de dependencia.



ansible-galaxy collection install community.general

🧰 Resumen del Proyecto: HAProxy + Keepalived para K3s HA
Este proyecto implementa una solución de balanceo de carga altamente disponible para el acceso al clúster Kubernetes mediante HAProxy y Keepalived, gestionando múltiples VIPs (IP Virtuales) para separar tráfico crítico del API y del Ingress HTTP/HTTPS.

🎯 Objetivo
Garantizar:

Acceso ininterrumpido al API de Kubernetes (puerto 6443).

Disponibilidad continua para tráfico HTTP/HTTPS hacia los servicios internos (Ingress).

Failover automático de las IPs virtuales entre múltiples nodos balanceadores.

🔧 Componentes Clave
Componente	Descripción
HAProxy	Balanceador de carga TCP/HTTP para API y tráfico web
Keepalived	Gestor de alta disponibilidad mediante VRRP para mover VIPs entre nodos
VIPs	IPs flotantes que garantizan un único punto de entrada para el tráfico
Ansible	Automatización completa del despliegue y configuración
K3s	Clúster Kubernetes ligero y altamente disponible

🌐 Arquitectura General de Red
bash
Copiar
Editar
                    ┌────────────────────────────┐
                    │        Clientes externos    │
                    └────────────┬───────────────┘
                                 │
                           ┌─────▼─────┐
                           │ Cloudflare│
                           └─────┬─────┘
                                 │
                       VPN / NAT / WireGuard
                                 │
                    ┌────────────▼────────────┐
                    │  Load Balancer Principal│  <- VIPs: 10.17.5.10 / 10.17.5.30
                    │   (HAProxy + Keepalived)│
                    └────────────┬────────────┘
              ┌─────────────────┴─────────────────┐
              │                                   │
     ┌────────▼────────┐               ┌──────────▼─────────┐
     │  K3s Master #1  │               │  K3s Master #2-3    │
     │ API + etcd      │◄─────────────► API + etcd           │
     └─────────────────┘               └─────────────────────┘

              │ Ingress tráfico HTTP/HTTPS via Traefik
              ▼
         ┌────────────┐
         │  Workers   │
         └────────────┘
📦 VIPs y Tráfico
Tipo de Tráfico	Puerto(s)	VIP	Destino final
API de Kubernetes	6443	10.17.5.10	Nodos master de K3s (HA)
Servicios Ingress	80 / 443	10.17.5.30	Nodos worker vía Traefik (interno)

Las VIPs son asignadas dinámicamente al nodo con mayor prioridad activa.

Si un nodo falla, Keepalived transfiere la VIP al siguiente disponible.

🛠️ Mecanismo de Alta Disponibilidad
HAProxy:

Actúa como proxy TCP para 6443 y como proxy HTTP para 80/443.

Verifica salud de los nodos K3s y trabajadores.

Permite configuración nonlocal_bind para bindear IPs no locales.

Keepalived:

Ejecuta VRRP y scripts de tracking (estado de HAProxy).

Usa prioridad para determinar nodo MASTER de las VIPs.

Ejecuta vip_master.sh, vip_backup.sh, vip_fault.sh según evento.

Ansible:

Automatiza instalación en nodos HA.

Configura todos los archivos .cfg, .service, .conf necesarios.

Detecta Flatcar y aplica configuraciones especiales si es necesario.

📑 Flujo de Implementación con Ansible
Detecta distribución (Flatcar o no).

Instala HAProxy, Keepalived y dependencias.

Configura sysctl para permitir nonlocal_bind.

Aplica configuraciones plantilladas (haproxy.cfg.j2, keepalived.conf.j2).

Configura override de systemd para evitar dependencias circulares.

Reinicia servicios y valida salud.

✅ Ventajas Clave
Alta disponibilidad real (failover automático).

Separación de tráfico crítico.

Escalabilidad horizontal simple.

Configuración 100% automatizada y auditable (IaC).

Seguridad de acceso por VPN y Cloudflare (si aplica).
---
ansible-k8s-ha-loadbalancer/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── host_vars/
│   ├── 10.17.3.12.yml        # loadbalancer1
│   ├── 10.17.3.13.yml        # loadbalancer2
│   └── 192.168.0.30.yml        # k8s-api-lb (nodo principal)
├── playbooks/
│   └── install_haproxy_keepalived.yml
├── templates/
│   ├── haproxy/
│   │   └── haproxy.cfg.j2
│   └── keepalived/
│       └── keepalived.conf.j2
├── Makefile
└── README.md
---




🖥️ Tabla de Máquinas y Servicios
Hostname	IP	Rol	Servicio	Estado Esperado	Comentario
k8s-api-lb	192.168.0.30	Nodo principal de balanceo	haproxy	🟢 Activo	Nodo que debería mantener las VIPs activas (por prioridad más baja)
keepalived	🟢 Activo (MASTER)	Controla VIPs 10.17.5.10 (API) y 10.17.5.30 (Ingress)
loadbalancer1	10.17.3.12	Nodo de respaldo 1 de balanceo	haproxy	🟢 Activo	Nodo backup, asume VIPs si k8s-api-lb cae
keepalived	🟢 Activo (BACKUP)	Se convierte en MASTER si el nodo principal falla
loadbalancer2	10.17.3.13	Nodo de respaldo 2 de balanceo	haproxy	🟢 Activo	Segundo backup, entra si ambos anteriores fallan
keepalived	🟢 Activo (BACKUP)	Estado pasivo, listo para asumir en caso de emergencia
master1	10.17.4.21	Kubernetes API + etcd + bootstrap	k3s server	🟢 Activo	Primer nodo que inicia el cluster sin necesidad de VIP
master2	10.17.4.22	Kubernetes API + etcd	k3s server	🟢 Activo	Se une al clúster vía IP real o API VIP
master3	10.17.4.23	Kubernetes API + etcd	k3s server	🟢 Activo	Parte del quorum de etcd
worker1	10.17.4.24	Nodo de aplicación	k3s agent	🟢 Activo	Recibe tráfico HTTP/HTTPS vía Traefik + VIP Ingress
worker2	10.17.4.25	Nodo de aplicación	k3s agent	🟢 Activo	Balanceado por Traefik
worker3	10.17.4.26	Nodo de aplicación	k3s agent	🟢 Activo	Balanceado por Traefik
storage1	10.17.4.27	NFS + Longhorn	nfs-server	🟢 Activo	Montado como RWX (PostgreSQL, compartido) y RWO (Longhorn)
infra-cluster	10.17.3.11	DNS (CoreDNS), NTP (Chrony)	dns, ntp	🟢 Activo	Servidor de infraestructura para sincronía y resolución interna
postgresql1	10.17.3.14	Base de datos centralizada	postgresql	🟢 Activo	Puede estar montado en NFS compartido

🎯 Comportamiento de Failover de Keepalived
🧠 VIP 10.17.5.10 (API Server)
Asignada por defecto al nodo k8s-api-lb

Si este cae:

loadbalancer1 detecta la caída y asume la IP VIP

Si también cae, loadbalancer2 asume la IP

➡️ El acceso al API (puerto 6443) sigue funcionando sin interrupciones para kubectl, etcd, y kubelet.

🌐 VIP 10.17.5.30 (Ingress HTTP/HTTPS)
También manejada por k8s-api-lb

Redirige tráfico HTTP/HTTPS (puertos 80/443) hacia los pods (Traefik interno)

Failover automático entre los tres balanceadores según prioridad

➡️ El tráfico web externo es reenviado correctamente a través del Ingress aunque un nodo de balanceo falle.

📊 Resumen de Estados Esperados

| Nodo            | Servicio    | Estado         | Observaciones                              |
|-----------------|-------------|----------------|-------------------------------------------|
| k8s-api-lb      | haproxy     | ✅ corriendo   | Posee ambas VIPs (por prioridad)          |
|                 | keepalived  | ✅ corriendo   | Estado MASTER                             |
| loadbalancer1   | haproxy     | ✅ corriendo   | Espera en BACKUP                          |
|                 | keepalived  | ✅ corriendo   | BACKUP con menor prioridad                |
| loadbalancer2   | haproxy     | ✅ corriendo   | Espera en BACKUP                          |
|                 | keepalived  | ✅ corriendo   | BACKUP                                    |

## 📦 Importante sobre HAProxy

- Requiere `net.ipv4.ip_nonlocal_bind = 1` para aceptar conexiones en IPs VIP que no estén asignadas localmente.
- Se arranca incluso si la VIP no está disponible aún (por diseño de HA).
- Las configuraciones están correctamente desacopladas gracias al override systemd y `After=haproxy.service`.

## ✅ Conclusiones

- Tu diseño es resiliente, modular y de alta disponibilidad real.
- El clúster no depende de las VIPs para arrancar, lo cual rompe el ciclo “huevo-gallina”.
- En caso de falla de cualquier balanceador, los otros asumen sin intervención humana.
- La infraestructura está lista para producción y escalamiento.


# 📦 Instalación de HAProxy y Keepalived

```bash
sudo ansible-playbook -i inventory/hosts.ini ansible/playbooks/install_haproxy_keepalived_full.yml
```

# 🗑️ Desinstalación de HAProxy y Keepalived

```bash
sudo ansible-playbook -i inventory/hosts.ini ansible/playbooks/uninstall_haproxy_keepalived.yml
```
