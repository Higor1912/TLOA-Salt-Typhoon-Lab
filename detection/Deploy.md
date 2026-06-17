# TLOA Lab — Guia de Deploy
## Sysmon Config + Wazuh Rules (Salt Typhoon Coverage)

---

## Ordem de Execução

```
1. Aplicar Sysmon (DC01 + WS01)
2. Verificar telemetria chegando no Wazuh
3. Deploy das regras customizadas
4. Validar com testes controlados
5. Reexecutar o lab Salt Typhoon
```

---

## Passo 1 — Aplicar Sysmon Config (DC01 e WS01)

Execute em cada máquina Windows como Administrador:

```powershell
# Se Sysmon já instalado — atualizar config
Sysmon64.exe -c sysmon-tloa.xml

# Se Sysmon não instalado — instalar com config
Sysmon64.exe -accepteula -i sysmon-tloa.xml

# Confirmar que o serviço está rodando
Get-Service Sysmon64
```

Verificar que os eventos estão sendo gerados:

```powershell
# Teste rápido — deve gerar EID 1 e EID 3
powershell.exe -NonInteractive -Command "Get-ADDomain"

# Ver últimos eventos Sysmon
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20 |
    Select-Object Id, TimeCreated, Message |
    Format-Table -Wrap
```

---

## Passo 2 — Verificar Telemetria no Wazuh

No servidor Wazuh, confirmar que os eventos estão chegando:

```bash
# Ver eventos recentes do agente (substitua NOME_DO_AGENTE)
tail -f /var/ossec/logs/archives/archives.json | grep -i "sysmon"

# Buscar EID 1 (process create) dos últimos 5 minutos
grep '"id":"1"' /var/ossec/logs/archives/archives.json | tail -20
```

Se não aparecer nada, verificar o agente:

```bash
# Status do agente no manager
/var/ossec/bin/agent_control -l

# Logs do agente (no Windows — pasta de instalação do Wazuh)
type "C:\Program Files (x86)\ossec-agent\ossec.log" | findstr ERROR
```

---

## Passo 3 — Deploy das Regras Customizadas

```bash
# Copiar arquivo de regras para o diretório correto
cp tloa_salt_typhoon_rules.xml /var/ossec/etc/rules/

# Verificar sintaxe antes de restartar
/var/ossec/bin/wazuh-logtest

# Restartar o Wazuh Manager para carregar as novas regras
systemctl restart wazuh-manager

# Confirmar que as regras foram carregadas (procurar pelo ID 100001)
grep "100001" /var/ossec/logs/ossec.log
```

---

## Passo 4 — Validação das Regras (Testes Controlados)

Execute cada teste na WS01 e verifique se o alerta correspondente aparece no Wazuh Dashboard.

### Teste 1 — Regra 100002 (nltest recon)
```powershell
nltest /dclist:tloa.local
```
**Esperado:** Alerta rule_id 100002, level 10, grupo `recon,ad_enum`

### Teste 2 — Regra 100011 (PowerShell EncodedCommand)
```powershell
$cmd = "Get-Date"
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
powershell.exe -EncodedCommand $encoded
```
**Esperado:** Alerta rule_id 100011, level 12, grupo `lotl,obfuscation`

### Teste 3 — Regra 100001 (PowerShell network connection)
```powershell
$conn = New-Object System.Net.Sockets.TcpClient
$conn.Connect("192.168.204.10", 445)
$conn.Close()
```
**Esperado:** Alerta rule_id 100001, level 10, grupo `recon,network_connection`

### Teste 4 — Regra 100031 (Scheduled task + EncodedCommand)
```powershell
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("Get-Date"))
schtasks /Create /TN "TestTask" /TR "powershell.exe -EncodedCommand $encoded" /SC ONCE /ST 23:59 /F
# Limpar depois do teste
schtasks /Delete /TN "TestTask" /F
```
**Esperado:** Alertas rule_id 100030 + 100031, level 14

### Teste 5 — Cadeia de Correlação (Regra 100101)
```powershell
# Simular logon RDP de service account (gera EID 4624 LogonType 10)
# — fazer login via RDP com svc_backup de host Kali —

# Depois, criar scheduled task com EncodedCommand
$encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("whoami"))
schtasks /Create /TN "\Microsoft\Windows\Maintenance\TestCorr" /TR "powershell.exe -EncodedCommand $encoded" /SC DAILY /ST 03:00 /F
```
**Esperado:** Alerta rule_id 100101, level 15, grupo `campaign,salt_typhoon,critical`

---

## Passo 5 — Reexecutar o Lab Salt Typhoon

Com as regras ativas, executar as 6 fases do lab novamente e documentar:

| Fase | Gap Original | Regra Cobrindo | Alerta Gerado? |
|------|-------------|----------------|----------------|
| 1 — Recon | Port scan TcpClient | 100001 | ? |
| 1 — Recon | nltest dclist | 100002/100003 | ? |
| 2 — LOTL | Get-ADUser | 100012 | ? |
| 2 — LOTL | PowerShell -Encoded | 100011 | ? |
| 3 — Creds | RDP svc_account | 100020 | ? |
| 4 — Persist | Scheduled task | 100030/100031 | ? |
| 5 — Enum | Get-ADGroupMember | 100040 | ? |
| 6 — Hunting | Cadeia completa | 100101 | ? |

Preencher a tabela e incluir no relatório como seção "Before/After Detection Coverage".

---

## Troubleshooting Rápido

**Regra não dispara:**
- Verificar se o `if_sid` pai está sendo gerado primeiro
- Conferir se o campo `win.eventdata.X` corresponde ao que chega no evento (usar wazuh-logtest)
- EID do Sysmon no Wazuh: EID 1 = sid 61612, EID 3 = sid 61603, EID 10 = sid 61632, EID 11 = sid 61635, EID 13 = sid 61640

**Testar uma regra manualmente:**
```bash
# No servidor Wazuh — simular um evento e ver qual regra dispara
/var/ossec/bin/wazuh-logtest
# Colar o JSON do evento e pressionar Enter
```

**Ver alertas em tempo real:**
```bash
tail -f /var/ossec/logs/alerts/alerts.json | python3 -m json.tool
```