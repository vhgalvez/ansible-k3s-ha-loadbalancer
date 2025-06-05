# Diseño de Red para VIPs en el Clúster Kubernetes HA

## ✅ Objetivo

Gestionar múltiples direcciones IP flotantes (VIPs) de alta disponibilidad en Kubernetes mediante `Keepalived` y un único bridge virtual compartido.

---

## 🌐 VIPs Definidas

- `10.17.5.10`: VIP para la API de Kubernetes (`6443`)
- `10.17.5.30`: VIP para el tráfico Ingress (`80` y `443`)

---

## 🧱 Topología de Red (Resumen)

| Nodo             | Interfaz | Bridge Virtual | VIPs que puede asumir             |
|------------------|----------|----------------|----------------------------------|
| `k8s-api-lb`      | `eth1`   | `br-vip`       | `10.17.5.10`, `10.17.5.30`       |
| `loadbalancer1`  | `eth1`   | `br-vip`       | `10.17.5.10`, `10.17.5.30`       |
| `loadbalancer2`  | `eth1`   | `br-vip`       | `10.17.5.10`, `10.17.5.30`       |

- Todos los nodos de balanceo tienen conectividad por el mismo bridge.
- No se asignan las VIPs manualmente. `Keepalived` se encarga de anunciarlas dinámicamente según el nodo que esté en estado `MASTER`.

---

## 🛑 ¿Se requieren dos bridges separados?

**No.**
- Ambas VIPs pertenecen a la misma red (`10.17.5.0/24`)
- Se pueden gestionar en una única interfaz de red (`eth1`)
- Basta un único `bridge` (ej. `br-vip`)

---

## 🧠 Configuración de Keepalived (Ejemplo)

```conf
vrrp_instance VI_API {
  state BACKUP
  interface eth1
  virtual_router_id 51
  priority 100
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1234
  }
  virtual_ipaddress {
    10.17.5.10
  }
}

vrrp_instance VI_INGRESS {
  state BACKUP
  interface eth1
  virtual_router_id 52
  priority 100
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1234
  }
  virtual_ipaddress {
    10.17.5.30
  }
}
```


🧠 Resumen Teórico del Sistema de Balanceo de Carga K3s HA
🎯 Objetivo General
Proveer acceso altamente disponible y balanceado a los siguientes componentes de tu clúster K3s:

API de Kubernetes (puerto 6443) – para controladores, kubectl, CI/CD, etc.

Ingress HTTP/HTTPS (puertos 80 y 443) – para acceder a servicios web expuestos públicamente.

🧱 Componentes Involucrados
🧩 1. HAProxy
Actúa como el balanceador de carga TCP.

Distribuye tráfico entrante hacia los masters (para la API) o workers (para Ingress).

Se configura con:

frontend: escucha en una IP VIP (flotante) y puerto.

backend: lista de servidores reales (nodos) a los que reenvía el tráfico con chequeo de salud TCP (check).

🧩 2. Keepalived
Se encarga de gestionar las IP flotantes (VIPs) de alta disponibilidad.

Usa el protocolo VRRP para elegir qué nodo balanceador es el MASTER y cuáles son BACKUP.

Si un nodo falla, mueve automáticamente la VIP al siguiente nodo con más prioridad.

🧩 3. VIPs (Virtual IPs)
Dos IPs virtuales compartidas entre los balanceadores:

10.17.5.10: VIP para la API de Kubernetes.

10.17.5.30: VIP para Ingress HTTP/HTTPS.

Solo un nodo tiene activamente la VIP en su interfaz br-vip, los otros la tienen en espera.

🔄 ¿Cómo funciona el flujo del tráfico?
A. Kubernetes API (6443)
El cliente (kubectl, Jenkins, etc.) se conecta a la IP 10.17.5.10:6443.

HAProxy en el nodo activo con esa VIP recibe la conexión.

Reenvía la petición a uno de los masters (round-robin + TCP check).

B. Ingress HTTP/HTTPS (80/443)
Un navegador o cliente externo se conecta a 10.17.5.30:80 o :443.

HAProxy enruta ese tráfico a uno de los workers que ejecuta el Ingress Controller (Traefik).

El Ingress decide cómo redirigir al servicio final dentro del clúster.

🛡️ Alta Disponibilidad: ¿Qué pasa si un nodo cae?
Keepalived detecta que haproxy ha muerto (gracias a vrrp_script).

Automáticamente migra la VIP al siguiente nodo disponible.

El servicio sigue disponible sin cambios para el usuario final (el dominio o IP sigue siendo la misma).

🧮 Teoría del Balanceo (HAProxy)
Modo TCP → No interpreta HTTP, solo reenvía paquetes binarios.

Algoritmo roundrobin → Distribuye conexiones entrantes de manera equitativa.

check → Verifica que los puertos estén abiertos y operativos en los nodos.

transparent → Permite que el cliente vea la IP del destino final (si está soportado en el entorno).

🌉 Interfaz br-vip
Todos los balanceadores deben tener una interfaz virtual común llamada br-vip.

Esta interfaz es donde se asignan (de forma dinámica) las IPs VIP mediante Keepalived.

Aísla el tráfico VIP del resto de la red de gestión, lo que permite controlar y mover las IPs sin conflictos.

✅ Beneficios de este diseño
Alta disponibilidad real: 3 balanceadores y VIPs redundantes.

Failover automático: no requiere intervención humana si un nodo cae.

Escalable: puedes añadir más masters o workers sin tocar el sistema de entrada.

Separación de tráfico: API y tráfico web manejados por VIPs distintos.

