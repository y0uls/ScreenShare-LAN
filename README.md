# 🖥️ ScreenShare-LAN

> Outil de **télé-assistance interne** : visualisez l'écran d'un poste Windows de votre réseau local, depuis un simple navigateur, sans rien installer sur la cible.

![Platform](https://img.shields.io/badge/platform-Windows-0078D6?logo=windows)
![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B%20%7C%207%2B-5391FE?logo=powershell&logoColor=white)
![Language](https://img.shields.io/badge/UI-navigateur-orange)
![Auth](https://img.shields.io/badge/auth-Active%20Directory-blue)
![License](https://img.shields.io/badge/license-interne-lightgrey)

---

## 📑 Sommaire

- [Aperçu](#-aperçu)
- [Fonctionnalités](#-fonctionnalités)
- [Fonctionnement](#-fonctionnement)
- [Prérequis](#-prérequis)
- [Installation](#-installation)
- [Utilisation](#-utilisation)
- [Paramètres](#-paramètres)
- [Interface web](#-interface-web)
- [Enregistrement](#-enregistrement)
- [Dépannage](#-dépannage)
- [Sécurité &amp; usage responsable](#-sécurité--usage-responsable)
- [Structure du projet](#-structure-du-projet)

---

## 🎯 Aperçu

`ScreenShare-LAN.ps1` permet à un technicien de **voir en direct l'écran d'un poste distant** sur le LAN, le temps d'une intervention de support.

Le principe :

1. Le technicien lance le script depuis sa machine et saisit ses **identifiants administrateur AD**.
2. Le script se connecte au poste cible (via **WinRM**, avec replis **CIM/WMI** puis **PsExec**) et y déploie un petit serveur de capture.
3. Le **navigateur s'ouvre automatiquement** sur `http://<cible>:<port>/` et affiche l'écran distant, rafraîchi en temps réel.
4. À la fin, **tout est nettoyé** sur le poste cible (tâche, règle de pare-feu, fichiers, processus).

Aucune installation permanente sur la cible : le serveur de capture est déployé puis supprimé à chaque session.

---

## ✨ Fonctionnalités

- 🔌 **Zéro installation cliente** — l'affichage se fait dans n'importe quel navigateur.
- 🔁 **Connexion robuste** — 3 niveaux de repli : WinRM → CIM/WMI → PsExec.
- 🖼️ **Multi-écrans** — bascule entre « Tous les écrans » et chaque moniteur individuel.
- 🎯 **DPI mixtes gérés** — *Per-Monitor DPI Aware V2* : un écran portable à 150 % s'affiche correctement à côté de moniteurs à 100 %.
- ⚡ **Affichage réactif** — la page interroge l'endpoint `/snapshot` en boucle courte (robuste face aux tampons proxy/antivirus des connexions longues).
- 🖱️ **Curseur inclus** dans la capture.
- 🔒 **Vérification de session** — refuse de démarrer si le poste est verrouillé (capture impossible).
- 🧹 **Nettoyage complet** en fin de session.
- 🏷️ **Branding neutre** — dossier, tâche, règle de pare-feu et titre dérivés du paramètre `-AppName`.
- 🎥 **Enregistrement vidéo** — capture d'une séquence au format WebM directement depuis le navigateur du technicien, écran au choix, sans rien installer côté cible.

---

## ⚙️ Fonctionnement

Le script se déroule en 5 étapes (visibles dans la console) :

| Étape | Action |
|------:|--------|
| **1/5** | Test de connectivité + WinRM (activation via CIM ou PsExec si inactif) |
| **2/5** | Ajout de la cible aux `TrustedHosts` |
| **3/5** | Nettoyage préalable, création du dossier applicatif, `urlacl`, règle de pare-feu, dépôt de `Worker.ps1` + `launch.vbs`, détection de la session interactive |
| **3.5** | Vérification qu'une **session est active** (déverrouillée) sur la cible |
| **4/5** | Lancement du worker via **tâche planifiée → wscript** (fenêtre masquée) sur la session de l'utilisateur |
| **5/5** | Attente que le flux réponde, puis **ouverture du navigateur** |

Le **worker** est un serveur HTTP autonome (C# compilé à la volée via `Add-Type`) qui capture l'écran avec `System.Drawing` et l'expose en JPEG.

```
┌─────────────┐        WinRM / CIM / PsExec         ┌──────────────────────┐
│  Technicien │ ─────────────────────────────────▶ │   Poste cible (LAN)  │
│  (ce script)│                                     │  ┌────────────────┐  │
│             │                                     │  │  Worker.ps1    │  │
│  Navigateur │ ◀──── HTTP /snapshot  /monitors ── │  │  (serveur C#)  │  │
└─────────────┘         http://<cible>:8080/        │  └────────────────┘  │
                                                    └──────────────────────┘
```

---

## 📋 Prérequis

- **Windows** (technicien et cible), avec **PowerShell 5.1** ou **PowerShell 7+** (`pwsh`).
- **Droits administrateur** sur le poste cible via un compte **Active Directory**.
- **WinRM joignable** sur la cible, *ou* le blob **PsExec** renseigné dans le script (variable `$base64PsExec`) pour l'activer à distance.
- Une **session utilisateur ouverte et déverrouillée** sur la cible (sinon la capture est impossible).
- Un **LAN de confiance** : le flux HTTP est **en clair et sans authentification**.

---

## 📥 Installation

```powershell
git clone https://github.com/<votre-compte>/ScreenShare-LAN.git
cd ScreenShare-LAN
```

Aucune dépendance à installer : le script est autonome.

> 💡 Si la stratégie d'exécution bloque le script, lancez-le avec `-ExecutionPolicy Bypass` (voir ci-dessous) ou débloquez le fichier : `Unblock-File .\ScreenShare-LAN.ps1`.

---

## 🚀 Utilisation

Lancement interactif (le script demande la cible, le domaine et les identifiants) :

```powershell
powershell -ExecutionPolicy Bypass -File .\ScreenShare-LAN.ps1
```

Avec des paramètres :

```powershell
# Poste par IP, port et cadence personnalisés
powershell -ExecutionPolicy Bypass -File .\ScreenShare-LAN.ps1 -TargetPC 192.168.1.42 -Port 8080 -Fps 15

# Poste par nom, qualité réduite pour un lien lent
powershell -ExecutionPolicy Bypass -File .\ScreenShare-LAN.ps1 -TargetPC PC-COMPTA-03 -Quality 35 -MaxWidth 1280

# Branding personnalisé (nom du dossier / tâche / pare-feu)
powershell -ExecutionPolicy Bypass -File .\ScreenShare-LAN.ps1 -TargetPC PC-07 -AppName SupportShare
```

Le script se **relance automatiquement en administrateur** si nécessaire. À la fin de l'intervention, appuyez sur **Entrée** dans la console pour **arrêter le partage et tout nettoyer** sur la cible.

---

## 🎛️ Paramètres

| Paramètre   | Type   | Défaut            | Description |
|-------------|--------|-------------------|-------------|
| `-TargetPC` | string | *(demandé)*       | IP ou nom du poste cible |
| `-Port`     | int    | `8080`            | Port HTTP d'écoute sur la cible |
| `-Fps`      | int    | `10`              | Images par seconde |
| `-Quality`  | int    | `50`              | Qualité JPEG (1–100) |
| `-MaxWidth` | int    | `1920`            | Largeur max du flux ; réduction au-delà (`0` = pas de réduction) |
| `-Domain`   | string | `$env:USERDOMAIN` | Domaine AD pour l'authentification |
| `-AppName`  | string | `ScreenShare`     | Branding : pilote dossier, tâche, règle de pare-feu et titre |

---

## 🌐 Interface web

Le serveur expose plusieurs points d'accès :

| Route              | Description |
|--------------------|-------------|
| `/`                | Page de visualisation (barre de sélection d'écran, rafraîchissement auto, bouton d'enregistrement) |
| `/snapshot?mon=N`  | Une image JPEG. `mon` = index de l'écran ; absent = bureau complet |
| `/monitors`        | Liste des écrans au format JSON |
| `/diag`            | Diagnostic (dernière erreur, géométrie, cache) |
| `/stream?mon=N`    | Flux MJPEG *(historique — conservé mais plus utilisé par la page)* |

---

## 🎥 Enregistrement

La page de visualisation intègre un **enregistreur vidéo** côté navigateur : rien n'est installé ni écrit sur le poste cible.

- Choisissez l'écran dans la barre (`Tous`, `Ecran 1`, `Ecran 2`…), puis cliquez **● REC** ; le bouton passe en **■ STOP** (rouge) pendant la capture.
- Un second clic arrête l'enregistrement et **télécharge** un fichier `capture-AAAAMMJJ-HHMMSS.webm` sur la machine du technicien.
- La séquence suit l'écran affiché : changer de moniteur en cours de capture est possible (taille fixe, bandes noires si le format change).
- Cadence et qualité calées sur le flux (`-Fps` / `-Quality`). Format **WebM** (VP9/VP8), convertible en MP4 avec `ffmpeg` au besoin.

> Compatible Chrome, Edge et Firefox. Sur un navigateur sans prise en charge de `MediaRecorder`, un message l'indique et le partage se poursuit normalement.

---

## 🩺 Dépannage

| Symptôme | Cause probable / solution |
|----------|---------------------------|
| **« Aucune session interactive active »** | Le poste est verrouillé ou personne n'est connecté. Demandez à l'utilisateur de déverrouiller sa session, puis relancez. |
| **WinRM inaccessible** | Activez WinRM sur la cible (`Enable-PSRemoting -Force`) ou renseignez le blob **PsExec** (`$base64PsExec`) dans le script. |
| **`WSMan:\localhost\Client\TrustedHosts` introuvable** | Le service WinRM **local** (côté technicien) n'était pas démarré. Le script le lance désormais automatiquement ; une console **administrateur** est requise. |
| **Écran noir malgré une session ouverte** | Consultez `http://<cible>:<port>/diag` et le journal `C:\ProgramData\<AppName>\worker.log`. |
| **Le flux ne répond pas après 25 s** | Pare-feu, port déjà utilisé, ou worker non lancé. Vérifiez qu'une session est active et testez un autre `-Port`. |
| **Le port est déjà pris** | Relancez avec un autre `-Port` (ex. `-Port 8090`). |

Journaux et fichiers utiles sur la cible (supprimés au nettoyage) : `C:\ProgramData\<AppName>\` (`Worker.ps1`, `launch.vbs`, `worker.log`, `worker.pid`).

---

## 🔐 Sécurité & usage responsable

> ⚠️ **À lire avant tout déploiement.**

- **Flux en clair et sans authentification.** N'importe quelle machine du segment réseau peut ouvrir `http://<cible>:<port>/`. À réserver **strictement à un LAN de confiance** ; ne jamais exposer le port sur Internet.
- **Traces visibles sur la cible.** L'outil est *neutre*, pas *furtif* : la tâche planifiée, la règle de pare-feu, le port en écoute et des événements Windows (création de tâche, WinRM 4104, journal Security) restent visibles. Le nettoyage supprime les artefacts créés, mais pas les événements d'audit déjà écrits.
- **Cadre légal.** Capturer l'écran d'une personne suppose de l'**informer au préalable** et de disposer d'une base légitime. En France, la surveillance des salariés est encadrée (**Code du travail**, doctrine **CNIL**) : une captation à l'insu de l'utilisateur est en principe **illicite**. Le standard pour du support est que la personne **sait** et **consent** (information, politique interne, ou demande de prise en main).
- **Recommandation.** Pour un usage conforme, affichez un **indicateur visible de session de support** côté poste et/ou recueillez le **consentement** de l'utilisateur avant de démarrer.
- **Enregistrements.** Les vidéos produites par le bouton REC sont des **données personnelles** : conservées sur le poste du technicien, elles relèvent des mêmes règles d'information et de consentement, et doivent être protégées puis supprimées selon la politique interne.

Utilisez cet outil uniquement sur des machines que vous êtes **autorisé à administrer**, dans le respect de la réglementation applicable et de la politique de votre organisation.

---

## 📂 Structure du projet

```
ScreenShare-LAN/
├── ScreenShare-LAN.ps1     # Script principal (déploiement distant + worker embarqué)
└── README.md               # Ce fichier
```

---

<p align="center"><sub>Outil de télé-assistance interne — à utiliser sur réseau de confiance et dans le respect de la réglementation.</sub></p>
