# MAC-Flooding-Attack-Lab# Ataque MAC Flooding

![Python](https://img.shields.io/badge/Python-3.x-blue)
![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-red)
![Environment](https://img.shields.io/badge/Environment-GNS3%20%7C%20IOSvL2-orange)
![Attack](https://img.shields.io/badge/Attack-MAC%20Flooding-purple)
![Mitigation](https://img.shields.io/badge/Mitigation-Port%20Security-darkgreen)
![Use](https://img.shields.io/badge/Use-Controlled%20Lab-yellow)

## Información del proyecto

| Dato                  | Información                                        |
| --------------------- | -------------------------------------------------- |
| Autor                 | Darwing                                            |
| Matrícula             | 2024-2690                                          |
| Docente               | Jonathan Rondon                                    |
| Repositorio           | https://github.com/TuUsuario/MAC-Flooding-Attack   |
| Video demostrativo    | https://youtu.be/x3o-aAg5ZMc          |
| Documentación técnica | docs/documentacion-tecnica-profesional.pdf         |
| Red de laboratorio    | 20.24.26.0/24                                      |

---

## Aviso de uso responsable

Este proyecto fue desarrollado únicamente con fines educativos, académicos y de laboratorio controlado. Las pruebas deben ejecutarse solamente en entornos propios o autorizados como GNS3, EVE-NG, PNETLab o laboratorios internos. No debe utilizarse en redes públicas, empresariales o de terceros sin autorización explícita.

---

## Objetivo del laboratorio

Demostrar el funcionamiento de un ataque **MAC Flooding**, donde el atacante envía miles de tramas Ethernet con direcciones MAC origen falsas y aleatorias para saturar la tabla CAM del switch. Cuando la tabla se llena, el switch entra en modo fail-open y comienza a reenviar tramas por todos los puertos como si fuera un hub, exponiendo el tráfico de todos los dispositivos.

Después de demostrar el impacto, se aplica una contramedida basada en **Port Security** para bloquear el ataque.

---

## Archivos del repositorio

| Archivo                                        | Descripción                                                        |
| ---------------------------------------------- | ------------------------------------------------------------------ |
| `mac_flooding.py`                              | Script principal para ejecutar el ataque MAC Flooding.             |
| `mitigacion-mac-flooding.md`                   | Documento con las contramedidas aplicadas.                         |
| `README.md`                                    | Guía principal del laboratorio.                                    |
| `docs/documentacion-tecnica-profesional.pdf`   | Documentación técnica profesional detallada del laboratorio.       |
| `images/`                                      | Capturas de pantalla de evidencia del laboratorio.                 |

---

## Topología del laboratorio

```
         R1 (20.24.26.91)
         f0/0
          |
         Gi0/2
        [SW-1]
       /       \
   Gi0/0      Gi0/1
   Kali        PC1
(20.24.26.90)  (20.24.26.92)
```

| Dispositivo | Rol               | Interfaz | Dirección IP    | Descripción                         |
| ----------- | ----------------- | -------- | --------------- | ----------------------------------- |
| R1          | Gateway           | F0/0     | 20.24.26.91/24  | Router principal de la red          |
| SW-1        | Switch capa 2     | Gi0/0~2  | N/A             | Switch objetivo del ataque          |
| Kali Linux  | Atacante          | eth0     | 20.24.26.90/24  | Ejecuta el flood de tramas Ethernet |
| PC1         | Víctima / Cliente | e0       | 20.24.26.92/24  | Equipo cuyo tráfico es expuesto     |

---

## Requisitos previos

- GNS3 o entorno de virtualización equivalente.
- Switch Cisco IOSvL2.
- Kali Linux con Python 3 y Scapy instalado.
- Permisos de superusuario (`sudo`).
- Conectividad de capa 2 entre Kali y el switch.

```bash
pip install scapy
```

---

## Parámetros del script

| Parámetro          | Descripción                                              | Ejemplo    |
| ------------------ | -------------------------------------------------------- | ---------- |
| `-i` / `--iface`   | Interfaz de red usada para enviar las tramas             | `-i eth0`  |
| `-c` / `--count`   | Cantidad de tramas a enviar (0 = infinito)               | `-c 5000`  |
| `-d` / `--delay`   | Tiempo entre tramas en segundos                          | `-d 0.001` |
| `-v` / `--verbose` | Muestra cada trama enviada en pantalla                   | `-v`       |

---

## Flujo del laboratorio

### 1. Estado inicial del switch

Antes de iniciar el ataque verificar el conteo de entradas MAC:

```
SW-1# show mac address-table count
SW-1# show mac address-table
```

### 2. Ejecución del ataque

```bash
sudo python3 mac_flooding.py -i eth0 -c 5000 -v
```

Durante la ejecución:

```
╔══════════════════════════════════════════════╗
║         MAC Flooding Attack                  ║
║  Interfaz : eth0                            ║
║  Tramas   : 5000                            ║
╚══════════════════════════════════════════════╝
[*] Saturando tabla CAM... (Ctrl+C para detener)

[     1] SRC=02:a1:b2:c3:d4:e5 → DST=02:f1:e2:d3:c4:b5
[     2] SRC=02:11:22:33:44:55 → DST=02:aa:bb:cc:dd:ee
...
[+] Total tramas enviadas: 5000
```

Para detener: `Ctrl + C`

### 3. Verificar el incremento en la tabla CAM

```
SW-1# show mac address-table count
```

Se observará un aumento masivo de entradas dinámicas en la tabla MAC.

### 4. Aplicar la contramedida — Port Security

```
enable
configure terminal
interface gigabitEthernet0/0
 description HACIA-KALI-ATACANTE
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 exit
interface gigabitEthernet0/1
 description HACIA-PC1-CLIENTE
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 exit
end
write memory
```

### 5. Verificar la violación de seguridad

Relanzar el ataque con Port Security activo:

```bash
sudo python3 mac_flooding.py -i eth0 -c 1000
```

El switch detectará múltiples MACs desde Gi0/0 y apagará el puerto:

```
SW-1# show logging
SW-1# show interfaces GigabitEthernet0/0
SW-1# show port-security interface GigabitEthernet0/0
```

El puerto quedará en estado **err-disabled**, bloqueando el ataque.

---

## Comandos de verificación completos

```
SW-1# show mac address-table count
SW-1# show mac address-table
SW-1# show port-security
SW-1# show port-security interface GigabitEthernet0/0
SW-1# show interfaces status
SW-1# show logging
```

---

## Video demostrativo

[Ver video del laboratorio en YouTube](https://www.youtube.com/watch?v=PENDIENTE)

---

## Conclusión

El ataque MAC Flooding evidenció que un switch puede aprender miles de direcciones MAC falsas en un corto período, alterando su comportamiento normal de conmutación. Cuando la tabla CAM se satura, el switch reenvía el tráfico por todos los puertos, exponiendo las comunicaciones de todos los dispositivos conectados.

La contramedida mediante **Port Security** resultó efectiva: al detectar múltiples MACs desde el puerto del atacante, el switch activó una violación de seguridad y colocó la interfaz en estado err-disabled, bloqueando el ataque y protegiendo el tráfico legítimo de la red.

---

## Autor

**Darwing**  
Matrícula: **2024-2690**  
Repositorio: https://github.com/TuUsuario/MAC-Flooding-Attack
