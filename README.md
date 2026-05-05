#  Práctica E-Commerce — Subnetting & IP Forwarding

Diseño y configuración de una infraestructura de red con subnetting de máscara variable (VLSM), asignación de IPs estáticas mediante Netplan y activación de IP Forwarding para permitir la comunicación entre subredes distintas. Proyecto individual desarrollado en el módulo **0369 – Implantació de Sistemes Operatius** del Grado Superior ASIX en La Salle Gràcia.

---

##  Descripción

Cuando una empresa necesita segmentar su red en departamentos o zonas con diferente número de equipos, no puede permitirse desperdiciar direcciones IP asignando el mismo bloque a todas las subredes. La solución es el **subnetting de máscara variable (VLSM)**: dividir un rango de IPs en subredes de distintos tamaños, cada una con exactamente el espacio que necesita.

En esta práctica se parte de la red `10.14.208.0/21` y se calculan a mano dos subredes ajustadas a los requisitos del enunciado: una para 30 máquinas y otra para 10. Una vez definidas las subredes, se configuran tres máquinas virtuales Ubuntu (dos routers y un cliente) con IPs estáticas y rutas estáticas para que puedan comunicarse entre sí a través de subredes distintas. Para que los routers sean capaces de reenviar paquetes entre redes, es necesario activar **IP Forwarding** en el kernel de Linux.

---

##  Funcionalidades principales

- **Subnetting VLSM calculado a mano:** a partir de un bloque `/21`, se calculan dos subredes óptimas (`/26` para 30 hosts y `/28` para 10 hosts) minimizando el desperdicio de IPs
- **Configuración de red estática con Netplan:** cada máquina virtual tiene IP fija, gateway y rutas estáticas definidas en el fichero YAML de Netplan, eliminando la dependencia de DHCP
- **Routing inter-redes con IP Forwarding:** activación del reenvío de paquetes IPv4 en el kernel Linux para que los routers virtuales puedan encaminar tráfico entre las dos subredes, verificado con `ping` y `mtr`

---

##  Tecnologías utilizadas

| Herramienta | Rol en el proyecto |
|---|---|
| **Ubuntu Server / Desktop** | Sistema operativo de las tres máquinas virtuales |
| **VirtualBox** | Plataforma de virtualización |
| **Netplan** | Gestión de interfaces de red y rutas estáticas en Ubuntu (`/etc/netplan/`) |
| **sysctl** | Parámetros del kernel — activación de `net.ipv4.ip_forward=1` |
| **ping** | Verificación básica de conectividad entre máquinas |
| **mtr** | Trazado de ruta (traceroute interactivo) para verificar el camino de los paquetes |
| **Draw.io** | Diseño del diagrama lógico de red |

---

##  Estructura del proyecto

```
📁 practica-ecommerce-subnetting/
├── 📄 README.md
├── 📁 subnetting/
│   ├── calculos-vlsm.md        # Cálculos de máscara variable
│   └── tabla-subredes.md       # Tabla con IP red, rango y broadcast
├── 📁 diagrama/
│   └── diagrama-logico.drawio  # Topología de red exportada de Draw.io
└── 📁 configuracion/
    ├── netplan-client-PC1.yaml
    ├── netplan-router2-PC2.yaml
    └── netplan-router1-PC3.yaml
```

---

##  Explicación detallada del proyecto

### 1. Cálculo de subredes (VLSM)

El punto de partida es la red `10.14.208.0/21`, que proporciona un bloque de 2048 direcciones. El enunciado requiere dos subredes con capacidades distintas, por lo que se aplica **VLSM** (Variable Length Subnet Masking): en lugar de dividir el espacio en subredes iguales, se asigna a cada una la máscara más pequeña que cubra sus necesidades.

| Máscara | IP de Red | Rango de hosts disponibles | IP Broadcast |
|---|---|---|---|
| `/26` | `10.14.208.0` | `10.14.208.1 – 10.14.208.62` | `10.14.208.63` |
| `/28` | `10.14.208.64` | `10.14.208.65 – 10.14.208.78` | `10.14.208.79` |

> **Decisión técnica clave:** la subred 1 necesitaba albergar 30 máquinas. A primera vista podría parecer que una máscara `/27` (que da 30 hosts usables) sería suficiente, pero hay que tener en cuenta que los dos routers que delimitan la subred también consumen IPs de ese rango. Sumando los 30 hosts + 2 interfaces de router = 32 direcciones necesarias, lo que obliga a usar una `/26` (62 hosts usables). Este matiz es uno de los errores más comunes en el diseño de redes.

Los cálculos se realizaron a mano usando la fórmula `2^n - 2` para determinar el número de hosts por subred y ajustando los bits de host necesarios para cada caso.

---

### 2. Diagrama lógico de red

Con las subredes calculadas, se diseñó el diagrama lógico en Draw.io para visualizar la topología antes de configurar las máquinas:

```
Internet ── R1 (PC3) ── [Subred 1 /26] ── R2 (PC2) ── [Subred 2 /28] ── Cliente (PC1)
              10.14.208.1                   10.14.208.2 | 10.14.208.65        10.14.208.66
```

- **PC3 (Router1):** una interfaz hacia Internet (DHCP) y una interfaz en la subred 1 con IP `10.14.208.1/26`
- **PC2 (Router2):** una interfaz en la subred 1 (`10.14.208.2/26`) y una interfaz en la subred 2 (`10.14.208.65/28`)
- **PC1 (Cliente):** una interfaz en la subred 2 con IP `10.14.208.66/28`, gateway `10.14.208.65`

---

### 3. Configuración de red con Netplan

En Ubuntu, la configuración de red se gestiona a través de ficheros YAML en `/etc/netplan/`. Se modifica el fichero `01-network-manager-all.yaml` (o `50-cloud-init.yaml` según la VM) para asignar IP estática, deshabilitar DHCP y definir rutas estáticas.

**Cliente (PC1) — subred 2:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 10.14.208.66/28
      routes:
        - to: default
          via: 10.14.208.65
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

**Router2 (PC2) — dos interfaces, una en cada subred:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 10.14.208.2/26
      routes:
        - to: default
          via: 10.14.208.1
    enp0s8:
      dhcp4: no
      addresses:
        - 10.14.208.65/28
```

**Router1 (PC3) — dos interfaces:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true          # interfaz hacia Internet
    enp0s8:
      dhcp4: no
      addresses:
        - 10.14.208.1/26
      routes:
        - to: 10.14.208.64/28
          via: 10.14.208.2
```

Una vez editado el fichero, se aplica la configuración con `sudo netplan try`, que aplica los cambios temporalmente y pide confirmación antes de hacerlos permanentes — útil para no perder acceso a la máquina si hay un error de configuración.

> **Problema encontrado:** Netplan emitía warnings de permisos (`Permissions for /etc/netplan/... are too open`) porque el fichero tenía permisos `644`. Aunque la configuración se aplicaba correctamente, el aviso indica que el fichero debería tener permisos `600` para evitar que otros usuarios puedan leer la configuración de red. En un entorno de producción esto sería importante, especialmente si el fichero contiene credenciales VPN o Wi-Fi.

---

### 4. IP Forwarding

Por defecto, Linux descarta los paquetes que llegan a una interfaz pero cuyo destino es otra red — es decir, actúa como host, no como router. Para que PC2 y PC3 puedan reenviar paquetes entre la subred 1 y la subred 2, hay que activar **IP Forwarding** en el kernel.

Esto se hace descomentando la línea correspondiente en `/etc/sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
# Descomentar la línea:
net.ipv4.ip_forward=1

# Aplicar sin reiniciar:
sudo sysctl -p
# Output esperado: net.ipv4.ip_forward = 1
```

> **¿Por qué es necesario?** Sin IP Forwarding activo, PC2 recibe el paquete del Cliente destinado a PC3, pero lo descarta en lugar de reenviarlo porque el sistema operativo no está configurado para actuar como router. Es una medida de seguridad por defecto: un equipo de usuario final no debería reenviar tráfico de terceros.

Este paso hay que realizarlo en **todas las máquinas que actúen como router** (PC2 y PC3). En PC1 (cliente) no es necesario.

---

### 5. Verificación de conectividad

**Ping desde el Cliente (PC1) hasta Router1 (PC3):**
```
5 packets transmitted, 5 received, 0% packet loss
rtt min/avg/max/mdev = 0.600/1.150/1.452/0.314 ms
```
Todos los paquetes llegan correctamente, confirmando que el routing entre subredes funciona.

**Trazado de ruta con `mtr` (desde PC1 → PC3):**
```
Host              Loss%  Snt  Last  Avg  Best  Wrst  StDev
1. 10.14.208.65   0.0%   15   0.3   0.7   0.3   3.6   0.8
2. 10.14.208.1    0.0%   15   0.6   1.1   0.5   2.9   0.7
```

El `mtr` confirma que el paquete pasa primero por `10.14.208.65` (interfaz de Router2 en la subred 2) y después llega a `10.14.208.1` (Router1), siguiendo exactamente la ruta diseñada en el diagrama lógico. La verificación en sentido inverso (desde PC3 hacia PC1) también fue exitosa.

---

##  Resumen de la topología

| Máquina | Rol | IP(s) | Subred |
|---|---|---|---|
| PC1 (Cliente) | Host final | `10.14.208.66/28` | Subred 2 |
| PC2 (Router2) | Router inter-subredes | `10.14.208.2/26` · `10.14.208.65/28` | Subred 1 + Subred 2 |
| PC3 (Router1) | Router de borde | `10.14.208.1/26` + DHCP | Subred 1 + Internet |

---

##  Contexto académico

Práctica individual realizada en el módulo **0369 – Implantació de Sistemes Operatius** (ICC0001-UF1-PR01) del Ciclo Formativo de Grado Superior ASIX en **La Salle Gràcia** (curso 25-26), con el objetivo de comprender el diseño de redes con VLSM, la configuración manual de interfaces en Linux y el concepto de IP Forwarding como base del routing entre subredes.

**Alumno:** Joan Peña Neira · **Profesor:** Josep Maria Gaya

### Comando `mtr` con destino a 10.14.208.65 (comprobación desde PC3 hacia Cliente)

<img width="886" height="113" alt="image" src="https://github.com/user-attachments/assets/60dfa752-c9fe-4deb-b8a1-aa41d8837e68" />

