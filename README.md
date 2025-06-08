# 🧰 Implementación de HAProxy + Keepalived (Ansible)

---

## 🧱 Descripción: ¿Qué estás construyendo?

Estás construyendo un entorno de Kubernetes de alta disponibilidad en máquinas virtuales KVM hospedadas en un servidor físico HP ProLiant. La configuración incluye K3s, HAProxy + Keepalived, Traefik como Ingress interno, almacenamiento distribuido con Longhorn + NFS, todo asegurado con WireGuard VPN y nftables.

---

## 🌐 Arquitectura de Red y Acceso Externo

```plaintext
[Usuarios Públicos]
       │
       ▼
+-------------------+
| Cloudflare CDN    | ◄── Proxy + HTTPS + WAF
| (example.com)     |
+-------------------+
       │
       ▼
IP dinámica DNS (example.com) API Cloudflare script bash actualizar la IP
       │
       ▼
+-----------------------------+
| Servidor WireGuard LAN       |
| NAT + VPN (192.168.0.40)     |
+-----------------------------+
       │
       ▼
Tráfico Interno Redirigido según Tipo
🎯 Segmentación del Tráfico en Producción
Tipo de Tráfico	VIP Asignada	Función
Kubernetes API (6443)	192.168.0.32	Asegura estabilidad para kubectl, etcd, control-plane
Ingress HTTP/HTTPS	192.168.0.33	Redirige el tráfico hacia servicios internos vía Traefik
```

Estas IPs Virtuales (VIPs) son gestionadas por HAProxy + Keepalived y cambian automáticamente entre nodos.

---

## 🧠 Balanceadores de Carga de Alta Disponibilidad

```plaintext
Nodo	IP	Función
loadbalancer1	192.168.0.30	HAProxy master + VIP
loadbalancer2	192.168.0.31	HAProxy backup + VIP
```

Los tres nodos tienen HAProxy + Keepalived instalados. Las VIPs 192.168.0.32 (API) y 192.168.0.33 (Ingress) son flotantes, con solo un nodo manteniéndolas en cada momento según la prioridad.

---

## ☸️ Clúster de Kubernetes (K3s HA)

```plaintext
Hostname	IP	Rol
master1	10.17.4.21	etcd + API
master2	10.17.4.22	etcd
master3	10.17.4.23	etcd
worker1	10.17.4.24	Nodo de aplicación
worker2	10.17.4.25	Nodo de aplicación
worker3	10.17.4.26	Nodo de aplicación
```

Todos los nodos usan Flatcar Container Linux. El clúster K3s opera en modo etcd HA. Se instala con `--tls-san 192.168.0.32` para permitir acceso a kubectl vía la VIP.

---

## 🚪 Controlador de Ingress (Traefik)

```plaintext
Tipo	Implementación
Traefik	Implementación interna en Kubernetes
```

El acceso se realiza a través de la VIP 192.168.0.33 gestionada por HAProxy. Traefik se comunica con los pods a través de ClusterIP y no está expuesto directamente.

---

## 💾 Almacenamiento Persistente

```plaintext
Nodo	IP	Rol
storage1	10.17.4.27	NFS + Longhorn
```

Longhorn (RWO):
- Microservicios
- Prometheus
- Grafana
- ELK

NFS (RWX):
- PostgreSQL → `/srv/nfs/postgresql`
- Datos compartidos → `/srv/nfs/shared`

---

## 🔐 Seguridad

```plaintext
Componente	Detalles
WireGuard	Acceso remoto seguro desde el VPS
nftables	Cortafuegos estricto en el servidor físico
Cloudflare	HTTPS, WAF, Protección DDoS
Autenticación	basicAuth para dashboards internos
DNS/NTP	Intra-clúster (10.17.3.11)
```

---

## 🧠 Automatización y CI/CD

```plaintext
Herramienta	Función
Terraform	Aprovisionamiento de VM y red
Ansible	Instalación y configuración (100% IaC)
Jenkins + ArgoCD	CI/CD interno
```

---

## 🖥 Tabla de Máquinas

```plaintext
Hostname	IP	Función
master1	10.17.4.21	K3s Master + etcd
master2	10.17.4.22	K3s Master + etcd
master3	10.17.4.23	K3s Master + etcd
worker1	10.17.4.24	Nodo de aplicación
worker2	10.17.4.25	Nodo de aplicación
worker3	10.17.4.26	Nodo de aplicación
storage1	10.17.4.27	Longhorn + NFS
loadbalancer1	192.168.0.30	HAProxy + Keepalived (VIPs)
loadbalancer2	192.168.0.31	HAProxy (backup)
postgresql1	10.17.3.14	PostgreSQL centralizado
infra-cluster	10.17.3.11	CoreDNS + Chrony
```

---

## ✅ Ventajas de Esta Arquitectura

- Alta disponibilidad real con múltiples VIPs separadas.
- Ingress controlado internamente con Traefik.
- Seguridad robusta a través de VPN, nftables y HTTPS.
- Automatización completa (Terraform + Ansible).
- Almacenamiento distribuido y tolerante a fallos.
- Diseño modular para crecimiento sin rediseño.

---

## 🧰 Documentación: Arranque del Clúster K3s con HAProxy + Keepalived + VIPs

### 📄 Objetivo

Permitir que el nodo master1 arranque el clúster K3s sin requerir que HAProxy o Keepalived estén activos y funcionales previamente. Esto resuelve el clásico problema de dependencia cíclica ("el huevo o la gallina") cuando se usa una VIP (IP Virtual) como punto de entrada al clúster.

### 🏛️ Arquitectura

```plaintext
API Server VIP: 192.168.0.32
Ingress VIP: 192.168.0.33

Masters:
10.17.4.21 (bootstrap)
10.17.4.22
10.17.4.23

Workers:
10.17.4.24, 10.17.4.25, 10.17.4.26, 10.17.4.27

Load Balancers:
192.168.0.30, 192.168.0.31
```

---

### 🔄 Orden de Inicio Esperado

1. El nodo master1 se inicializa con su IP real (10.17.4.21).
2. Se levanta el k3s-server y el etcd en master1.
3. Los otros masters se unen usando `https://10.17.4.21:6443` (no la VIP).
4. Una vez el clúster está operativo:
   - Se configura la VIP (192.168.0.32) con Keepalived.
   - Se habilita HAProxy en los nodos haproxy_keepalived.
   - HAProxy redirige el tráfico de `192.168.0.32:6443` hacia los masters disponibles.
   - El kubeconfig puede comenzar a usar la VIP como endpoint oficial.

---

### ✅ Configuración Correcta para Romper el Ciclo

1. **master1 usa su IP real para bootstrap**
   - El script de Ansible no apunta a la VIP (192.168.0.32) para levantar el nodo inicial.
   - Esto permite iniciar el API Server antes que HAProxy.

2. **HAProxy permite bind en IPs no locales**

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

3. **Keepalived no requiere HAProxy para iniciar**

```ini
# override.conf
[Unit]
After=haproxy.service
# NO incluye Requires=haproxy.service
```

Esto asegura que Keepalived pueda arrancar independientemente de HAProxy.

4. **VIP solo se usa después de la estabilización**
   - El uso de la VIP para el kubeconfig solo se hace después de validar que el balanceador HAProxy esté activo.

5. **Configuración de HAProxy**

```haproxy
frontend kubernetes_api
    bind 192.168.0.32:6443
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

### 🔧 Validaciones Adicionales

1. Verificar si HAProxy permite bind:

```bash
sysctl net.ipv4.ip_nonlocal_bind
```

2. Verificar configuración de HAProxy:

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
```

3. Verificar si está corriendo:

```bash
sudo systemctl status haproxy
```

---

### 🧪 Estado de los Balanceadores tras el Playbook `install_haproxy_keepalived.yml`

```plaintext
Nodo	Servicio	Estado	Observaciones
keepalived	✅ corriendo	Estado MASTER
loadbalancer1	haproxy	✅ corriendo	Posee ambas VIPs (por prioridad)
keepalived	✅ corriendo	BACKUP con menor prioridad
loadbalancer2	haproxy	✅ corriendo	Espera en BACKUP
keepalived	✅ corriendo	BACKUP
```

---

### 📦 Importante sobre HAProxy

- Requiere `net.ipv4.ip_nonlocal_bind = 1` para aceptar conexiones en IPs VIP que no estén asignadas localmente.
- Se arranca incluso si la VIP no está disponible aún (por diseño de HA).
- Las configuraciones están correctamente desacopladas gracias al override systemd y `After=haproxy.service`.

---

### ✅ Conclusiones

- Tu diseño es resiliente, modular y de alta disponibilidad real.
- El clúster no depende de las VIPs para arrancar, lo cual rompe el ciclo “huevo-gallina”.
- En caso de falla de cualquier balanceador, los otros asumen sin intervención humana.
- La infraestructura está lista para producción y escalamiento.

---

### 📦 Instalación de HAProxy y Keepalived

```bash
sudo ansible-playbook -i inventory/hosts.ini ansible/playbooks/install_haproxy_keepalived_full.yml
```

---

### 🗑️ Desinstalación de HAProxy y Keepalived

```bash
sudo ansible-playbook -i inventory/hosts.ini ansible/playbooks/uninstall_haproxy_keepalived.yml
```