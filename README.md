# 🛡️ Laboratoire SOC avec Splunk
**Projet :** Lab SOC complet pour entraîner la **détection**, l’**alerte** et l’**investigation** avec **Splunk**, attaques simulées et mappage **MITRE ATT&CK**. 
**Plateformes :** VMware, Ubuntu, Windows 11, Kali Linux, Splunk Enterprise. 

---  

## 📖 Vue d’ensemble  

Ce dépôt fournit tout le nécessaire pour reproduire un SOC fonctionnel :
- Réseau virtuel segmenté (interne + externe).  
- Splunk SIEM qui collecte les logs IIS via le **Universal Forwarder**.  
- Honeypot IIS hébergeant `customer-data-dump.html`.  
- Attaquant Kali qui réalise des scans (`nmap`) et des téléchargements (`curl`, `wget`) pour déclencher une alerte.  
- Tableau de bord temps réel et rapport SOC alignés sur le framework **MITRE ATT&CK**.

Le laboratoire est idéal pour :  
* Apprendre à configurer la chaîne `log → SIEM → alerte`.  
* S’entraîner au triage d’incidents.  
* Produire un rapport d’enquête SOC structuré.  

---  

## 📐 Architecture  

### Réseau interne – VMnet2 (`10.0.0.0/24`)  
  - `10.7.0.10`	Splunk SIEM (Ubuntu) - Collecte, corrélation, alertes  
  - `10.7.0.20`	Windows 10 (IIS) - Honeypot (héberge `customer-data-dump.html`)  
  - `10.7.0.30`	Kali Linux	Attaquant - Exécute curl, wget, nmap  
  - `10.7.0.40`	Analyste (Ubuntu) - Accès tableau de bord Splunk  
> Tous les NIC internes sont connectés uniquement à VMnet2 afin que le trafic de test reste isolé.  

### Réseau externe (NAT) – VMnet3 (`172.16.0.0/24`)  
  - Permet aux machines du réseau interne d’accéder à Internet (pour mises à jour + téléchargements).  
> Aucun port entrant n’est exposé depuis l’extérieur ; le laboratoire reste complètement isolé.

### Flux de données principal  
`Kali (10.0.0.30)` → `Windows IIS (10.0.0.20` → `Universal Forwarder (UF)` → `Splunk SIEM (10.0.0.10)`  
  - Le UF lit les logs IIS (`C:\inetpub\logs\LogFiles\W3SVC1\*`) et les transmet à Splunk via `TCP 9997`.  
  - Splunk indexe les logs, applique la recherche `uri_path="/customer-data-dump.html"` et déclenche l’alerte e‑mail.  


---
## 📘 Documentation détaillée

<details>
1️⃣ **Créer les réseaux virtuels**  
   - **VMnet2** : *Host‑Only*, DHCP OFF.  
   - **VMnet3** : *NAT*, DHCP ON.  
  > *Configurer dans le Virtual Network Editor* 

2️⃣ **Déployer les 4 VM** et affecter les cartes réseau :  
   - **NIC 1** → **VMnet2** (interne)  
   - **NIC 2** → **VMnet3** (externe)  

3️⃣ **Attribuer les IP statiques** sur le réseau interne (**VMnet2**) :  
   - Splunk SIEM `10.0.0.10`  
   - Windows 11 `10.0.0.20`  
   - Kali Linux `10.0.0.30`  
   - Analyste `10.0.0.40`  
   > *Pas de passerelle sur VMnet2 ; la route par défaut provient de VMnet3/NAT* 

4️⃣ **Installer Splunk Enterprise** sur Ubuntu Server (`10.0.0.10`)  
   - Activer l’écoute sur le port **9997**.  
   - Créer l’index **`iis_logs`**.  

5️⃣ **Configurer Windows 10**  
   - Activer le rôle **IIS**.  
   - Installer le **Splunk Universal Forwarder** et le pointer vers `10.0.0.10:9997`.  

6️⃣ **Créer le honeypot**  
   - Ajouter le fichier **`/customer-data-dump.html`** dans le répertoire web d’IIS.  

7️⃣ **Créer l’alerte Splunk**  
   - Recherche : `index=iis_logs uri_path="/customer-data-dump.html"`  
   - Type : **Per‑Result**.  
   - Action : **Send email** via **SMTP Mailtrap** (API key, port 2525).  

8️⃣ **Simuler l’attaque depuis Kali**  
   ```bash
   curl http://10.0.0.20/customer-data-dump.html
   wget http://10.0.0.20/customer-data-dump.html
   nmap -sS 10.0.0.20
  ```

9️⃣ **Vérifier**  
  - Les logs apparaissent dans l’index `iis_logs`.  
  - Un e‑mail d’alerte est reçu dans Mailtrap.  
  - Le tableau de bord Splunk se met à jour (compteur d’accès, top IP, codes HTTP).  

🔟 Cartographier les événements vers le framework MITRE ATT&CK pour le reporting.  

</details>


➡️ **Guide des phases détaillées** : [👉](soc-splunk-lab/GUIDE.md)

---

## 📚 Références
- [Splunk Enterprise](https://docs.splunk.com/Documentation/Splunk/latest/Installation/InstallonLinux) – Guide d’installation officielle (Linux)  
- [Universal Forwarder](https://docs.splunk.com/Documentation/Forwarder/latest/Forwarder/Configuretheuniversalforwarder) – Documentation de configuration des inputs  
- [IIS Logging](https://learn.microsoft.com/en-us/iis/configuration/system.applicationHost/sites/site/logFile) – Paramètres de journalisation IIS 10   
- [MITRE ATT&CK](https://attack.mitre.org/) – Base de connaissances ATT&CK (TTPs)  
- [Mailtrap](https://mailtrap.io/) – Service de test d’e‑mail (sandbox)  
- [Kali Linux Tools](https://www.kali.org/tools/) – Documentation officielle de `curl`, `wget`, `nmap`  
- [VMware Workstation](https://docs.vmware.com/en/VMware-Workstation-Pro/16.0/com.vmware.ws.using.doc/GUID-4E2A4F73-5D44-4E5A-9F6C-0F0C9F0A2FDC.html) – Guide de création de commutateurs VMnet  
- [Splunk Dashboard XML](https://docs.splunk.com/Documentation/Splunk/latest/Viz/DashboardXML) – Format d’import/export des tableaux de bord  


---







