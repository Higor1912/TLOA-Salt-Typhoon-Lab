# TLOA-Salt-Typhoon-Lab

> **[PT-BR]** Emulação de adversário e detecção de TTPs associadas ao Salt Typhoon em ambiente Active Directory monitorado por Wazuh e Sysmon.
>
> **[EN]** Adversary emulation and detection of Salt Typhoon-associated TTPs in an Active Directory environment monitored by Wazuh and Sysmon.

---

## PT-BR

### Sobre o Lab

Este repositório documenta a simulação das principais TTPs do **Salt Typhoon** (TAG-22 / GhostEmperor) em um home lab Active Directory, com foco em visibilidade defensiva, identificação de gaps de detecção e desenvolvimento de regras customizadas para Wazuh.

O lab é parte do **TLOA Framework** — metodologia de auditoria ofensiva orientada por ameaças reais.

📄 **Post completo no Medium:** [Salt Typhoon no Home Lab: como simular espionagem persistente e (finalmente) detectá-la](#) *(link após publicação)*

---

### TTPs Cobertas

| Fase | TTP | Técnica | Regra TLOA | Level |
|------|-----|---------|-----------|-------|
| 1 | T1018 | Remote System Discovery (nltest) | 110003 | 12 |
| 1 | T1046 | Network Service Scanning (TcpClient) | 110001 | 10 |
| 2 | T1059.001 | PowerShell — AD recon LOTL | 92103 | 10 |
| 3 | T1078 | Valid Accounts — credenciais explícitas | 110021 | 10 |
| 3 | T1078 | Valid Accounts — privilégio especial | 110022 | 12 |
| 4 | T1053.005 | Scheduled Task — EncodedCommand | 110031 | 14 |
| 5 | T1087.002 | Account Discovery — SPN enum | 92103 | 10 |

---

### Ambiente

```
VMnet1 — 192.168.204.0/24
├── DC01   Windows Server 2025 Datacenter  192.168.204.132  Domain Controller
├── WS01   Windows 10 Pro                  192.168.204.130  Endpoint / Alvo
├── Kali   Kali Linux 2026.1               192.168.204.128  Atacante
└── Wazuh  Ubuntu 64-bit (Wazuh 4.7.5)    192.168.204.129  SIEM / HIDS
```

**Domínio:** `tloa.local`
**Sysmon:** v15.20
**Wazuh:** 4.7.5

---

### Estrutura do Repositório

```
TLOA-Salt-Typhoon-Lab/
├── README.md
├── detection/
│   ├── tloa_salt_typhoon_rules.xml   # 25 regras Wazuh (range 110001–110150)
│   ├── sysmon-tloa.xml               # Sysmon config customizado para LOTL/AD
│   └── DEPLOY.md                     # Guia de instalação passo a passo
├── evidence/
│   └── screenshots/                  # Prints organizados por fase do lab
└── report/
    └── salt-typhoon-lab.html         # Relatório técnico completo (TLOA Framework)
```

---

### Como Usar

**1. Aplicar o Sysmon config (DC01 e WS01):**
```powershell
Sysmon64.exe -accepteula -i detection\sysmon-tloa.xml
```

**2. Adicionar canal Sysmon no agente Wazuh (cada host Windows):**
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**3. Deploy das regras no Wazuh Manager:**
```bash
sudo cp detection/tloa_salt_typhoon_rules.xml /var/ossec/etc/rules/
sudo systemctl restart wazuh-manager
```

Consulte `detection/DEPLOY.md` para o guia completo com validação e troubleshooting.

---

### Principais Achados

- **Gap fundamental:** Canal `Microsoft-Windows-Sysmon/Operational` ausente no `ossec.conf` do agente tornava EID 1, 3, 10, 11 e 13 invisíveis para o Wazuh — independentemente de qualquer regra configurada.

- **Hierarquia de SIDs não documentada:** Regras customizadas para EIDs do canal Security sem cobertura padrão (ex: 4648, 4702) devem usar `if_sid>60103`. A cadeia `60000 → 60001 → 60103` não está clara na documentação oficial do Wazuh 4.7.5.

- **Windows Server 2025 como defensor:** RC4 deprecado, SMB signing obrigatório e LDAP signing enforced bloquearam técnicas legadas (Kerberoasting clássico, PTH via SMB) — documentado como postura defensiva positiva.

- **Resultado final:** 21 alertas das regras TLOA em 24h, 7 de level 12+, cobrindo todas as 6 fases simuladas.

---

### Referências

- CISA Advisory AA24-038A — Salt Typhoon Telecom Compromise (fev/2024)
- MITRE ATT&CK — [Salt Typhoon (G1045)](https://attack.mitre.org/groups/G1045/)
- Microsoft MSTIC — Salt Typhoon Threat Actor Tracking
- [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)

---

## EN

### About

This repository documents the simulation of **Salt Typhoon** (TAG-22 / GhostEmperor) TTPs in a home lab Active Directory environment, focusing on defensive visibility, detection gap analysis, and custom Wazuh rule development.

Part of the **TLOA Framework** — a threat-led offensive audit methodology.

📄 **Full write-up on Medium:** https://medium.com/@higorsilva1816/salt-typhoon-no-home-lab-como-simular-espionagem-persistente-e-finalmente-detect%C3%A1-la-8376593522a8
---

### TTPs Covered

| Phase | TTP | Technique | TLOA Rule | Level |
|-------|-----|-----------|-----------|-------|
| 1 | T1018 | Remote System Discovery (nltest) | 110003 | 12 |
| 1 | T1046 | Network Service Scanning (TcpClient) | 110001 | 10 |
| 2 | T1059.001 | PowerShell — AD recon LOTL | 92103 | 10 |
| 3 | T1078 | Valid Accounts — explicit credentials | 110021 | 10 |
| 3 | T1078 | Valid Accounts — special privilege | 110022 | 12 |
| 4 | T1053.005 | Scheduled Task — EncodedCommand | 110031 | 14 |
| 5 | T1087.002 | Account Discovery — SPN enum | 92103 | 10 |

---

### Lab Environment

```
VMnet1 — 192.168.204.0/24
├── DC01   Windows Server 2025 Datacenter  192.168.204.132  Domain Controller
├── WS01   Windows 10 Pro                  192.168.204.130  Endpoint / Target
├── Kali   Kali Linux 2026.1               192.168.204.128  Attacker
└── Wazuh  Ubuntu 64-bit (Wazuh 4.7.5)    192.168.204.129  SIEM / HIDS
```

**Domain:** `tloa.local` | **Sysmon:** v15.20 | **Wazuh:** 4.7.5

---

### Key Findings

- **Root cause of zero alerts:** Missing `Microsoft-Windows-Sysmon/Operational` channel in the Wazuh agent `ossec.conf` made EID 1, 3, 10, 11 and 13 completely invisible to the SIEM — regardless of any configured rules.

- **Undocumented Wazuh SID hierarchy:** Custom rules for Security channel EIDs without default coverage (e.g. 4648, 4702) must use `if_sid>60103`. The chain `60000 → 60001 → 60103` is not clearly documented in the official Wazuh 4.7.5 docs.

- **Windows Server 2025 as a defensive ally:** Deprecated RC4, mandatory SMB signing and LDAP signing enforcement blocked legacy techniques — documented as positive security posture findings.

- **Final result:** 21 TLOA rule alerts in 24h, 7 at level 12+, covering all 6 simulated phases.

---

### Quick Start

```bash
# Wazuh Manager — deploy rules
sudo cp detection/tloa_salt_typhoon_rules.xml /var/ossec/etc/rules/
sudo systemctl restart wazuh-manager
```

```powershell
# Windows hosts — apply Sysmon config
Sysmon64.exe -c detection\sysmon-tloa.xml
```

See `detection/DEPLOY.md` for the full step-by-step guide.

---

### References

- CISA Advisory AA24-038A — Salt Typhoon Telecom Compromise (Feb/2024)
- MITRE ATT&CK — [Salt Typhoon (G1045)](https://attack.mitre.org/groups/G1045/)
- Microsoft MSTIC — Salt Typhoon Threat Actor Tracking
- [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)

---

*Part of the [TLOA Framework](https://github.com/Higor1912) — Threat-Led Offensive Audit.*
