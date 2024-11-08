
# Mise en place d'un Serveur VPN WireGuard avec Accès Restreint au Serveur Applicatif

Ce document décrit les étapes pour configurer un serveur VPN WireGuard permettant à des clients VPN d’accéder à un serveur applicatif distinct via une redirection. Les règles UFW garantiront que seules les machines connectées au VPN auront accès au serveur applicatif, hébergé sous Apache2. Les étapes incluent également des consignes de sécurité pour limiter l’accès SSH aux clients VPN.

## Prérequis
- **Environnement** : Machines sous VMware, connectées au réseau `192.168.179.0`
- **Serveur VPN WireGuard** : IP du VPN en `10.0.0.0`
- **Serveur Applicatif** : Héberge Apache2
- **Firewall** : UFW pour restreindre les accès

## Sommaire
1. [Installation du Serveur VPN WireGuard](#1-installation-du-serveur-vpn-wireguard)
2. [Configuration et Installation du Client VPN](#2-configuration-et-installation-du-client-vpn)
3. [Installation du Serveur Applicatif](#3-installation-du-serveur-applicatif)
4. [Configuration UFW](#4-configuration-ufw)
5. [Vérification des Accès](#5-vérification-des-accès)
6. [Réflexions et Considérations Sécuritaires](#6-réflexions-et-considérations-sécuritaires)

---

## 1. Installation du Serveur VPN WireGuard
1. **Mise à jour du serveur** :
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Installation de WireGuard** :
   ```bash
   sudo apt install wireguard -y
   ```

3. **Configuration du Serveur WireGuard** :
   - Créez les clés privées et publiques pour le serveur :
     ```bash
     wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
     ```
   - Configurez le fichier `/etc/wireguard/wg0.conf` :
     ```ini
     [Interface]
     Address = 10.0.0.1/24
     ListenPort = 51820
     PrivateKey = [clé privée du serveur]

     # Règle de redirection vers le serveur applicatif
     PostUp = iptables -A FORWARD -i wg0 -o ens192 -j ACCEPT
     PostUp = iptables -A FORWARD -o wg0 -i ens192 -j ACCEPT
     PostUp = iptables -t nat -A POSTROUTING -o ens192 -j MASQUERADE
     PostDown = iptables -D FORWARD -i wg0 -o ens192 -j ACCEPT
     PostDown = iptables -D FORWARD -o wg0 -i ens192 -j ACCEPT
     PostDown = iptables -t nat -D POSTROUTING -o ens192 -j MASQUERADE

     # Connexions des clients VPN
     [Peer]
     PublicKey = [clé publique du client]
     AllowedIPs = 10.0.0.2/32
     ```

4. **Démarrage de WireGuard** :
   ```bash
   sudo systemctl enable wg-quick@wg0
   sudo systemctl start wg-quick@wg0
   ```

## 2. Configuration et Installation du Client VPN
1. **Installer WireGuard sur le client** :
   ```bash
   sudo apt install wireguard -y
   ```

2. **Générer les clés du client** :
   ```bash
   wg genkey | tee /etc/wireguard/client_private.key | wg pubkey > /etc/wireguard/client_public.key
   ```

3. **Configurer le client** dans `/etc/wireguard/wg0.conf` :
   ```ini
   [Interface]
   Address = 10.0.0.2/24
   PrivateKey = [clé privée du client]

   [Peer]
   PublicKey = [clé publique du serveur]
   Endpoint = [IP publique du serveur VPN]:51820
   AllowedIPs = 192.168.179.0/24
   PersistentKeepalive = 25
   ```

4. **Démarrer WireGuard sur le client** :
   ```bash
   sudo systemctl enable wg-quick@wg0
   sudo systemctl start wg-quick@wg0
   ```

## 3. Installation du Serveur Applicatif (Apache2)
1. **Installation d’Apache2** :
   ```bash
   sudo apt update
   sudo apt install apache2 -y
   ```

2. **Vérification de l’installation** :
   - Accédez à `http://[IP du serveur applicatif]` pour voir la page d’accueil d’Apache.

## 4. Configuration UFW
1. **Autoriser uniquement les connexions VPN vers le serveur applicatif** :
   ```bash
   sudo ufw allow 51820/udp
   sudo ufw allow from 10.0.0.0/24 to any port 80,443 proto tcp
   ```

2. **Bloquer l’accès SSH aux autres, sauf pour le VPN** :
   ```bash
   sudo ufw allow from 10.0.0.0/24 to any port 22 proto tcp
   sudo ufw deny 22
   ```

3. **Activer UFW** :
   ```bash
   sudo ufw enable
   ```

## 5. Vérification des Accès
- **Vérifier l’accès du client VPN au serveur applicatif** :
  - Connectez-vous au client VPN.
  - Accédez à `http://[IP du serveur applicatif]` depuis le client pour confirmer l’accès.

- **Vérifier qu'un utilisateur non connecté au VPN n'a pas accès au serveur applicatif** :
  - Depuis un appareil hors VPN, essayez d’accéder à `http://[IP du serveur applicatif]`.
  - L’accès devrait être refusé.

## 6. Réflexions et Considérations Sécuritaires

1. **Restreindre l’accès au serveur applicatif uniquement pour le VPN** :
   - Utilisez les règles UFW pour que seul le réseau `10.0.0.0/24` puisse accéder au serveur applicatif.

2. **Limiter l’accès SSH au client VPN uniquement** :
   - Les règles UFW configurées permettent aux connexions SSH uniquement pour les adresses du réseau VPN `10.0.0.0/24`.

3. **Prévoir une issue de secours en cas de problème avec l’interconnexion** :
   - Pour éviter une perte d’accès SSH en cas de problème VPN, configurez une clé SSH pour un utilisateur d’urgence avec accès direct.
   - Documentez le processus de redémarrage manuel de WireGuard et assurez un accès physique ou un autre point d’entrée sécurisé pour les administrateurs.

---

**Note** : Adaptez les configurations et les règles en fonction de votre environnement et de la politique de sécurité de votre organisation.
