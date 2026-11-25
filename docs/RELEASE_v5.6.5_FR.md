# Notes de version v5.6.5 (2026-11-25)

**Valide à partir de la version 5.6.5**

Version documentation uniquement. Aucun changement de comportement du firmware.

## 📚 Documentation

### Nouveau : découplage de l'alimentation de la matrice NeoPixel
Nouvelle note matérielle [POWER_DECOUPLING_FR.md](POWER_DECOUPLING_FR.md) ([English](POWER_DECOUPLING.md)) qui couvre :
- Pourquoi une matrice WS2812B 8x8 consomme par à-coups (jusqu'à 3,84A en blanc plein, ~1,9A à la luminosité par défaut de 128)
- Ce que voit le rail 5V et ce que fait un condensateur de 1000µF
- Conséquences en son absence : resets brownout de l'ESP32, WiFi/OTA qui décrochent, scintillement et premiers pixels qui bavent, guirlande qui scintille, faux déclenchements du PIR
- Ce que le condensateur ne corrige pas (alim sous-dimensionnée, fils fins, niveau logique 3,3V vers 5V)
- Check-list d'installation

### Protections optionnelles
Toutes sont **optionnelles** : le montage d'origine a fonctionné sans pendant tout Noël l'an dernier.

| Protection | Statut |
|------------|--------|
| Condensateur 1000µF à l'entrée de la matrice | Optionnel, recommandé en cas de resets ou de scintillement |
| Résistance 330-470Ω sur DATA | Optionnelle |
| Diode Schottky 3A en série sur le 5V de la matrice | Optionnelle (abaisse le seuil VDD pour le signal data 3,3V) |
| Diode de polarité | Sécurité supplémentaire optionnelle |

### Guide matériel
[HARDWARE_GUIDE.md](HARDWARE_GUIDE.md) / [HARDWARE_GUIDE_FR.md](HARDWARE_GUIDE_FR.md) : condensateur et résistance documentés comme optionnels, diode Schottky ajoutée aux composants optionnels, liens vers la nouvelle note.

---

## 🚀 Installation

```bash
git pull origin main
git checkout v5.6.5
pio run -e <your_board> -t upload
```

Le firmware étant inchangé, le flash ne sert qu'à mettre à jour le numéro de version affiché.

---

**Version précédente** : [v5.6.4](RELEASE_v5.6.4_FR.md)  
**Changelog complet** : [CHANGELOG_FR.md](../CHANGELOG_FR.md)
