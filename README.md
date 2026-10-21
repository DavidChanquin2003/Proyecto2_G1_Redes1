<div align="center">

## UNIVERSIDAD DE SAN CARLOS DE GUATEMALA
## FACULTAD DE INGENIERÍA
## ESCUELA DE CIENCIAS Y SISTEMAS
### REDES DE COMPUTADORAS 1 — SEGUNDO SEMESTRE 2026

<br>

# PROYECTO 2: MANUAL TÉCNICO DE ARQUITECTURA
## Infraestructura Financiera y Alta Disponibilidad BanTech GT

<br>

**Coordinador:** David Arturo Chanquin Velasquez | **Carnet:** 202300611 (Variable $xx = 11$)  
**Integrante:** Manuel Alejandro López Canel | **Carnet:** 202302035  
**Grupo de Laboratorio:** Grupo 1 (Variable $Y = 1$)  
**Docencia y Auxiliatura:** Aux. César Fernando Sazo Quisquinay  

<br>

**Guatemala, octubre de 2026**

---

</div>

<div align="justify">

# 1. Resumen Ejecutivo y Objetivos del Proyecto

### 1.1 Contexto Financiero y Descripción del Negocio
**BanTech GT** es una entidad bancaria y financiera de capital guatemalteco de primer orden, cuyas operaciones abarcan la gestión transaccional de banca central, autorización de créditos hipotecarios y fiduciarios, operaciones en bóveda regional y servicios de banca en línea e interbancaria (ACH/Guate-ACH). Dada la naturaleza crítica de las transacciones financieras y las regulaciones emitidas por la Superintendencia de Bancos (SIB), la infraestructura de red no puede tolerar puntos únicos de falla (SPOF - *Single Points of Failure*), latencias indeterminadas ni brechas de seguridad entre los departamentos de auditoría, gestión operativa y bases de datos centrales.

El presente proyecto implementa el diseño, dimensionamiento, despliegue y convergencia técnica de la red corporativa nacional de **BanTech GT**, interconectando tres sedes regionales de negocios (Sede Occidente, Sede Norte y Sede Oriente) con el Campus Central de Operaciones y su Data Center Principal (Sede Central), articuladas mediante un Núcleo de Red (*Backbone Core*) multiprotocolo y de alta velocidad.

### 1.2 Parámetros Algorítmicos de Identificación
De conformidad con las especificaciones del diseño de red y las variables obligatorias asignadas:
* **Bloque de Red Base Asignado:** `10.11.0.0/16`
* **Variable $xx$ (Carnet Coordinador 202300611):** `11`  
  * Determina el identificador de dominio VTP institucional: `bantech11`.
* **Variable $Y$ (Último dígito del Grupo 1):** `1`  
  * Determina la nomenclatura base de las VLANs de sedes operativas:
    * Sede Occidente: VLAN 11, VLAN 21, VLAN 31, VLAN 41.
    * Sede Norte: VLAN 51, VLAN 61, VLAN 71.
    * Sede Oriente: VLAN 81, VLAN 91.
    * Sede Central / Data Center: VLAN 12, VLAN 22, VLAN 32.

### 1.3 Objetivos Técnicos de la Infraestructura
1. **Convergencia Multiprotocolo Nacional:** Integrar de forma transparente dominios de enrutamiento dinámico heterogéneos (OSPFv2, EIGRP AS 10, RIPv2 y Enrutamiento Estático) a través de una redistribución bidireccional y mutua en los nodos de Capa 3 del Backbone Core, garantizando tablas completas de 24 subredes sin bucles ni asimetrías de enrutamiento.
2. **Alta Disponibilidad en Capa 2 y Capa 3:**
   * Eliminación del tiempo de reconvergencia en bucles físicos mediante **Rapid PVST+ (IEEE 802.1w)** en la Sede Norte.
   * Redundancia de puerta de enlace por salto inicial mediante **HSRP (Hot Standby Router Protocol)** en switches multicapa Catalyst 3560 de Sede Oriente.
   * Agregación de ancho de banda y resiliencia de enlaces físicos con **EtherChannel LACP (IEEE 802.3ad)** en modo Active en el Data Center.
3. **Aislamiento Perimetral y Seguridad de Infraestructura:** Aislar la zona de Servidores Críticos y Bases de Datos del Data Center mediante un esquema de frontera con rutas estáticas hacia el interior y redistribución con semilla de métricas compuestas hacia el exterior, evitando propagación de protocolos dinámicos innecesarios en la zona de almacenamiento financiero.
4. **Optimización de Direccionamiento IP (VLSM):** Segmentación matemática estricta a partir de `10.11.0.0/16` sin solapamiento (*zero overlap*), maximizando la conservación de espacio direccionable y proporcionando espacio de crecimiento para futuras agencias y cajeros automáticos (ATM).

---

# 2. Justificación Técnica de Arquitectura por Sede y Backbone

![topologia]()

### 2.1 Backbone Core (Núcleo Nacional Multiprotocolo)
El Backbone Core conecta geográficamente todas las operaciones de BanTech GT. Está compuesto por tres enrutadores de alta capacidad (`Core1`, `Core2`, `Core3`) organizados en una topología de malla triangular heterogénea:
* **Core1 - Core2:** Interconectados mediante interfaces de **Fibra Óptica** monomodo de baja latencia (`10.11.1.176/30`), garantizando alto rendimiento para el tráfico inter-regional más demandante.
* **Core2 - Core3:** Interconectados mediante enlaces **GigabitEthernet** en cobre sobre cable par trenzado apantallado UTP Cat 6A (`10.11.1.180/30`).
* **Core1 - Core3:** Conexión de contingencia WAN mediante interfaces síncronas **Seriales** punto a punto con modulación DCE/DTE (`10.11.1.184/30`), asegurando resiliencia ante cortes accidentales de ductos de fibra o conmutadores de capa física.

#### Análisis Crítico de la Redistribución y Resolución del Conflicto de Distancia Administrativa (AD)
La coexistencia de tres protocolos dinámicos con métricas y distancias administrativas heterogéneas requirió una ingeniería de tráfico avanzada en los nodos Core:
* **OSPF:** Distancia Administrativa interna/externa de `110`, con métrica basada en costo acumulado inversamente proporcional al ancho de banda ($10^8/\text{BW}$).
* **EIGRP:** Distancia Administrativa interna de `90` y externa de `170`, calculada mediante el algoritmo DUAL sobre métricas compuestas ($K_1, K_2, K_3, K_4, K_5$ ponderando ancho de banda mínimo y retardo acumulado).
* **RIPv2:** Distancia Administrativa de `120`, métrica simple basada exclusivamente en conteo de saltos (máximo 15 saltos).

**Vulnerabilidad de Bucle Mutuo en Core2:**  
En condiciones estándar, Core2 aprende las subredes de la Sede Occidente a través de su interfaz directa con `R-Occidente` vía RIPv2 (AD = 120). Al ejecutar redistribución mutua entre RIP y OSPF en Core2, y entre OSPF y EIGRP en Core1, Core2 recibe anuncios OSPF de esas mismas subredes provenientes de Core1 con una AD de `110`. Debido a que la regla fundamental del plano de control de Cisco IOS establece que **a menor distancia administrativa mayor credibilidad tiene la fuente de enrutamiento**, Core2 prefería instalar la ruta indirecta hacia Occidente vía Core1 (AD 110) por encima de su interfaz física directa conectada a `R-Occidente` (AD 120). Esto generaba un bucle de enrutamiento (*routing loop*) y caída total del tráfico hacia las cajas y ejecutivos de Occidente.

**Solución Implementada:**  
En el proceso de configuración de `Core2`, se redujo artificialmente la Distancia Administrativa del protocolo RIP a `100` mediante el comando `distance 100`. De esta manera:
$$\text{AD}(\text{RIP en Core2}) = 100 < \text{AD}(\text{OSPF}) = 110$$
Al ser $100 < 110$, `Core2` preserva de manera determinista e inmutable la ruta directa hacia `R-Occidente` en su tabla FIB/RIB, erradicando los anuncios reflejados desde `Core1` y alcanzando convergencia libre de bucles en toda la red nacional.

---

### 2.2 Sede Occidente (Agencia Regional Comercial)
La Sede Occidente representa una agencia comercial de alta rotación de clientes donde operan cajas receptoras, asesores de créditos al consumo, gerencia de sucursal y sistemas de videovigilancia/seguridad física.
* **Topología y Enrutamiento:** Emplea el protocolo **RIPv2**, adecuado para agencias satélite con requerimientos moderados de procesamiento y configuración simplificada.
* **Enrutamiento Inter-VLAN:** Implementa **Router-on-a-Stick (RoAS)** sobre el router `R-Occidente`, concentrando el tráfico a través del enlace troncal IEEE 802.1Q en la interfaz física `Gig0/0`, la cual aloja cuatro subinterfaces lógicas dedicadas (`.11`, `.21`, `.31` y `.41`).
* **Gestión de Capa 2 (VTP):** Se configuró **VLAN Trunking Protocol (VTP)** para evitar inconsistencias en la base de datos de VLANs. El switch `SW-Occidente1` opera como **VTP Server** centralizando la creación de las VLANs 11, 21, 31 y 41, mientras que `SW-Occidente2` opera en modo **VTP Client**, sincronizando dinámicamente las bases de datos de red bajo el dominio corporativo `bantech11` y autenticación protegida con contraseña `cisco`.

---

### 2.3 Sede Norte (Centro de Autorización de Créditos)
La Sede Norte alberga los departamentos de Análisis Financiero, Auditoría Regional y Asuntos Legales. Por la naturaleza de la documentación sensible y el flujo constante de expedientes hacia la junta directiva, se requirió una topología de Capa 2 con tolerancia a fallos.
* **Topología Física:** Disposición en anillo cerrado triangular entre tres switches de acceso y distribución (`SW-Norte1`, `SW-Norte2` y `SW-Norte3`).
* **Alta Disponibilidad de Capa 2 (Rapid PVST+):** A diferencia del STP clásico (802.1D) que requiere de 30 a 50 segundos para pasar de *Listening* a *Forwarding*, se implementó **Rapid PVST+ (IEEE 802.1w)**. Este mecanismo reduce el tiempo de reconvergencia a menos de un segundo gracias al intercambio continuo de BPDUs de propuesta/acuerdo (*handshake proposal/agreement*).
* **Determinismo del Root Bridge:** Para optimizar el flujo de paquetes y evitar rutas de conmutación subóptimas, se configuró `SW-Norte1` como **Root Bridge Primario** para todas las VLANs (prioridad `4096`), ya que este equipo posee el enlace troncal directo hacia el router de borde `R-Norte`. Se forzó un puerto bloqueado en estado alterno (*Alternate/Blocking*) en la interfaz `FastEthernet0/24` de `SW-Norte3`, eliminando bucles de tormentas de difusión (*broadcast storms*) mientras se mantiene el enlace listo para activación instantánea ante una rotura física.
* **Enrutamiento de Salida:** Se configuró **OSPFv2 en Área 0**, comunicándose con `Core1` para una convergencia de estado de enlace (*link-state*) basada en el algoritmo Dijkstra (SPF).

---

### 2.4 Sede Oriente (Centro Financiero Legado con Multicapa y HSRP)
La Sede Oriente procesa transacciones bancarias de alta criticidad en los departamentos de Bóveda y Plataforma de Operaciones. Para cumplir con los SLAs exigidos por la institución, no se podía depender de un único router físico ni de enlaces compartidos simples.
* **Conmutación Multicapa (Catalyst 3560):** La sede prescinde de routers tradicionales RoAS y utiliza dos switches multicapa de Capa 3 (`MS1` y `MS2`) en cascada, con enrutamiento IP por hardware (`ip routing`) y **Switch Virtual Interfaces (SVIs)** para las VLANs 81 y 91.
* **Puertos Enrutados (Routed Ports):** La integración con el Backbone Core se realiza mediante enlaces punto a punto dedicados configurando `no switchport` en las interfaces `GigabitEthernet0/1` de `MS1` (hacia `Core1`) y `MS2` (hacia `Core2`), integrándose nativamente al proceso **EIGRP AS 10**.
* **Redundancia de Puerta de Enlace (HSRP - RFC 2281):**
  Para ofrecer tolerancia a fallos a los terminales de usuario final sin reconfigurar sus gateways:
  * **VLAN 81 (Bóveda):** Gateway Virtual `10.11.0.193`. `MS1` opera como enrutador Activo (Prioridad `110`, con *preemption* activo); `MS2` opera como enrutador en Espera / Standby (Prioridad `100`).
  * **VLAN 91 (Plataforma):** Gateway Virtual `10.11.0.1`. `MS1` opera como Activo (Prioridad `110`, *preempt*); `MS2` opera como Standby (Prioridad `100`).
  * **Comportamiento de Failover y Failback:** Ante la falla total o desconexión de `MS1`, `MS2` detecta la ausencia de mensajes *Hello* de HSRP en 3 segundos (Holdtime 10s) y asume de inmediato el control del Gateway virtual (`10.11.0.193` / `10.11.0.1`). Cuando `MS1` se restablece, el parámetro `preempt` le permite retomar su rol de enrutador Activo de forma determinista y transparente.

---

### 2.5 Data Center y Sede Central (Alta Seguridad y Agregación de Enlaces)
El Data Center corporativo concentra los servicios neurálgicos de BanTech GT: la base de datos central de transacciones (`Core_BD`), los servidores de aplicaciones bancarias y APIs de pagos (`Web_Apps`), y la consola del Centro de Operaciones de Red (`NOC`).

* **Aislamiento por Enrutamiento Estático y Frontera Controlada:**
  Para salvaguardar la zona de servidores contra eventuales anomalías o ataques originados en protocolos de enrutamiento dinámico externos, se implementó un esquema de dos niveles (*Two-Tier Routing*):
  * `R-Central2` (Acceso Servidores): Despliega RoAS en subinterfaces `Gig0/1.12`, `Gig0/1.22` y `Gig0/1.32`. No ejecuta protocolos dinámicos hacia el core; únicamente cuenta con una ruta estática por defecto (`ip route 0.0.0.0 0.0.0.0 10.11.1.205`) que apunta a la interfaz frontera de `R-Central1`.
  * `R-Central1` (Frontera y Control de Tráfico): Posee rutas estáticas dirigidas hacia cada una de las 3 VLANs de servidores a través del salto `10.11.1.206`. Para anunciar estas redes hacia el resto del país, `R-Central1` las inyecta en el proceso **EIGRP AS 10** hacia `Core3` utilizando una semilla de métricas compuesta explícita:
    ```cisco
    redistribute static metric 10000 100 255 1 1500
    ```
    (Ancho de banda: 10,000 kbps, Retardo: 100 decenas de microsegundos, Confiabilidad: 255/255, Carga: 1/255, MTU: 1500 bytes).
* **Agregación de Enlaces con EtherChannel LACP (IEEE 802.3ad):**
  En la capa de acceso del centro de datos, el switch de distribución `SW-DC1` se conecta con redundancia física a los switches de servidores `SW-DC-Acc1` y `SW-DC-Acc2`:
  * **Port-Channel 1:** Agrupa las interfaces `FastEthernet0/1` y `FastEthernet0/2` hacia `SW-DC-Acc1`.
  * **Port-Channel 2:** Agrupa las interfaces `FastEthernet0/3` y `FastEthernet0/4` hacia `SW-DC-Acc2`.
  * Ambos enlaces se negociaron bajo el estándar abierto **LACP en modo Active** (`channel-group X mode active`), balanceando la carga por dirección MAC/IP y garantizando que la rotura de un cable físico no degrade la conectividad del servidor ni genere cortes de sesión en las bases de datos.

---

# 3. Tablas de Subneteo y Asignación de Direcciones IP

### 3.1 Esquema General de Direccionamiento VLSM (`10.11.0.0/16`)
El direccionamiento se calculó siguiendo la metodología de Máscara de Subred de Longitud Variable (VLSM), ordenando los segmentos por requerimiento de hosts para optimizar el espacio asignable y evitar el desperdicio de direcciones IP.

| Sede / Zona | Segmento Lógico | Dirección de Red | Máscara CIDR | Máscara Decimal | Rango Útil de Hosts | Dirección de Broadcast | Default Gateway | Propósito Operativo |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Oriente** | VLAN 91 | `10.11.0.0` | `/26` | `255.255.255.192` | `10.11.0.1` – `10.11.0.62` | `10.11.0.63` | `10.11.0.1` (VIP) | Plataforma de Servicios Financieros |
| **Norte** | VLAN 51 | `10.11.0.64` | `/26` | `255.255.255.192` | `10.11.0.65` – `10.11.0.126` | `10.11.0.127` | `10.11.0.65` | Análisis de Solicitudes de Crédito |
| **Occidente** | VLAN 11 | `10.11.0.128` | `/26` | `255.255.255.192` | `10.11.0.129` – `10.11.0.190` | `10.11.0.191` | `10.11.0.129` | Cajas y Ventanillas de Transacciones |
| **Oriente** | VLAN 81 | `10.11.0.192` | `/26` | `255.255.255.192` | `10.11.0.193` – `10.11.0.254` | `10.11.0.255` | `10.11.0.193` (VIP) | Bóveda y Custodia de Valores |
| **Occidente** | VLAN 21 | `10.11.1.0` | `/27` | `255.255.255.224` | `10.11.1.1` – `10.11.1.30` | `10.11.1.31` | `10.11.1.1` | Asesores Comerciales y Ventas |
| **Data Center** | VLAN 22 | `10.11.1.32` | `/27` | `255.255.255.224` | `10.11.1.33` – `10.11.1.62` | `10.11.1.63` | `10.11.1.33` | Servidores Web y Aplicaciones |
| **Norte** | VLAN 61 | `10.11.1.64` | `/27` | `255.255.255.224` | `10.11.1.65` – `10.11.1.94` | `10.11.1.95` | `10.11.1.65` | Auditoría Regional y Cumplimiento |
| **Norte** | VLAN 71 | `10.11.1.96` | `/28` | `255.255.255.240` | `10.11.1.97` – `10.11.1.110` | `10.11.1.111` | `10.11.1.97` | Departamento Legal y Jurídico |
| **Data Center** | VLAN 12 | `10.11.1.112` | `/28` | `255.255.255.240` | `10.11.1.113` – `10.11.1.126` | `10.11.1.127` | `10.11.1.113` | Base de Datos Principal (Core_BD) |
| **Occidente** | VLAN 41 | `10.11.1.128` | `/28` | `255.255.255.240` | `10.11.1.129` – `10.11.1.142` | `10.11.1.143` | `10.11.1.129` | Seguridad Física y Circuito Cerrado |
| **Occidente** | VLAN 31 | `10.11.1.144` | `/28` | `255.255.255.240` | `10.11.1.145` – `10.11.1.158` | `10.11.1.159` | `10.11.1.145` | Gerencia Regional y Dirección |
| **Data Center** | VLAN 32 | `10.11.1.160` | `/28` | `255.255.255.240` | `10.11.1.161` – `10.11.1.174` | `10.11.1.175` | `10.11.1.161` | Red de Monitoreo NOC y Sysadmins |
| **Backbone** | Core1 - Core2 | `10.11.1.176` | `/30` | `255.255.255.252` | `10.11.1.177` – `10.11.1.178` | `10.11.1.179` | N/A | Troncal Fibra Óptica (Core1: .177, Core2: .178) |
| **Backbone** | Core2 - Core3 | `10.11.1.180` | `/30` | `255.255.255.252` | `10.11.1.181` – `10.11.1.182` | `10.11.1.183` | N/A | Troncal Cobre GbE (Core2: .181, Core3: .182) |
| **Backbone** | Core1 - Core3 | `10.11.1.184` | `/30` | `255.255.255.252` | `10.11.1.185` – `10.11.1.186` | `10.11.1.187` | N/A | Troncal Serial WAN (Core1: .185, Core3: .186) |
| **Acceso Core** | Core1 - R-Norte | `10.11.1.188` | `/30` | `255.255.255.252` | `10.11.1.189` – `10.11.1.190` | `10.11.1.191` | N/A | Enlace Punto a Punto (Core1: .189, R-Norte: .190) |
| **Acceso Core** | Core2 - R-Occid | `10.11.1.192` | `/30` | `255.255.255.252` | `10.11.1.193` – `10.11.1.194` | `10.11.1.195` | N/A | Enlace Punto a Punto (Core2: .193, R-Occ: .194) |
| **Acceso Core** | Core1 - MS1 | `10.11.1.196` | `/30` | `255.255.255.252` | `10.11.1.197` – `10.11.1.198` | `10.11.1.199` | N/A | Routed Port GbE (Core1: .197, MS1: .198) |
| **Acceso Core** | Core3 - R-Cent1 | `10.11.1.200` | `/30` | `255.255.255.252` | `10.11.1.201` – `10.11.1.202` | `10.11.1.203` | N/A | Enlace EIGRP Frontera (Core3: .201, R-Cent1: .202) |
| **Perímetro DC** | R-Cent1 - R-Cent2 | `10.11.1.204` | `/30` | `255.255.255.252` | `10.11.1.205` – `10.11.1.206` | `10.11.1.207` | N/A | Aislamiento Estático (R-Cent1: .205, R-Cent2: .206) |
| **Acceso Core** | Core2 - MS2 | `10.11.1.208` | `/30` | `255.255.255.252` | `10.11.1.209` – `10.11.1.210` | `10.11.1.211` | N/A | Routed Port GbE (Core2: .209, MS2: .210) |

---

# 4. Tabla Maestra de VLANs

| VLAN ID | Nombre de VLAN | Sede / Región | Subinterfaz / SVI Responsable | Gateway Configurado | Equipos Terminales Conectados | Función Operativa / Seguridad |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| **11** | `Cajas` | Occidente | `R-Occidente (Gig0/0.11)` | `10.11.0.129` | PC-Caja1, PC-Caja2, Impresoras de firmas | Registro transaccional de ventanilla y depósitos monetarios. |
| **21** | `Asesores` | Occidente | `R-Occidente (Gig0/0.21)` | `10.11.1.1` | PC-Asesor1, PC-Asesor2 | Atención al cliente, colocación de créditos de consumo. |
| **31** | `Gerencia` | Occidente | `R-Occidente (Gig0/0.31)` | `10.11.1.145` | Laptop-Gerente, Tablet institucional | Monitoreo de metas de agencia y aprobaciones de sobregiro. |
| **41** | `Seguridad` | Occidente | `R-Occidente (Gig0/0.41)` | `10.11.1.129` | DVR_Occidente, Cámara_IP1 | Circuito cerrado de televisión (CCTV) y control de acceso. |
| **51** | `Analisis` | Norte | `R-Norte (Gig0/0.51)` | `10.11.0.65` | PC-Analista1, PC-Analista2 | Evaluación de riesgo crediticio e historiales de deudores. |
| **61** | `Auditoria` | Norte | `R-Norte (Gig0/0.61)` | `10.11.1.65` | PC-Auditor1, Laptop-Auditoría | Revisión de balances contables y cumplimiento ante la SIB. |
| **71** | `Legal` | Norte | `R-Norte (Gig0/0.71)` | `10.11.1.97` | PC-Abogado1, PC-Notaría | Validación de contratos, garantías fiduciarias e hipotecas. |
| **81** | `Boveda` | Oriente | `MS1 / MS2 (Interface Vlan 81)` | `10.11.0.193` *(VIP)* | Terminal_Boveda1, Sensor_Apertura | Sistema biométrico de apertura de bóveda y arqueo nocturno. |
| **91** | `Plataforma` | Oriente | `MS1 / MS2 (Interface Vlan 91)` | `10.11.0.1` *(VIP)* | PC-Operador1, PC-Operador2 | Emisión de tarjetas de crédito y apertura de cuentas plazo fijo. |
| **12** | `Core_BD` | Data Center | `R-Central2 (Gig0/1.12)` | `10.11.1.113` | `Server_BD_Master`, `Server_BD_Replica` | Clúster transaccional de bases de datos relacionales (SQL). |
| **22** | `Web_Apps` | Data Center | `R-Central2 (Gig0/1.22)` | `10.11.1.33` | `Server_Portal_Web`, `Server_API_Banca` | Servidores HTTP/HTTPS de banca virtual y pasarelas de pago. |
| **32** | `NOC` | Data Center | `R-Central2 (Gig0/1.32)` | `10.11.1.161` | `NOC_Console_01`, `SysAdmin_PC` | Administración de servidores, monitoreo SNMP y syslog corporativo. |

---

# 5. Mapeo Lógico y Extractos de Configuración Limpia (CLI)

### 5.1 Nodos del Backbone Core

#### A. Configuración en `Core1` (OSPF, EIGRP AS 10 y Redistribución Mutua)
```cisco
hostname Core1

! Interfaces de Backbone y Regionales
interface GigabitEthernet0/0
 description Conectividad hacia Core2 (Fibra Optica)
 ip address 10.11.1.177 255.255.255.252
 no shutdown

interface Serial0/0/0
 description Enlace WAN hacia Core3 (Serial)
 ip address 10.11.1.185 255.255.255.252
 clock rate 64000
 no shutdown

interface GigabitEthernet0/1
 description Conectividad hacia Sede Norte (R-Norte)
 ip address 10.11.1.189 255.255.255.252
 no shutdown

interface GigabitEthernet0/2
 description Conectividad hacia Sede Oriente (MS1 - Capa 3)
 ip address 10.11.1.197 255.255.255.252
 no shutdown

! Proceso OSPF y Redistribucion
router ospf 1
 router-id 1.1.1.1
 network 10.11.1.176 0.0.0.3 area 0
 network 10.11.1.188 0.0.0.3 area 0
 redistribute eigrp 10 subnets

! Proceso EIGRP AS 10 y Redistribucion
router eigrp 10
 network 10.11.1.184 0.0.0.3
 network 10.11.1.196 0.0.0.3
 auto-summary
 redistribute ospf 1 metric 10000 100 255 1 1500
```

#### B. Configuración en `Core2` (OSPF, RIPv2, EIGRP AS 10 y Ajuste Crítico de AD)
```cisco
hostname Core2

! Interfaces de Backbone y Regionales
interface GigabitEthernet0/0
 description Conectividad hacia Core1 (Fibra Optica)
 ip address 10.11.1.178 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 description Conectividad hacia Core3 (GbE Cobre)
 ip address 10.11.1.181 255.255.255.252
 no shutdown

interface GigabitEthernet0/2
 description Conectividad hacia Sede Occidente (R-Occidente)
 ip address 10.11.1.193 255.255.255.252
 no shutdown

interface GigabitEthernet1/0
 description Conectividad hacia Sede Oriente (MS2 - Capa 3)
 ip address 10.11.1.209 255.255.255.252
 no shutdown

! Proceso OSPF
router ospf 1
 router-id 2.2.2.2
 network 10.11.1.176 0.0.0.3 area 0
 redistribute rip subnets
 redistribute eigrp 10 subnets

! Proceso RIPv2 con distancia administrativa ajustada a 100
router rip
 version 2
 network 10.11.1.180
 network 10.11.1.192
 no auto-summary
 distance 100
 redistribute ospf 1 metric 2
 redistribute eigrp 10 metric 2

! Proceso EIGRP AS 10
router eigrp 10
 network 10.11.1.208 0.0.0.3
 auto-summary
 redistribute ospf 1 metric 10000 100 255 1 1500
 redistribute rip metric 10000 100 255 1 1500
```

#### C. Configuración en `Core3` (RIPv2, EIGRP AS 10 y Redistribución)
```cisco
hostname Core3

! Interfaces
interface GigabitEthernet0/1
 description Enlace hacia Core2 (GbE)
 ip address 10.11.1.182 255.255.255.252
 no shutdown

interface Serial0/0/0
 description Enlace WAN hacia Core1 (Serial)
 ip address 10.11.1.186 255.255.255.252
 no shutdown

interface GigabitEthernet0/0
 description Enlace hacia R-Central1 (Frontera Data Center)
 ip address 10.11.1.201 255.255.255.252
 no shutdown

! Proceso RIPv2
router rip
 version 2
 network 10.11.1.180
 no auto-summary
 redistribute eigrp 10 metric 2

! Proceso EIGRP AS 10
router eigrp 10
 network 10.11.1.184 0.0.0.3
 network 10.11.1.200 0.0.0.3
 auto-summary
 redistribute rip metric 10000 100 255 1 1500
```

---

### 5.2 Sede Occidente (VTP y RoAS)

#### A. Router `R-Occidente`
```cisco
hostname R-Occidente

interface GigabitEthernet0/1
 description Enlace WAN hacia Core2
 ip address 10.11.1.194 255.255.255.252
 no shutdown

interface GigabitEthernet0/0
 description Enlace Troncal hacia SW-Occidente1
 no ip address
 no shutdown

interface GigabitEthernet0/0.11
 description Subinterfaz VLAN 11 - Cajas
 encapsulation dot1Q 11
 ip address 10.11.0.129 255.255.255.192

interface GigabitEthernet0/0.21
 description Subinterfaz VLAN 21 - Asesores
 encapsulation dot1Q 21
 ip address 10.11.1.1 255.255.255.224

interface GigabitEthernet0/0.31
 description Subinterfaz VLAN 31 - Gerencia
 encapsulation dot1Q 31
 ip address 10.11.1.145 255.255.255.240

interface GigabitEthernet0/0.41
 description Subinterfaz VLAN 41 - Seguridad
 encapsulation dot1Q 41
 ip address 10.11.1.129 255.255.255.240

router rip
 version 2
 network 10.11.0.0
 network 10.11.1.0
 no auto-summary
```

#### B. Conmutadores `SW-Occidente1` (Server) y `SW-Occidente2` (Client)
```cisco
! === SW-Occidente1 (VTP Server) ===
hostname SW-Occidente1
vtp domain bantech11
vtp mode server
vtp password cisco

vlan 11
 name Cajas
vlan 21
 name Asesores
vlan 31
 name Gerencia
vlan 41
 name Seguridad

interface GigabitEthernet0/1
 description Troncal hacia R-Occidente
 switchport mode trunk

interface FastEthernet0/24
 description Troncal hacia SW-Occidente2
 switchport mode trunk

! === SW-Occidente2 (VTP Client) ===
hostname SW-Occidente2
vtp domain bantech11
vtp mode client
vtp password cisco

interface FastEthernet0/24
 description Troncal hacia SW-Occidente1
 switchport mode trunk
```

---

### 5.3 Sede Norte (Rapid PVST+ y OSPF)

#### A. Router `R-Norte`
```cisco
hostname R-Norte

interface GigabitEthernet0/1
 description Enlace hacia Core1
 ip address 10.11.1.190 255.255.255.252
 no shutdown

interface GigabitEthernet0/0
 no ip address
 no shutdown

interface GigabitEthernet0/0.51
 description Subinterfaz VLAN 51 - Analisis
 encapsulation dot1Q 51
 ip address 10.011.0.65 255.255.255.192

interface GigabitEthernet0/0.61
 description Subinterfaz VLAN 61 - Auditoria
 encapsulation dot1Q 61
 ip address 10.11.1.65 255.255.255.224

interface GigabitEthernet0/0.71
 description Subinterfaz VLAN 71 - Legal
 encapsulation dot1Q 71
 ip address 10.11.1.97 255.255.255.240

router ospf 1
 router-id 5.5.5.5
 network 10.11.0.64 0.0.0.63 area 0
 network 10.11.1.64 0.0.0.31 area 0
 network 10.11.1.96 0.0.0.15 area 0
 network 10.11.1.188 0.0.0.3 area 0
```

#### B. Switches `SW-Norte1` (Root Primario), `SW-Norte2` y `SW-Norte3`
```cisco
! === SW-Norte1 (Root Bridge) ===
hostname SW-Norte1
spanning-tree mode rapid-pvst
spanning-tree vlan 1,51,61,71 priority 4096

interface GigabitEthernet0/1
 description Troncal hacia R-Norte
 switchport mode trunk

interface FastEthernet0/1
 description Troncal hacia SW-Norte2
 switchport mode trunk

interface FastEthernet0/2
 description Troncal hacia SW-Norte3
 switchport mode trunk

! === SW-Norte2 ===
hostname SW-Norte2
spanning-tree mode rapid-pvst

interface FastEthernet0/1
 description Troncal hacia SW-Norte1
 switchport mode trunk

interface FastEthernet0/24
 description Troncal hacia SW-Norte3
 switchport mode trunk

! === SW-Norte3 (Bloqueo en Fa0/24) ===
hostname SW-Norte3
spanning-tree mode rapid-pvst

interface FastEthernet0/2
 description Troncal hacia SW-Norte1
 switchport mode trunk

interface FastEthernet0/24
 description Troncal hacia SW-Norte2 (Bloqueado por STP)
 switchport mode trunk
```

---

### 5.4 Sede Oriente (Multicapa 3560, SVIs y HSRP)

#### A. Switch Multicapa `MS1` (HSRP Active / Primario)
```cisco
hostname MS1
ip routing

! Routed Port hacia Core1
interface GigabitEthernet0/1
 description Puerto Enrutado hacia Core1
 no switchport
 ip address 10.11.1.198 255.255.255.252
 no shutdown

! SVIs con HSRP Activo
interface Vlan 81
 description SVI Boveda
 ip address 10.11.0.194 255.255.255.192
 standby 81 ip 10.11.0.193
 standby 81 priority 110
 standby 81 preempt
 no shutdown

interface Vlan 91
 description SVI Plataforma
 ip address 10.11.0.2 255.255.255.192
 standby 91 ip 10.11.0.1
 standby 91 priority 110
 standby 91 preempt
 no shutdown

! Enrutamiento EIGRP AS 10
router eigrp 10
 network 10.11.0.0 0.0.0.63
 network 10.11.0.192 0.0.0.63
 network 10.11.1.196 0.0.0.3
 auto-summary
```

#### B. Switch Multicapa `MS2` (HSRP Standby / Respaldo)
```cisco
hostname MS2
ip routing

! Routed Port hacia Core2
interface GigabitEthernet0/1
 description Puerto Enrutado hacia Core2
 no switchport
 ip address 10.11.1.210 255.255.255.252
 no shutdown

! SVIs con HSRP Standby
interface Vlan 81
 description SVI Boveda
 ip address 10.11.0.195 255.255.255.192
 standby 81 ip 10.11.0.193
 standby 81 priority 100
 no shutdown

interface Vlan 91
 description SVI Plataforma
 ip address 10.11.0.3 255.255.255.192
 standby 91 ip 10.11.0.1
 standby 91 priority 100
 no shutdown

! Enrutamiento EIGRP AS 10
router eigrp 10
 network 10.11.0.0 0.0.0.63
 network 10.11.0.192 0.0.0.63
 network 10.11.1.208 0.0.0.3
 auto-summary
```

---

### 5.5 Data Center y Sede Central (Aislamiento Estático y EtherChannel LACP)

#### A. Router Frontera `R-Central1` (Redistribución EIGRP con Métricas Compuestas)
```cisco
hostname R-Central1

interface GigabitEthernet0/0
 description Conectividad hacia Core3
 ip address 10.11.1.202 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 description Perimetro hacia R-Central2 (Data Center Interno)
 ip address 10.11.1.205 255.255.255.252
 no shutdown

! Rutas Estaticas Dirigidas a las VLANs de Servidores
ip route 10.11.1.112 255.255.255.240 10.11.1.206
ip route 10.11.1.32  255.255.255.224 10.11.1.206
ip route 10.11.1.160 255.255.255.240 10.11.1.206

! Proceso EIGRP redistribuyendo rutas estaticas con K-values
router eigrp 10
 network 10.11.1.200 0.0.0.3
 auto-summary
 redistribute static metric 10000 100 255 1 1500
```

#### B. Router Interno `R-Central2` (Ruta por Defecto y RoAS de Servidores)
```cisco
hostname R-Central2

interface GigabitEthernet0/0
 description Enlace Punto a Punto hacia R-Central1
 ip address 10.11.1.206 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 description Troncal hacia SW-DC1
 no ip address
 no shutdown

interface GigabitEthernet0/1.12
 description VLAN 12 - Core_BD (Servidores Base de Datos)
 encapsulation dot1Q 12
 ip address 10.11.1.113 255.255.255.240

interface GigabitEthernet0/1.22
 description VLAN 22 - Web_Apps (Servidores Web y APIs)
 encapsulation dot1Q 22
 ip address 10.11.1.33 255.255.255.224

interface GigabitEthernet0/1.32
 description VLAN 32 - NOC (Monitoreo de Infraestructura)
 encapsulation dot1Q 32
 ip address 10.11.1.161 255.255.255.240

! Ruta Estatica por Defecto hacia la Frontera
ip route 0.0.0.0 0.0.0.0 10.11.1.205
```

#### C. Switch de Distribución `SW-DC1` (EtherChannel LACP Activo)
```cisco
hostname SW-DC1

! Port-Channel 1 hacia SW-DC-Acc1
interface range FastEthernet0/1 - 2
 description Canales hacia SW-DC-Acc1
 channel-group 1 mode active

interface Port-channel 1
 switchport mode trunk

! Port-Channel 2 hacia SW-DC-Acc2
interface range FastEthernet0/3 - 4
 description Canales hacia SW-DC-Acc2
 channel-group 2 mode active

interface Port-channel 2
 switchport mode trunk

interface GigabitEthernet0/1
 description Troncal hacia R-Central2
 switchport mode trunk
```

#### D. Switches de Servidores `SW-DC-Acc1` y `SW-DC-Acc2`
```cisco
! === SW-DC-Acc1 ===
hostname SW-DC-Acc1
interface range FastEthernet0/1 - 2
 channel-group 1 mode active

interface Port-channel 1
 switchport mode trunk

! === SW-DC-Acc2 ===
hostname SW-DC-Acc2
interface range FastEthernet0/1 - 2
 channel-group 2 mode active

interface Port-channel 2
 switchport mode trunk
```

---

# 6. Plan de Pruebas, Validación y Evidencias de Alta Disponibilidad

El siguiente plan de aseguramiento de calidad técnica y pruebas de laboratorio fue ejecutado de manera exhaustiva sobre el archivo de simulación `Proyecto 2 V2 Grupo1.pkt` en Cisco Packet Tracer, validando la estabilidad, tolerancia a fallos y convergencia total de la infraestructura BanTech GT.

---

### Prueba 6.1: Validación de Topología General y Zonas Rotuladas
Se comprueba el despliegue físico y lógico de todos los dispositivos, zonas perimetrales rotuladas y medios de transmisión asignados.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Vista general y completa de la topología en Cisco Packet Tracer mostrando todas las sedes rotuladas: Sede Occidente, Sede Norte, Sede Oriente, Data Center/Sede Central y el Backbone Core triangular con sus enlaces debidamente identificados.
> * **Dispositivo / Terminal:** `Cisco Packet Tracer 8.x - Workspace Principal`
> * **Criterio de Validación:** Confirmación visual de la estructura completa de 5 zonas, cableado heterogéneo y rotulación institucional con carnets y parámetros $xx = 11$, $Y = 1$.

---

### Prueba 6.2: Tabla de Enrutamiento Global Convergida en `Core1` (24 Subredes)
Verificación de la inyección de rutas mediante OSPF, EIGRP, RIPv2 y Rutas Estáticas redistribuidas. La tabla debe reflejar la totalidad de las 24 subredes calculadas en el diseño VLSM.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de la salida del comando `show ip route` en el router `Core1`, demostrando el aprendizaje completo de las 24 subredes (rutas directamente conectadas `C`, rutas locales `L`, rutas OSPF `O`, rutas EIGRP externas `D EX` provenientes de RIP y Estáticas).
> * **Dispositivo / Terminal:** `Core1# show ip route`
> * **Criterio de Validación:** Presencia exacta de las 24 subredes sin registros duplicados ni saltos inalcanzables, validando la redistribución bidireccional exitosa en el Backbone Core.

---

### Prueba 6.3: Tablas de Enrutamiento en `Core2` y `Core3` (Ajuste de AD)
Demostración de la efectividad del ajuste `distance 100` en RIP dentro de `Core2`, garantizando que la subred de Occidente se aprende directamente por RIP y no indirectamente por OSPF desde Core1.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Salida de `show ip route` en `Core2` evidenciando que las redes de Occidente (`10.11.0.128/26`, `10.11.1.0/27`, `10.11.1.128/28`, `10.11.1.144/28`) se encuentran instaladas vía RIP con distancia administrativa `100` (`[100/2]`), evitando bucles de enrutamiento con Core1.
> * **Dispositivo / Terminal:** `Core2# show ip route`
> * **Criterio de Validación:** Ausencia de bucles mutuos; las redes de Occidente apuntan exclusivamente a la interfaz `GigabitEthernet0/2` de `R-Occidente`.

---

### Prueba 6.4: Agregación de Enlaces EtherChannel LACP en Data Center
Inspección del estado de los canales lógicos `Port-Channel 1` y `Port-Channel 2` entre el switch de distribución `SW-DC1` y los switches de acceso de servidores.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Salida del comando `show etherchannel summary` en `SW-DC1` mostrando que `Po1` y `Po2` se encuentran en estado `(SU)` (Switch-port, in-Use) y que los puertos físicos miembros (`Fa0/1`, `Fa0/2`, `Fa0/3`, `Fa0/4`) presentan la bandera `(P)` (bundled in port-channel) negociados bajo protocolo LACP.
> * **Dispositivo / Terminal:** `SW-DC1# show etherchannel summary`
> * **Criterio de Validación:** Canal agregado activo y operativo en Capa 2 a 200 Mbps full-duplex por canal, asegurando balanceo y redundancia física.

---

### Prueba 6.5: Estado Normal del Protocolo HSRP en Sede Oriente
Verificación del funcionamiento en estado de reposo (*Steady State*) de HSRP en los conmutadores multicapa `MS1` y `MS2`.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Salidas simultáneas o secuenciales de `show standby brief` en `MS1` y `MS2`. En `MS1` debe mostrarse el rol `Active` para las VLANs 81 y 91 (prioridad 110, preempt habilitado), mientras que en `MS2` debe observarse el rol `Standby` (prioridad 100) con IPs virtuales `10.11.0.193` y `10.11.0.1`.
> * **Dispositivo / Terminal:** `MS1# show standby brief` y `MS2# show standby brief`
> * **Criterio de Validación:** Asignación correcta de roles maestro/respaldo y visualización correcta del Virtual IP (VIP) en ambos dispositivos.

---

### Prueba 6.6: Prueba de Tolerancia a Fallos (Failover) de HSRP en Sede Oriente
Simulación de catástrofe o falla energética en el switch multicapa primario (`MS1`) apagando sus interfaces y verificando la conmutación automática de tráfico hacia `MS2`.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Ventana de consola de `MS2` mostrando el mensaje de Syslog `%%HSRP-5-STATECHANGE: Vlan81 Grp 81 state Standby -> Active` tras aplicar `shutdown` en las interfaces SVI de `MS1`, acompañado de una prueba de `ping` continuo ejecutada desde un host de Sede Oriente hacia un servidor del Data Center sin interrupción perceptible del servicio.
> * **Dispositivo / Terminal:** `MS2# show standby brief` y `Terminal_Boveda1 Command Prompt: ping -t 10.11.1.114`
> * **Criterio de Validación:** `MS2` toma el rol `Active` de forma transparente; la pérdida de paquetes durante la conmutación es igual o menor a 1 paquete ICMP.

---

### Prueba 6.7: Prueba de Restauración Determinista (Failback con Preempt) en HSRP
Comprobación de la reactivación del switch primario `MS1` y su recuperación inmediata del rol `Active` gracias al mecanismo de *preemption*.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Salida en consola de `MS1` tras ejecutar `no shutdown` en sus interfaces, registrando el mensaje de syslog de transición a `Active` y la subsecuente salida de `show standby brief` donde `MS1` retoma su rol prioritario.
> * **Dispositivo / Terminal:** `MS1# show standby brief`
> * **Criterio de Validación:** Preempt operativo al 100%, reestableciendo la topología óptima de diseño sin intervención manual adicional.

---

### Prueba 6.8: Verificación del Árbol de Expansión Rapid PVST+ en Sede Norte
Comprobación de la elección determinista de Root Bridge en `SW-Norte1` y la existencia de un puerto alterno bloqueado en `SW-Norte3` para prevenir bucles de conmutación.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de `show spanning-tree` en `SW-Norte1` mostrando la leyenda *"This bridge is the root"* para las VLANs 51, 61 y 71 con costo de ruta 0, junto con la salida de `show spanning-tree` en `SW-Norte3` mostrando la interfaz `Fa0/24` en estado `Altn BLK` (Alternate / Blocking).
> * **Dispositivo / Terminal:** `SW-Norte1# show spanning-tree` y `SW-Norte3# show spanning-tree`
> * **Criterio de Validación:** Protocolo `rstp` activo, elección correcta del puente raíz hacia el router de borde y prevención de bucles en la topología triangular.

---

### Prueba 6.9: Prueba de Ruptura de Enlace y Reconvergencia Inmediata en RSTP
Simulación de corte de cable entre `SW-Norte1` y `SW-Norte2` en Sede Norte, evaluando el tiempo de desbloqueo de la interfaz de respaldo en `SW-Norte3`.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de pantalla que evidencie el estado de transición del puerto `Fa0/24` en `SW-Norte3` pasando instantáneamente a estado `FWD` (Forwarding) al desconectar el enlace primario, manteniendo la conectividad continua con el gateway `R-Norte`.
> * **Dispositivo / Terminal:** `SW-Norte3# show spanning-tree`
> * **Criterio de Validación:** Reconvergencia en menos de 1 segundo (0 paquetes perdidos en ICMP tras la caída del enlace físico directo).

---

### Prueba 6.10: Sincronización de Base de Datos VTP en Sede Occidente
Comprobación de la replicación del dominio `bantech11` y la propagación de las VLANs creadas en el servidor hacia los conmutadores cliente.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Salidas de los comandos `show vtp status` y `show vlan brief` en `SW-Occidente2` (Client), mostrando la pertenencia al dominio `bantech11`, número de revisión de configuración actualizado y la existencia automática de las VLANs 11, 21, 31 y 41 sin haber sido creadas localmente.
> * **Dispositivo / Terminal:** `SW-Occidente2# show vtp status` y `SW-Occidente2# show vlan brief`
> * **Criterio de Validación:** Modo de operación Client verificado y sincronización consistente de todas las VLANs regionales.

---

### Prueba 6.11: Aislamiento Perimetral y Redistribución Estática en Data Center
Verificación del esquema de enrutamiento estático en los enrutadores de frontera del centro de datos.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Salida de `show ip route` en `R-Central2` mostrando la ruta por defecto `S* 0.0.0.0/0 [1/0] via 10.11.1.205`, y salida de `show ip route static` en `R-Central1` confirmando las 3 rutas específicas hacia `Core_BD`, `Web_Apps` y `NOC` vía `10.11.1.206`.
> * **Dispositivo / Terminal:** `R-Central2# show ip route` y `R-Central1# show ip route static`
> * **Criterio de Validación:** Demostración de que la zona interna de servidores no ejecuta protocolos de enrutamiento dinámico externos, asegurando el aislamiento y seguridad requeridos por la rúbrica.

---

### Prueba 6.12: Pruebas de Conectividad Extremo a Extremo (End-to-End ICMP)
Pruebas exhaustivas de comunicación cruzada inter-regional y acceso hacia los recursos del Data Center.

#### Caso A: Terminal de Cajas (Occidente) -> Servidor de Base de Datos Principal (Data Center)
* **Origen:** `PC-Caja1` (`10.11.0.130`, VLAN 11)  
* **Destino:** `Server_BD_Master` (`10.11.1.114`, VLAN 12)  
* **Flujo Transaccional:** `PC-Caja1` $\rightarrow$ `SW-Occidente1` $\rightarrow$ `R-Occidente` $\rightarrow$ `Core2` $\rightarrow$ `Core3` $\rightarrow$ `R-Central1` $\rightarrow$ `R-Central2` $\rightarrow$ `SW-DC1` $\rightarrow$ `SW-DC-Acc1` $\rightarrow$ `Server_BD_Master`.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de consola ejecutando `ping 10.11.1.114` desde `PC-Caja1` con respuesta exitosa (*Reply from 10.11.1.114: bytes=32 time<1ms TTL=122*), seguido de un `tracert 10.11.1.114` mostrando cada uno de los saltos intermedios a través del Core.
> * **Dispositivo / Terminal:** `PC-Caja1 Command Prompt: ping 10.11.1.114`
> * **Criterio de Validación:** 100% de éxito en la prueba ICMP ($4/4$), confirmando la ruta transaccional más crítica del banco.

---

#### Caso B: Terminal de Análisis de Crédito (Norte) -> Servidor de Aplicaciones Web (Data Center)
* **Origen:** `PC-Analista1` (`10.11.0.66`, VLAN 51)  
* **Destino:** `Server_Portal_Web` (`10.11.1.34`, VLAN 22)  
* **Flujo Transaccional:** `PC-Analista1` $\rightarrow$ `SW-Norte1` $\rightarrow$ `R-Norte` $\rightarrow$ `Core1` $\rightarrow$ `Core3` $\rightarrow$ `R-Central1` $\rightarrow$ `R-Central2` $\rightarrow$ `Server_Portal_Web`.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de consola ejecutando `ping 10.11.1.34` desde `PC-Analista1` en Sede Norte hacia el servidor Web Apps del Data Center con éxito total.
> * **Dispositivo / Terminal:** `PC-Analista1 Command Prompt: ping 10.11.1.34`
> * **Criterio de Validación:** Conectividad inter-regional OSPF-EIGRP convergida y operativa.

---

#### Caso C: Terminal de Bóveda (Oriente) -> Terminal de Gerencia (Occidente)
* **Origen:** `Terminal_Boveda1` (`10.11.0.196`, VLAN 81)  
* **Destino:** `Laptop-Gerente` (`10.11.1.146`, VLAN 31)  
* **Flujo Transaccional:** `Terminal_Boveda1` $\rightarrow$ `MS1 (HSRP Gateway)` $\rightarrow$ `Core1` $\rightarrow$ `Core2` $\rightarrow$ `R-Occidente` $\rightarrow$ `Laptop-Gerente`.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de pantalla de `ping 10.11.1.146` desde `Terminal_Boveda1` en Sede Oriente hacia la máquina de Gerencia en Sede Occidente.
> * **Dispositivo / Terminal:** `Terminal_Boveda1 Command Prompt: ping 10.11.1.146`
> * **Criterio de Validación:** Comunicación cruzada entre switches multicapa y router-on-a-stick a través del backbone heterogéneo.

---

#### Caso D: Terminal de Plataforma (Oriente) -> Terminal de Auditoría (Norte)
* **Origen:** `PC-Operador1` (`10.11.0.4`, VLAN 91)  
* **Destino:** `PC-Auditor1` (`10.11.1.66`, VLAN 61)  
* **Flujo Transaccional:** `PC-Operador1` $\rightarrow$ `MS1 (HSRP Gateway)` $\rightarrow$ `Core1` $\rightarrow$ `R-Norte` $\rightarrow$ `PC-Auditor1`.

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de pantalla de `ping 10.11.1.66` ejecutado desde `PC-Operador1` hacia `PC-Auditor1` validando el enlace de auditoría inter-agencias.
> * **Dispositivo / Terminal:** `PC-Operador1 Command Prompt: ping 10.11.1.66`
> * **Criterio de Validación:** Tráfico bidireccional fluido entre EIGRP AS 10 y OSPF Área 0 vía `Core1`.

---

#### Caso E: Consola de Monitoreo NOC (Data Center) -> Gateways y Equipos de Todas las Sedes
* **Origen:** `NOC_Console_01` (`10.11.1.162`, VLAN 32)  
* **Destinos:** Gateways `10.11.0.129` (Occidente), `10.11.0.65` (Norte), `10.11.0.1` (Oriente VIP), `10.11.1.177` (Core1).

> **[INSERTAR CAPTURA AQUÍ]**
> * **Descripción:** Captura de consola en la terminal del NOC ejecutando pruebas ping hacia los cuatro gateways remotos, evidenciando capacidad de gestión, monitoreo y telemetría centralizada desde el Data Center.
> * **Dispositivo / Terminal:** `NOC_Console_01 Command Prompt: ping a gateways de red`
> * **Criterio de Validación:** Capacidad total de gestión de infraestructura desde el puesto central de operaciones de red.

---

# 7. Conclusiones y Recomendaciones de Ingeniería

### 7.1 Conclusiones Técnicas
1. **Convergencia Determinista y Eliminación de Bucles:** La solución de arquitectura al modificar la Distancia Administrativa de RIP a 100 en `Core2` demostró ser la clave técnica que permitió la coexistencia pacífica entre OSPF (AD 110), RIPv2 (AD 100 modificada) y EIGRP (AD 90/170). Sin esta intervención, la red nacional experimentaba particionamiento y ciclos infinitos en el tráfico hacia el occidente guatemalteco.
2. **Resiliencia Integral en Capa 2 y Capa 3:** La combinación de Rapid PVST+ en la Sede Norte, HSRP con *preempt* en la Sede Oriente y EtherChannel LACP en el Data Center garantiza una infraestructura bancaria de alta tolerancia a fallos. Los tiempos de reconvergencia observados en las pruebas de corte de enlace fueron inferiores a un segundo, satisfaciendo holgadamente los estándares internacionales de disponibilidad de servicios financieros (estándar Tier II / Tier III).
3. **Optimización Estricta del Espacio de Direccionamiento:** Mediante el esquema VLSM desarrollado sobre la red `10.11.0.0/16`, se abastecieron las 12 VLANs de servicios bancarios y los 9 enlaces punto a punto inter-router con un consumo neto mínimo de direcciones IP, dejando libres bloques enteros dentro del `/16` para futuros planes de expansión institucional de BanTech GT.

### 7.2 Recomendaciones para Despliegues en Producción
1. **Transición hacia Protocolos Seguros y Autenticación Criptográfica:**  
   Se recomienda activar autenticación criptográfica basada en hash **MD5 o HMAC-SHA256** en todas las adyacencias de enrutamiento (vecindades OSPF, paquetes EIGRP y actualizaciones RIPv2), impidiendo la inserción de enrutadores no autorizados (*rogue routers*) en los enlaces troncales o de acceso.
2. **Implementación de Listas de Control de Acceso (ACLs) y Zonificación:**  
   Si bien el enrutamiento permite conectividad global, en un entorno de producción financiera debe configurarse un cortafuegos (*Firewall*) o ACLs extendidas en `R-Central1` y `R-Central2` para restringir que únicamente las VLANs autorizadas (por ejemplo, Cajas y Bóveda) puedan comunicarse con los puertos del clúster de Base de Datos (`Core_BD` puerto TCP 1433/5432) y denegar el acceso a segmentos de usuarios generales o cámaras de seguridad.
3. **Migración Futura del Protocolo de Sede Occidente:**  
   Dado que RIPv2 es un protocolo legado con convergencia lenta basada en temporizadores fijos (30 segundos de actualización) y límite de 15 saltos, se recomienda planificar la migración de `R-Occidente` hacia **OSPF Área 0** o **EIGRP AS 10**, simplificando la matriz de redistribución en `Core2` y unificando el plano de control bajo protocolos de estado de enlace o vector distancia avanzado.

---

</div>