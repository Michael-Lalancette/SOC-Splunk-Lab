## 📑 Table des matières

- [Phase 1 — Réseaux virtuels](#phase-1---réseaux-virtuels)
  - [VMnet1 (Host-Only)](#vmnet1-host-only)
  - [VMnet8 (NAT/DHCP)](#vmnet8-natdhcp)

- [Phase 2 — Configuration des VMs](#phase-2---configuration-des-vms)
  - [SOC-Splunk-Server](#️-soc-splunk-server)
  - [SOC-W11](#️-soc-w11)
  - [SOC-ATK](#️-soc-atk)
  - [SOC-Workstation](#️-soc-workstation)

- [Phase 3 — Installation de Splunk Enterprise](#phase-3---installation-de-splunk-enterprise)
  
  
- [Phase 4 — Configuration des Forwarders & Logs](#phase-4---configuration-des-forwarders--logs)
- [Phase 5 — Détection & Alerting](#phase-5---détection--alerting)
- [Phase 6 — Investigation & Workflows](#phase-6---investigation--workflows)



---



## Phase 1 - Réseaux virtuels

### 🎯 Objectif
Mettre en place deux réseaux virtuels sous VMware pour le laboratoire :  
  - Un réseau isolé (Host-Only) pour la communication interne entre les VMs du lab, sans passerelle vers l’extérieur.  
  - Un réseau externe (NAT) pour fournir temporairement un accès internet aux VMs (mises à jour et téléchargements d’outils).  


### VMnet1 (Host-Only)
  - Créer/configurer un réseau Host-Only dédié.  
  - Désactiver le DHCP.
  - Plage IP : `10.7.0.0/24` (adresses attribuées manuellement).
  > **Résultat ✅ :** Les VMs connectées à VMnet1 communiquent entre elles uniquement, sans accès à internet ni au réseau physique de l’hôte.  
  ![VMnet1](./images/vmnet1.png)


### VMnet8 (NAT/DHCP)
  - Activé par défaut dans VMware.
  - Laisser le DHCP activé (distribution auto d’adresses).
  - Plage IP : `172.16.0.0/24` (adresses attribuées dynamiquement aux VMs).
  - Ce réseau utilise le NAT (Network Address Translation) pour fournir un accès internet aux VMs.  
  > **Résultat ✅ :** Les VMs connectées à VMnet8 peuvent accéder à internet pour téléchargements et mises à jour. 
  ![VMnet8](./images/vmnet8.png)


---

## Phase 2 - Configuration des VMs

### 🎯 Objectif
Déployer et préparer les machines virtuelles du laboratoire : définir les ressources, configurer les interfaces réseau, installer les paquets de base, et effectuer des vérifications simples avant la phase d’application.


### 🖥️ SOC-Splunk-Server
  **Specs** : 
  - OS : 👉 [Ubuntu Server 24.04.03 LTS](https://ubuntu.com/download/server) 
  - vCPU : 4
  - RAM : 12GB
  - Disque : 100GB
  - NIC1 : Host-only (`10.7.0.10/24`)
  - NIC2 : NAT/DHCP (`172.16.0.x/24`) - temporaire



  **Configuration réseau** :
  - Choisir installation minimale pour garder contrôle sur les paquets installés.
  
  - Configuration de **eth0** – réseau interne (Host‑Only, adresse statique)
    - Adresse IPv4 : `10.7.0.10`
    - DNS : `8.8.8.8`
        
    > 💡 Pourquoi pas de Gateway? Le réseau host-only est non routé : indiquer une gateway pousserait tout le trafic non-local vers un chemin inexistant et provoquerait des pertes de connectivité.
    ![splunk-ipv4-1](./images/splunk-ipv4-1.png)

  - Configuration de **eth1** – réseau externe (NAT, adresse dynamique via DHCP) :
    - Mode : DHCP automatique.
    - L’interface reçoit une IP dynamique (ex : `172.16.0.129`).
        
    > 💡 Fournit accès Internet (mises à jour + téléchargement de Splunk).  
    ![splunk-dhcp-1](./images/splunk-dhcp-1.png)

  - Après le reboot de la machine, installer paquets essentiels et activer SSH :
      ```bash
      # Installer paquets
      sudo apt update
      sudo apt install -y openssh-server iputils-ping curl net-tools

      # Activer/démarrer SSH au boot
      sudo systemctl enable --now ssh

      # Vérifier SSH
      systemctl status ssh
      ```


  **✅ Vérifications** :  
  - `ip -br a` → confirme la présence des deux interfaces (`10.0.0.10` et `172.16.0.129`).
  - `ping 8.8.8.8 -c 3` → vérifie la connectivité Internet.  
   ![splunk-verif](./images/splunk-verif.png)


> ⚠️ Prendre un snapshot de la VM juste avant d’installer Splunk, afin de pouvoir revenir rapidement en cas de problème.

---

### 🖥️ SOC-W11
  **Specs** : 
  - OS : 👉 [Microsoft Windows 11 ISO](https://www.microsoft.com/en-us/software-download/windows11?msockid=3093134cf4a46e83086606b8f5856f87)
  - vCPU : 2
  - RAM : 4GB
  - Disque : 60GB
  - NIC1 : Host-only (`10.7.0.20/24`)
  - NIC2 : NAT/DHCP (`172.16.0.x/24`) - temporaire

  **Configuration réseau** :  
  - Configuration de **eth0** – réseau interne (Host‑Only, adresse statique)  
      1. `Win + R` → taper `ncpa.cpl` → OK (ouvre directement Connexions réseau)
      2. Ethernet → Propriétés → Internet Protocol Version 4 (TCP/IPv4)  

    ![w11-eth0](./images/w11-eth0.png)

  - Configuration de **eth1** – réseau externe (NAT, adresse dynamique via DHCP) :
    - Mode : DHCP automatique.
    - L’interface reçoit une IP dynamique (ex : `172.16.0.130`).

    ![w11-eth1](./images/w11-eth1.png)


  **✅ Vérifications** :  
  - `ipconfig` → confirme la présence des deux interfaces (`10.0.0.20` et `172.16.0.130`).
  - `ping 8.8.8.8 -n 3` → vérifie la connectivité Internet.
  - `ping 10.7.0.10 -n 3` → vérifie la connectivité avec le serveur Splunk.  
   ![w11-verif-1](./images/w11-verif-1.png)  
   ![w11-verif-2](./images/w11-verif-2.png)


> ⚠️ Prendre un snapshot "clean" de la VM en cas d'incident.

---

### 🖥️ SOC-ATK
  **Specs** : 
  - OS : 👉 [Kali Linux ](https://www.kali.org/)
  - vCPU : 2
  - RAM : 4GB
  - Disque : 40GB
  - NIC1 : Host-only (`10.7.0.30/24`)
  - NIC2 : NAT/DHCP (`172.16.0.x/24`) - temporaire

  **Configuration réseau (netplan)** :  
  - Interface **eth0** – réseau interne (Host‑Only, adresse statique)
    ```bash
    sudo nmcli con add type ethernet ifname eth0 con-name eth0-static ipv4.addresses 10.7.0.30/24 ipv4.dns "8.8.8.8 1.1.1.1" ipv4.method manual
    sudo nmcli con up eth0-static
    ```

  - Interface **eth1** – réseau externe (NAT, adresse dynamique via DHCP)
    ```bash
    sudo nmcli con add type ethernet ifname eth1 con-name eth1-dhcp ipv4.method auto
    sudo nmcli con up eth1-dhcp
    ```
    ![kali-cli-eth](./images/kali-cli-eth.png)   

    > 💡 Remarque : selon la version de Kali, les interfaces peuvent être renommées (ex : ens33, ens34, etc.).    
    >  Identifiez les noms exacts avec la commande `ip -br a` et adaptez les paramètres `ifname` en conséquence.  

  **✅ Vérifications** :  
  - `ip -br a` → confirme la présence des deux interfaces (`10.0.0.30` et `172.16.0.131`).
  - `ping 8.8.8.8 -c 3` → vérifie la connectivité Internet.
  - `ping 10.7.0.[10-20] -c 3` → vérifie la connectivité avec les différentes VMs.
    ![kali-cli-verif-1](./images/kali-cli-verif-1.png)    
    ![kali-cli-verif-2](./images/kali-cli-verif-2.png)     

  - N.B : Pour autoriser le ping vers la machine Windows, il faut activer la règle **ICMPv4-In** dans le pare-feu de la machine Windows.    
    ![win11-firewall-icmpv4](./images/win11-firewall-icmpv4.png)  
    - Une fois la règle activée, la commande `ping 10.7.0.20 -c 3` confirme la connectivité.  


> ⚠️ Prendre un snapshot "clean" de la VM en cas d'incident.

  ---



### 🖥️ SOC-Workstation
  **Specs** : 
  - OS : 👉 [Ubuntu Desktop 24.04.3 LTS](https://ubuntu.com/download/desktop)
  - vCPU : 4
  - RAM : 8GB
  - Disque : 40GB
  - NIC1 : Host-only (`10.7.0.40/24`)
  - NIC2 : NAT/DHCP (`172.16.0.x/24`) - temporaire

  **Configuration réseau** :
  - Configuration de **eth0/ens33** – réseau interne (Host‑Only, adresse statique)
    - Adresse IPv4 : `10.7.0.40`
    - Netmask : `255.255.255.0`
    - DNS : `8.8.8.8, 1.1.1.1`
    
    ![soc-eth0](./images/soc-eth0.png)


  - Configuration de **eth1/ens34** – réseau externe (NAT, adresse dynamique via DHCP) :
    - Dans IPv4 : IPv4 Method = Automatic (DHCP).
    - L’interface reçoit une IP dynamique (ex : `172.16.0.132`).



  **✅ Vérifications** :  
  - `ip -br a` → confirme la présence des deux interfaces  (`10.0.0.40` et `172.16.0.132`).
  - `ping 8.8.8.8 -c 3` → vérifie la connectivité Internet.
  - `ping 10.7.0.[10-30] -c 3` → vérifie la connectivité avec les différentes VMs.

    ![soc-verif-1](./images/soc-verif-1.png)    
    ![soc-verif-2](./images/soc-verif-2.png)  

> ⚠️ Prendre un snapshot "clean" de la VM en cas d'incident.

---

## 📊 Tableau Récapitulatif
| VM                | OS                   | eth0 (Host-only) | eth1 (NAT/DHCP) | Rôle            |
| ----------------- | -------------------- | ---------------- | --------------- | --------------- |
| SOC-Splunk-Server | Ubuntu Server 24.04  | 10.7.0.10/24     | DHCP            | Collecte & SIEM |
| SOC-W11           | Windows 11           | 10.7.0.20/24     | DHCP            | Victime         |
| SOC-Kali          | Kali Linux           | 10.7.0.30/24     | DHCP            | Attaquant       |
| SOC-Workstation   | Ubuntu Desktop 24.04 | 10.7.0.40/24     | DHCP            | Analyste        |



---

## Phase 3 - Installation de Splunk Enterprise

### 🎯 Objectif  
Installer Splunk Enterprise sur la VM `SOC-Splunk-Server`, activer le service, configurer l’autostart et valider l’accès au tableau de bord depuis la station analyste.




### 1. Téléchargement de Splunk Enterprise  
  - Naviguer sur la page [Splunk Enterprise](https://www.splunk.com/en_us/download/splunk-enterprise.html).  
  - Créer un compte Splunk et choisir l’installateur Linux `.deb`.  
  - Copier le lien `wget` fourni par Splunk (option Copy wget link).  
> 💡 Cette URL sera utilisée plus tard avec `wget`depuis le serveur Ubuntu.  
    ![splunk-download](./images/splunk-download.png) 




### 2. Connexion SSH 
  - Depuis le poste SOC-Workstation (Ubuntu Desktop), se connecter sur le serveur Ubuntu via SSH :  
    ```bash
    ssh splunk-admin@10.7.0.10
    ```
    ![ssh](./images/ssh.png)  




### 3. Récupération et installation
  - Récupérer le fichier `.deb` avec `wget` :  
    ```bash
    wget -O splunk.deb "<URL_copiée_avec_wget>"
    ```  
    ![splunk-download-2](./images/splunk-download-2.png)
      
    > N.B. : `-O` nomme spécifiquement le fichier `splunk.deb` au lieu du long nom par défaut.

  - Installer le paquet Splunk :  
    ```bash
    sudo dpkg -i splunk.deb
    ```  
  - Lancer Splunk et accepter la license :
    ```bash
    sudo /opt/splunk/bin/splunk start --accept-license
    ```  
  - Créer le compte admin (`splunk-admin`) et lui associer un mot de passe approprié.
    > 💡 Ce sera vos credentials pour vous connecter via l'interface.  
  - L'URL d'accès est indiquée à la fin du téléchargement : `http://10.7.0.10:8000`  

  - Pour faire démarrer automatiquement Splunk au boot :
    ```bash
    sudo /opt/splunk/bin/splunk enable boot-start
    ```
  - Vérifier finalement que le service est up and running :
    ```bash
    sudo /opt/splunk/bin/splunk status
    ```  
    > **Résultat ✅ :** `splunkd` en cours d’exécution (PID xxxx) et tous les helpers actifs.  




### 4. Accès au Splunk Dashboard
  - Sur la VM SOC‑Workstation :  
    - Ouvrir Firefox.  
    - Saisir `http://10.7.0.10:8000`.  
    - La page de connexion Splunk s’affiche.
    ![splunk-dash-1](./images/splunk-dash-1.png)  
    - Se connecter avec les identifiants créés précédemment.  
  > **Résultat ✅ :** Le tableau de bord Splunk Enterprise apparaît, confirmant que le serveur est fonctionnel et joignable depuis le réseau interne.  
    ![splunk-dash-2](./images/splunk-dash-2.png)  




### 📌 Bilan
  - Splunk installé, démarrage automatique configuré, service actif sur le port `8000`.  
  - Interface web accessible depuis la station analyste.  
  - Prêt pour la phase suivante : configuration des inputs, forwarders et premières recherches.   



> ⚠️ Snapshot : prenez un snapshot de la VM SOC‑Splunk‑Server avant de poursuivre avec la configuration des inputs.



---






