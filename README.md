# Integrazione Suricata con Wazuh - Guida Completa

## 📋 Panoramica

Questa guida documenta come abbiamo integrato Suricata IDS con Wazuh SIEM per il monitoraggio di sicurezza in tempo reale. L'integrazione permette a Wazuh di analizzare automaticamente i log JSON di Suricata e generare alert per eventi di sicurezza.

## 🎯 Obiettivo

Configurare Wazuh (running in Docker) per leggere e processare automaticamente i log di Suricata (`/var/log/suricata/eve.json`) dall'host in tempo reale.

## ⚙️ Prerequisiti

- Wazuh deployment con Docker Compose (single-node)
- Suricata installato e configurato sull'host
- File di log Suricata: `/var/log/suricata/eve.json`
- Accesso root/sudo

## 🔧 Passi di Implementazione

### 1. Backup della Configurazione Esistente

```bash
# Backup del docker-compose.yml
sudo cp /opt/wazuh-docker/single-node/docker-compose.yml /opt/wazuh-docker/single-node/docker-compose.yml.backup

# Backup della configurazione Wazuh
sudo cp /opt/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf /opt/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf.backup
```

### 2. Modifica Docker Compose

Aggiungere il volume mount per Suricata nel file `docker-compose.yml`:

```yaml
services:
  wazuh.manager:
    # ... altre configurazioni ...
    volumes:
      # ... volumi esistenti ...
      - /var/log/suricata:/var/log/suricata:ro  # ← AGGIUNTO
      # ... altri volumi ...
```

**Comando per aggiungere automaticamente:**
```bash
sudo sed -i '/filebeat_var:\/var\/lib\/filebeat/a\      - /var/log/suricata:/var/log/suricata:ro' /opt/wazuh-docker/single-node/docker-compose.yml
```

### 3. Configurazione Wazuh Manager

Aggiungere la configurazione per leggere i log Suricata nel file `config/wazuh_cluster/wazuh_manager.conf`:

```xml
<!-- Suricata Logs -->
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

**Comando per aggiungere automaticamente:**
```bash
sudo sed -i '248a\
\
  <!-- Suricata Logs -->\
  <localfile>\
    <log_format>json</log_format>\
    <location>/var/log/suricata/eve.json</location>\
  </localfile>' /opt/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf
```

### 4. Riavvio dei Container

```bash
# Stop dei container
cd /opt/wazuh-docker/single-node
sudo docker compose down

# Riavvio con nuova configurazione
sudo docker compose up -d
```

### 5. Verifica dell'Integrazione

```bash
# Verificare che Wazuh stia leggendo i log Suricata
sudo docker exec single-node-wazuh.manager-1 grep -i "suricata\|eve.json" /var/ossec/logs/ossec.log

# Verificare l'accesso ai file Suricata
sudo docker exec single-node-wazuh.manager-1 ls -la /var/log/suricata/

# Verificare lo stato dei servizi Wazuh
sudo docker exec single-node-wazuh.manager-1 /var/ossec/bin/wazuh-control status
```

## ✅ Verifiche di Funzionamento

### Test 1: Verifica Volume Mount
```bash
sudo docker exec single-node-wazuh.manager-1 ls -la /var/log/suricata/
```
**Atteso:** Lista dei file Suricata (eve.json, fast.log, stats.log, suricata.log)

### Test 2: Verifica Logcollector
```bash
sudo docker exec single-node-wazuh.manager-1 grep "Analyzing file.*eve.json" /var/ossec/logs/ossec.log
```
**Atteso:** Messaggio "Analyzing file: '/var/log/suricata/eve.json'"

### Test 3: Test delle Regole Suricata
```bash
echo '{"timestamp":"2025-09-13T22:44:06.000000+0200","event_type":"alert","src_ip":"192.168.1.100","dest_ip":"192.168.1.1","dest_port":80,"proto":"TCP","alert":{"action":"allowed","gid":1,"signature_id":2019236,"rev":3,"signature":"ET WEB_SERVER Possible CVE-2014-6271 Attempt","category":"Attempted Administrator Privilege Gain","severity":1}}' | sudo docker exec -i single-node-wazuh.manager-1 /var/ossec/bin/wazuh-logtest -l /var/log/suricata/eve.json
```
**Atteso:** Decodifica JSON e attivazione regola ID 86601

## 📊 Monitoraggio in Tempo Reale

### Comandi Utili per il Monitoraggio

```bash
# Monitorare alert Wazuh
sudo docker exec single-node-wazuh.manager-1 tail -f /var/ossec/logs/alerts/alerts.json

# Monitorare log Wazuh
sudo docker exec single-node-wazuh.manager-1 tail -f /var/ossec/logs/ossec.log

# Monitorare eventi Suricata
sudo docker exec single-node-wazuh.manager-1 tail -f /var/log/suricata/eve.json

# Verificare eventi Suricata processati
sudo docker exec single-node-wazuh.manager-1 grep -i "suricata\|flow_id\|event_type" /var/ossec/logs/archives/archives.log
```

## 🔍 Tipi di Eventi Processati

| Tipo Evento Suricata | Descrizione | Genera Alert Wazuh |
|---------------------|-------------|-------------------|
| `alert` | Eventi di sicurezza rilevati | ✅ Sì (Level 3+) |
| `flow` | Informazioni sui flussi di rete | ⚠️ Solo se sospetti |
| `stats` | Statistiche periodiche | ❌ No (informativo) |
| `dns` | Query DNS | ⚠️ Solo se sospette |
| `http` | Traffico HTTP | ⚠️ Solo se sospetto |

## 📁 File di Configurazione Modificati

### `/opt/wazuh-docker/single-node/docker-compose.yml`
- **Aggiunto:** Volume mount `/var/log/suricata:/var/log/suricata:ro`

### `/opt/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf`  
- **Aggiunto:** Sezione `<localfile>` per eve.json con formato JSON

## 🔧 Risoluzione Problemi

### Problema: Wazuh non legge i file Suricata
**Soluzione:**
```bash
# Verificare mount
sudo docker inspect single-node-wazuh.manager-1 | grep -A 5 -B 5 suricata

# Verificare permessi
sudo docker exec single-node-wazuh.manager-1 ls -la /var/log/suricata/
```

### Problema: Errori di configurazione Wazuh
**Soluzione:**
```bash
# Verificare sintassi configurazione
sudo docker exec single-node-wazuh.manager-1 /var/ossec/bin/wazuh-control configtest

# Ripristinare backup
sudo cp /opt/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf.backup /opt/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf
```

### Problema: Container non si avvia
**Soluzione:**
```bash
# Verificare log container
sudo docker logs single-node-wazuh.manager-1 --tail 50

# Riavvio completo
sudo docker compose down && sudo docker compose up -d
```

## 📝 Regole Wazuh per Suricata

Le regole Suricata sono definite in:
- `/var/ossec/ruleset/rules/0475-suricata_rules.xml`
- `/var/ossec/ruleset/rules/0240-ids_rules.xml`

**Regole principali:**
- **86600**: Base Suricata message (Level 0)
- **86601**: Suricata Alert (Level 3)
- **86602+**: Regole specifiche per severity e categoria

## 🔄 Manutenzione

### Rotazione Log Suricata
Assicurarsi che la rotazione dei log Suricata non interrompa la lettura di Wazuh:

```bash
# Verificare configurazione logrotate
sudo cat /etc/logrotate.d/suricata
```

### Backup Periodico
```bash
# Script di backup configurazione
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
sudo cp /opt/wazuh-docker/single-node/docker-compose.yml /opt/backup/docker-compose.yml.$DATE
sudo cp /opt/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf /opt/backup/wazuh_manager.conf.$DATE
```

## 📈 Metriche di Successo

✅ **Integrazione Riuscita se:**
- Wazuh logcollector analizza eve.json ✓
- Eventi Suricata vengono decodificati ✓  
- Alert Wazuh generati per eventi "alert" di Suricata ✓
- Volume mount funzionante ✓
- Nessun errore nei log Wazuh ✓

## 🔗 Riferimenti

- [Wazuh Documentation - Log Data Analysis](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-analysis/index.html)
- [Suricata Documentation - Eve JSON](https://docs.suricata.io/en/suricata-6.0.0/output/eve/eve-json-output.html)
- [Wazuh Rules Reference](https://documentation.wazuh.com/current/user-manual/ruleset/index.html)

---
**Creato:** 13 Settembre 2025  
**Autore:** Integrazione Suricata-Wazuh  
**Versione:** Wazuh 4.12.0 + Suricata IDS  
**Stato:** ✅ Funzionante e Testato
