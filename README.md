# ESP32-8048S050-Lecteur-Multim-dia-Interactif
Borne tactile avec affichage d’images BMP et lecture audio MP3 - Arduino / LovyanGFX / ESP32-audioI2S

![Platform](https://img.shields.io/badge/platform-ESP32--S3-blue)
![Framework](https://img.shields.io/badge/framework-Arduino-teal)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Vue d’ensemble

Un lecteur multimédia interactif autonome pour la carte **Sunton ESP32-8048S050**.

* L’écran d’accueil affiche `logo.bmp` avec 3 boutons tactiles dessinés par-dessus
* Appuyer sur un bouton charge une image BMP en plein écran et lance le fichier MP3 associé
* À la fin de l’audio, le système revient automatiquement à l’écran d’accueil
* Pas d’OS, pas de tâches RTOS — simple boucle Arduino

---

## Matériel

| Composant    | Détails                                        |
| ------------ | ---------------------------------------------- |
| Carte        | Sunton ESP32-8048S050**C** (tactile capacitif) |
| SoC          | ESP32-S3, double cœur LX7, 240 MHz             |
| Écran        | 5" IPS, 800×480, RGB-565 parallèle             |
| Tactile      | GT911 capacitif, I2C                           |
| Audio        | Amplificateur MAX98357 I2S (intégré)           |
| Stockage     | Micro-SD (mode SPI 1-bit)                      |
| Haut-parleur | 4Ω ou 8Ω, min 1W                               |
| Alimentation | **Chargeur USB 5V / 2A requis**                |

> ⚠️ **Ne pas alimenter via un port USB d’ordinateur.** Le MAX98357 avec un haut-parleur 4Ω provoque des pics de courant qui entraînent des redémarrages (brownout) avec une alimentation faible.

---

## Dépendances

À installer via le gestionnaire de bibliothèques de l’IDE Arduino :

| Bibliothèque   | Auteur       | Version testée |
| -------------- | ------------ | -------------- |
| LovyanGFX      | lovyan03     | ≥ 1.1.x        |
| ESP32-audioI2S | schreibfaul1 | ≥ 2.x          |

Bibliothèques Arduino ESP32 intégrées utilisées : `SD_MMC`, `FS`

---

## Configuration Arduino IDE

| Paramètre           | Valeur                    |
| ------------------- | ------------------------- |
| Carte               | ESP32S3 Dev Module        |
| Schéma de partition | **Huge APP (3MB No OTA)** |
| PSRAM               | OPI PSRAM                 |
| Vitesse d’upload    | 921600                    |

> ⚠️ La partition **Huge APP est obligatoire** — ESP32-audioI2S ne tient pas dans la configuration par défaut.

---

## Configuration de la carte SD

Formatez la carte SD en **FAT32** et placez les fichiers suivants à la racine :

```
/
├── logo.bmp       ← fond écran d’accueil
├── bmg1.bmp       ← image bouton 1
├── bmg3.bmp       ← image bouton 2
├── bmg4.bmp       ← image bouton 3
├── BE-C.mp3       ← audio bouton 1
├── BE-F.mp3       ← audio bouton 2
└── BE-G.mp3       ← audio bouton 3
```

### Format des images

Les fichiers BMP doivent être en **24 bits non compressés, 800×480 pixels**.

Conversion avec FFmpeg :

```bash
ffmpeg -i input.png -vf scale=800:480 -pix_fmt bgr24 output.bmp
```

Ou avec GIMP :
Fichier → Exporter sous → BMP → 24 bits, sans compression RLE.

---

## Disposition des boutons

Trois boutons sont dessinés par programme sur `logo.bmp`.
Positions par défaut (modifiables dans `main.ino`) :

```cpp
Bouton boutons[3] = {
  {  60, 340, 200, 100, "/bmg1.bmp", "/BE-C.mp3", "BE-CALAVI",  0x1E90FF },
  { 300, 340, 200, 100, "/bmg3.bmp", "/BE-F.mp3", "BE-FSS",     0x32CD32 },
  { 540, 340, 200, 100, "/bmg4.bmp", "/BE-G.mp3", "BE-GODOMEY", 0xFF6347 }
};
// Champs : x, y, largeur, hauteur, chemin image, chemin audio, label, couleur
```

Concevez `logo.bmp` de manière à laisser la zone du bas (y > 320) libre pour les boutons,
ou adaptez les coordonnées selon votre design.

---

## Référence des GPIO

### Écran (bus parallèle RGB — fixe, ne pas modifier)

| Signal | GPIO                                    |
| ------ | --------------------------------------- |
| D0–D15 | 8,3,46,9,1,5,6,7,15,16,4,45,48,47,21,14 |
| HSYNC  | 39                                      |
| VSYNC  | 41                                      |
| PCLK   | 42                                      |
| DE     | 40                                      |

---

### Tactile GT911 (I2C)

| Signal | GPIO                        |
| ------ | --------------------------- |
| SDA    | 19                          |
| SCL    | 20                          |
| RST    | 38                          |
| INT    | Non connecté (mode polling) |

---

### Audio I2S → MAX98357

| Signal     | GPIO |
| ---------- | ---- |
| BCLK       | 0    |
| WS (LRC)   | 45   |
| DOUT (DIN) | 17   |

> ⚠️ **Ces GPIO I2S proviennent de la communauté** — Sunton n’a pas publié de schéma officiel pour cette version.
> GPIO45 est aussi utilisé par l’écran (`pin_d11`).
> Si aucun son ne sort, vérifiez les connexions avec un multimètre ou consultez le schéma communautaire sur esp3d.io.

---

### Carte SD (SPI 1-bit)

| Signal | GPIO |
| ------ | ---- |
| CLK    | 12   |
| MOSI   | 11   |
| MISO   | 13   |

---

### Rétroéclairage

| Signal | GPIO | Notes                                 |
| ------ | ---- | ------------------------------------- |
| BL     | 2    | PWM 20 kHz (réduction du bruit audio) |

---

## Réduction du bruit

Deux techniques sont utilisées pour limiter le bruit audio :

**1. PWM du rétroéclairage à 20 kHz**
→ évite les interférences avec l’alimentation audio :

```cpp
ledcSetup(0, 20000, 8);
ledcAttachPin(2, 0);
ledcWrite(0, 255);
```

**2. Volume limité à 12–15 / 21**
→ au-delà, l’amplificateur classe D introduit de la distorsion à cause de l’alimentation partagée.

---

## Machine d’état de l’application

```
┌─────────────┐   appui bouton   ┌──────────────┐
│  ETAT_LOGO  │ ───────────────► │ ETAT_LECTURE │
│             │                  │              │
│ logo.bmp    │ ◄─────────────── │ bmgX.bmp     │
│ + 3 boutons │   fin audio      │ + audio MP3  │
└─────────────┘                  └──────────────┘
```

---

## Limitations connues

* **Pas de lecture vidéo** — l’ESP32-S3 n’a pas de décodeur vidéo matériel
* **Conflit GPIO I2S** — GPIO45 partagé avec l’écran
* **Carte SD en mode 1-bit** — suffisant ici, mais le mode 4-bit serait plus rapide
* **Tactile en polling** — consommation CPU plus élevée (~10–20% vs <5%)

---

## Structure du projet

```
├── main.ino          ← code source complet
└── README.md         ← ce fichier
```

---

## Références

* LovyanGFX
* ESP32-audioI2S
* ESP32-8048S050 hardware reference — esp3d.io
* openHASP board page
* CircuitPython board page

---

## Licence

MIT — voir le fichier [LICENSE](LICENSE) pour plus de détails.

