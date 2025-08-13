# 🚀 Installation de Docker

Ce guide explique comment installer **Docker** et vérifier que l’installation fonctionne correctement.

---

## 📦 Prérequis

- Un système d’exploitation supporté :
  - Linux (Ubuntu, Debian, Fedora…)
  - macOS
  - Windows 10/11 (Pro ou Home avec WSL2)
- Droits administrateur (`sudo` sous Linux)
- Connexion internet

---

## 🔧 Installation

### Linux (Ubuntu / Debian)

```bash
# Mettre à jour les paquets
sudo apt update

# Installer les dépendances nécessaires
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Ajouter la clé GPG officielle de Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Ajouter le dépôt Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installer Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
