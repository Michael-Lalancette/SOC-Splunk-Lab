## Phase 1 - Réseaux virtuels

### 🎯 Objectif
Définir 2 réseaux virtuels sous VMware : un réseau **isolé** pour la communication interne du laboratoire et un réseau **NAT** permettant un accès Internet temporaire afin d’installer les outils/updates nécessaires..

### VMnet1 (Host-Only)
  - Créer/configurer un réseau Host-Only dédié.  
  - Désactivez le DHCP.  
  -  IP statiques seront attribuées dans le subnet `10.7.0.0/24`.  
  > **Résultat ✅ :** Les VM connectées à ce réseau ne voient que les autres VM du même réseau (aucune sortie vers Internet).  
  ![VMnet1](./images/vmnet1.png)


### VMnet8 (NAT/DHCP)
  - Activé par défaut.  
  - Laisser DHCP activé.  
  > **Résultat ✅ :** Les VM rattachées à ce réseau obtiennent une adresse IP via DHCP et peuvent accéder à Internet pour télécharger les outils et effectuer les updates.  
  ![VMnet8](./images/vmnet8.png)


---

## Phase 2 - Configuration des VMs

### 🎯 Objectif
Déployer les VM du laboratoire, configurer leurs ressources matérielles et leurs interfaces réseau, puis installer les paquets de base nécessaires à leur bon fonctionnement.


### 🖥️ SOC-Splunk-Server
  **Specs** : 
  - OS : 👉 [Ubuntu Server 24.04.03 LTS](https://ubuntu.com/download/server) 
  - vCPU : 4
  - RAM : 12GB
  - Disque : 100GB
  - NIC1 : Internal - Host-only (`10.7.0.10/24`)
  - NIC2 : External - NAT/DHCP (temporaire)



  **Configurations** :  
  - Choisir installation minimale pour un maximum de contrôle.
  
  - Configuration de **eth0** (réseau interne)
    - Adresse IPv4 : `10.7.0.10`
    - DNS : `8.8.8.8`
        
    > 💡 Pourquoi pas de Gateway? La switch interne ne fournit aucun routage extérieur. Si une passerelle était indiquée, Ubuntu tenterait d’envoyer tout le trafic non‑local vers un chemin inexistant, ce qui provoquerait des pertes de connectivité.
    ![splunk-ipv4-1](./images/splunk-ipv4-1.png)

  - Configuration de **eth1** (réseau externe/NAT) :
    - Mode : DHCP automatique.
    - L’interface reçoit une IP dynamique (ex : `192.168.0.129`).
        
    > 💡 Fournit accès Internet (mises à jour + téléchargement de Splunk).  
    ![splunk-dhcp-1](./images/splunk-dhcp-1.png)

  - Après le reboot de la machine, installer paquets nécessaires :
    ```bash
    sudo apt update && sudo apt install -y openssh-server iputils-ping curl
    ```
      Ces paquets offrent :
    - `openssh-server` – accès SSH.
    - `iputils-ping` – outil de diagnostic réseau.  
    - `curl` – téléchargement de fichiers (utile pour récupérer le paquet Splunk).  


  **✅ Vérifications** :  
  - `ip a` → confirme la présence des deux interfaces (10.0.0.10 et 192.168.0.192).
  - `ping 8.8.8.8 -c 3` → vérifie la connectivité Internet.  
   ![splunk-verif](./images/splunk-verif.png)


  🚀 **Prochaine étape – Phase 3 : Installation de Splunk Enterprise**  
  - Ton `SOC-Splunk-Server` est maintenant prêt.
  - La prochaine phase consistera à déployer Splunk Enterprise sur Ubuntu, ouvrir les ports nécessaires, et préparer les premiers indexes pour accueillir les journaux Windows et Sysmon.    

> 💡 Prendre un snapshot de la VM juste avant d’installer Splunk, afin de pouvoir revenir rapidement en cas de problème.


### 🖥️ SOC-W11 (Victime)
  **Specs** : 
  - OS : 👉 [Microsoft Windows 11 ISO](https://www.microsoft.com/en-us/software-download/windows11?msockid=3093134cf4a46e83086606b8f5856f87)
  - vCPU : 2
  - RAM : 4GB
  - Disque : 60GB
  - NIC1 : Internal - Host-only (`10.7.0.20/24`)
  - NIC2 : External - NAT/DHCP (temporaire)

  **Configurations** :  
  - Configuration de **eth0** (réseau interne)
     ``` 
       Network Connections  
         └─ Ethernet0  
             └─ Properties 
                 └─ Internet Protocol Version 4 (TCP/IPv4)  
                    └─ ON
      ``` 

    ![w11-eth0](./images/w11-eth0.png)

  - Configuration de **eth1** (réseau externe/NAT) :
    - Mode : DHCP automatique.
    - L’interface reçoit une IP dynamique (ex : `192.168.0.130`).

    ![w11-eth1](./images/w11-eth1.png)


  **✅ Vérifications** :  
  - `ipconfig` → confirme la présence des deux interfaces (10.0.0.10 et 192.168.0.192).
  - `ping 8.8.8.8 -n 3` → vérifie la connectivité Internet.
  - `ping 10.7.0.10 -n 3` → vérifie la connectivité avec le serveur Splunk.  
   ![w11-verif-1](./images/w11-verif-1.png)  
   ![w11-verif-2](./images/w11-verif-2.png)








