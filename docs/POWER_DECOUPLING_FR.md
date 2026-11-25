# ⚡ Découplage de l'alimentation de la matrice NeoPixel - LED-Garland-Anim

*Fiche de décision (matériel). Valable pour la matrice WS2812B 8x8 sur toutes les cartes supportées.*

## Décision

**Un condensateur électrolytique 1000µF / 16V entre le 5V et la masse de la matrice WS2812B, au plus près de la matrice, et une résistance de 470Ω en série sur la ligne de données près du premier pixel, sont documentés comme protections OPTIONNELLES.**

Statut : **optionnel**. Le montage d'origine a fonctionné sans pendant tout Noël l'an dernier, sans panne signalée. Ce sont des assurances bon marché (environ 1 €) contre une famille de pannes difficiles à diagnostiquer, à ajouter en cas de resets ou de scintillement, si tu augmentes la luminosité, ou si tu changes l'alimentation ou le câblage.

---

## 1. Pourquoi une matrice WS2812B consomme par à-coups

| Donnée | Valeur |
|--------|--------|
| Courant par pixel en blanc plein | ~60mA (3 x 20mA) |
| Matrice complète, blanc plein | 64 x 60mA = **3,84A** |
| Luminosité par défaut du firmware | `matrixBrightness = 128` (~50 %), soit jusqu'à ~1,9A dans le pire cas |
| Animation typique (quelques pixels allumés, surtout colorés) | 0,1 à 0,8A |

Ce qui compte n'est pas la moyenne mais la **variation**. Une animation comme `FIREWORKS`, `CONFETTI`, `DISCO_BALL` ou `PRESENTS` passe de presque noir à beaucoup de pixels vifs d'une image à la suivante. Chaque pixel allume ses sorties en quelques microsecondes quand la nouvelle trame est verrouillée (appel `show()`), donc la charge saute de plusieurs centaines de mA à plusieurs A très brutalement. Un pixel piloté en PWM tire aussi son courant par impulsions à ~400Hz, donc même une image fixe a une charge pulsée.

## 2. Ce que voit l'alimentation : le problème

Une alimentation et ses fils ne répondent pas instantanément :

1. **Inductance et résistance des fils.** Les jumpers Dupont et les rails de breadboard sont fins et longs. Un échelon de courant `di` provoque une chute `V = R x I + L x di/dt`. Avec 0,3Ω de jumpers et de breadboard, une charge de 2A coûte déjà 0,6V avant tout transitoire.
2. **Temps de réaction de la régulation.** La boucle de contrôle d'une alim à découpage réagit en quelques centaines de µs à quelques ms. Pendant ce temps, seule sa capacité de sortie tient la tension, et elle n'est pas dimensionnée pour un échelon de 3A.
3. **Rail partagé.** L'ESP32, le TB6612FNG, le capteur PIR et la matrice partagent le même 5V. Une chute causée par la matrice est donc vue par tout le reste.

Résultat : **une brève chute de tension (avec oscillations) sur le 5V à chaque fois que l'image change beaucoup**.

## 3. Ce que fait le découplage

Un condensateur est un petit réservoir d'énergie local. Quand la matrice réclame soudain du courant, le condensateur le fournit à quelques millimètres, sans que le courant ait à traverser les longs fils, et l'alimentation a le temps de rattraper. Quand la charge retombe, le condensateur absorbe le surplus.

Ordre de grandeur : la chute pendant que le condensateur fournit un échelon de courant vaut

```
ΔV = I x Δt / C
```

Pour un échelon de 1A pendant 100µs avec C = 1000µF : ΔV = 1 x 0,0001 / 0,001 = **0,1V**. Sans condensateur, le même échelon produit des chutes de 0,5V ou plus rien que dans le câblage.

Deux niveaux de découplage existent, avec des rôles différents :

| Composant | Rôle | Où |
|-----------|------|----|
| **1000µF électrolytique (réservoir)** | Fournit les gros échelons relativement lents (µs à ms), lisse les impulsions PWM à 400Hz | Entre 5V et GND à l'entrée de la matrice |
| **100nF céramique (haute fréquence)** | Absorbe les fronts très rapides (ns à µs). La plupart des matrices WS2812B en ont déjà un par pixel, rien à ajouter côté matrice | Déjà sur la matrice. À ajouter près de la carte ESP32 si elle redémarre malgré le condensateur principal |

### Résistance sur la ligne de données

La résistance de 470Ω sur la ligne de données (jusqu'ici optionnelle, désormais recommandée) limite le pic de courant dans l'entrée data du premier pixel à la mise sous tension et amortit les rebonds sur les fronts du signal. Placez-la près de la **matrice** (premier pixel), pas près de l'ESP32. Toute valeur entre 300Ω et 500Ω convient selon les recommandations habituelles.

## 4. Conséquences de l'absence de condensateur

Aucune n'est garantie. Un câblage court et épais avec une bonne alim 5A peut masquer le problème, surtout à luminosité 128. Elles deviennent probables avec un câblage sur breadboard, des jumpers fins, une alim bas de gamme ou une luminosité élevée.

| Symptôme | Mécanisme |
|----------|-----------|
| **Resets aléatoires de l'ESP32 / boucle de redémarrage** | La chute du 5V traverse le régulateur 3,3V de la carte (un AMS1117 typique exige ~4,4V en entrée pour rester régulé). En dessous, le 3,3V s'affaisse, le détecteur de brownout de l'ESP32 se déclenche et la puce redémarre. Typique au lancement d'une animation avec beaucoup de pixels vifs, ou quand le WiFi émet en même temps (pics de ~300mA). |
| **WiFi qui décroche, interface web injoignable, échecs MQTT/OTA** | La radio est la première à souffrir d'un 3,3V bas ou bruité. Une mise à jour OTA interrompue par un brownout peut obliger à reflasher en USB. |
| **Scintillement, couleurs fausses, premiers pixels qui bavent** | Le signal data est référencé à la masse. Si GND rebondit de quelques centaines de mV en charge, le signal 3,3V n'est plus lu proprement par le WS2812B (son seuil haut est déjà de 0,7 x VDD = ~3,5V sous 5V, donc quasiment aucune marge avec une sortie ESP32 en 3,3V). |
| **Luminosité qui « respire » ou baisse en bout de matrice** | Chute de tension le long du chemin d'alimentation. Quand la tension baisse, les puces bleue et verte (tension directe plus élevée) perdent leur éclat en premier, et les blancs tirent vers le rouge/orange. |
| **Scintillement de la guirlande (TB6612FNG)** | La guirlande et la matrice partagent le 5V. Les échelons de charge de la matrice modulent la luminosité de la guirlande. |
| **Faux déclenchements du PIR** | Le HC-SR501 est très sensible au bruit d'alimentation et se déclenche souvent quand le rail est bruité. Cela peut lancer des animations toutes seules, qui créent à leur tour des échelons de charge. |
| **Contrainte à la mise sous tension du premier pixel** | Brancher un 5V déjà présent sur une matrice déchargée et non découplée provoque un pic de tension pouvant abîmer l'entrée data du premier pixel. C'est la raison principale du condensateur dans les recommandations Adafruit pour les NeoPixels. |
| **Durée de vie réduite** | Pics répétés et brownouts sollicitent l'alim, le régulateur et les pixels. Peu probable de causer une panne immédiate, mais cause plausible de pixels morts à terme. |

## 5. Ce que le condensateur ne corrige pas

- Une **alimentation sous-dimensionnée** (moins que le courant crête demandé). Gardez 3A minimum, 5A idéalement, ou baissez la luminosité.
- Des **fils fins ou trop longs.** Utilisez des fils courts et épais (22AWG ou mieux) pour le 5V et GND vers la matrice, en liaison directe plutôt que par les rails de la breadboard si possible.
- Un **problème de niveau logique 3,3V vers 5V.** Si les premiers pixels bavent alors que l'alimentation est parfaitement stable, ajoutez un adaptateur de niveau (74AHCT125 ou 74HCT245) entre la broche data de l'ESP32 et la matrice.

## 6. Check-list d'installation

- [ ] Condensateur **1000µF, tension nominale 10V ou plus** (16V recommandé), faible ESR de préférence
- [ ] **Polarité** : la bande sur le boîtier marque la patte négative, à relier à GND. Une inversion peut le faire éclater
- [ ] Le brancher **directement à l'entrée 5V/GND de la matrice**, pattes les plus courtes possible, pas côté alimentation
- [ ] Raccorder **GND en premier** et l'alimentation en dernier, et éviter de brancher la matrice sur un rail déjà sous tension
- [ ] Résistance 470Ω en série sur la ligne de données, près de la matrice
- [ ] Optionnel : un second condensateur (470 à 1000µF) à l'entrée 5V de l'ESP32 si la carte redémarre encore
- [ ] Optionnel : garder la luminosité du firmware à 128 ou moins, sauf alimentation 5A

## 7. Récapitulatif des protections optionnelles

| Protection | Utile ? | Commentaire |
|------------|---------|-------------|
| Condensateur 1000µF à l'entrée de la matrice | Oui | Très recommandé si tu vois des symptômes (section 4) |
| Résistance 330-470Ω sur DATA | Oui | Recommandation Adafruit classique |
| Diode en série sur l'alim 5V de la matrice | Optionnel | Abaisse le VDD de la matrice pour mieux lire le signal data 3,3V (voir ci-dessous) |
| Diode de polarité | Optionnel | Seulement comme sécurité supplémentaire |

### Diode en série sur l'alimentation 5V (optionnel)

Le WS2812B lit un niveau haut au-dessus de 0,7 x VDD, soit ~3,5V sous 5V, ce qu'une sortie ESP32 en 3,3V atteint de justesse. Une diode en série sur le 5V de la matrice chute de 0,3 à 0,7V, la matrice voit donc environ 4,3 à 4,7V et son seuil descend vers 3,0 à 3,3V. Le signal 3,3V est alors lu avec plus de marge.

- On parle souvent de « diode sur le premier pixel ». **Sur une matrice 8x8, on ne peut pas isoler le premier pixel**, car les 64 pixels partagent le même rail : la diode se met donc sur toute l'alimentation de la matrice.
- **Prendre une Schottky 3A** (1N5822 ou SB360). Une 1N4001 est donnée pour 1A et chute plus d'1V à fort courant, ce qui ne convient pas ici.
- Placer le **condensateur 1000µF après la diode**, côté matrice.
- Garder l'ESP32 et les autres charges alimentés **avant** la diode, sur le 5V non modifié.
- La chute varie avec le courant, c'est donc un pansement. Un adaptateur de niveau (74AHCT125 ou 74HCT245) est le vrai remède et coûte à peu près pareil.

### Diode de polarité (optionnel)

Seulement si tu veux une sécurité supplémentaire contre une inversion de l'alimentation. Une Schottky en série fait les deux rôles (protection contre l'inversion et abaissement du seuil VDD), au prix de 0,3 à 0,4V sur le rail de la matrice.

## Voir aussi

- [HARDWARE_GUIDE_FR.md](HARDWARE_GUIDE_FR.md) : liste des composants et dimensionnement de l'alimentation
- [TROUBLESHOOTING_FR.md](TROUBLESHOOTING_FR.md) : diagnostic des resets et du scintillement
- [PIN_MAPPING_FR.md](PIN_MAPPING_FR.md) : broche data de la matrice par carte
