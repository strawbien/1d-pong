# CLAUDE.md : pong 1D (fork d'ogermer/1d-pong)

Ce fichier guide Claude Code sur ce dépôt. Langue de travail : **français**.

## Contexte

- Jeu de **pong 1D à deux joueurs** sur un ruban LED WS2812B piloté par un ESP32.
- Projet personnel de François, joué en famille. **V1 = prototype sur breadboard**, puis V2 intégrée dans un boîtier.
- Code d'origine : https://github.com/ogermer/1d-pong (licence MIT). Le fork doit **conserver la notice de copyright MIT** d'origine dans le README et la licence.
- Le projet d'origine est prévu pour une WEMOS D1 Mini ESP32 et **55 LEDs**. Ce fork vise **144 LEDs** et une **ESP32 DevKit V1 30 broches**.

## Règles de travail pour Claude

1. **Distinguer les faits des hypothèses** : toute affirmation non vérifiée dans le code, la datasheet ou un test doit être signalée comme *hypothèse*.
2. **Commencer par les vérifications simples** (compilation, valeurs de config, câblage) avant toute explication ou refonte complexe.
3. **Compiler après chaque modification** avec `pio run`. **Ne flasher la carte (`pio run -t upload`) que sur demande explicite**, car le matériel est branché sur le poste de François.
4. Faire des changements petits et ciblés, avec un commit par sujet et des messages de commit en français.
5. **Consigner chaque décision** dans la section « Journal des décisions » en bas de ce fichier (date, décision, raison).
6. Garder la structure d'ogermer (machine à états dans `main.cpp`, animations auto-enregistrées). Écrire les commentaires du nouveau code en français, sans réécrire les commentaires anglais existants.
7. Code non bloquant : **jamais de `delay()`** dans la boucle de jeu ou les animations. Utiliser `millis()`.

## Mise en place du fork (à faire une fois)

```bash
# 1. Forker ogermer/1d-pong sur GitHub (bouton Fork), puis :
git clone https://github.com/<mon-compte>/1d-pong.git
cd 1d-pong
git remote add upstream https://github.com/ogermer/1d-pong.git
git checkout -b v1-esp32dev-144leds
```

Outils : VS Code et l'extension PlatformIO IDE, ou la CLI `pio`.

- Compiler : `pio run`
- Flasher : `pio run -t upload` (maintenir BOOT si l'upload échoue)
- Moniteur série : `pio device monitor` (115200 bauds)
- Récupérer les évolutions d'origine : `git fetch upstream && git merge upstream/main`

## Matériel V1 (tel que câblé)

| Élément | Détail |
|---|---|
| Microcontrôleur | ESP32 DevKit V1 30 broches (ESP-32D, USB-C), sur carte d'extension à bornier |
| Ruban | BTF-LIGHTING WS2812B, 1 m, **144 LEDs**, 5 V, ordre des couleurs GRB (*hypothèse, à confirmer au test*) |
| Level shifter | 74AHCT125 (DIP-14) : broche 2 ← GPIO 5, broche 3 → 330 Ω → DIN ; broches 1 et 7 à GND, broche 14 à +5 V |
| Alimentation | 5 V par connecteur USB-C à bornier (chargeur USB ou powerbank) ; condensateur de 1000 µF sur l'arrivée ; fusible réarmable ~2,5 A prévu |
| Boutons | 2 boutons d'arcade 24 mm non lumineux, entre la broche et GND (pull-up interne) |

### Broches

| Fonction | GPIO | Sérigraphie DevKit |
|---|---|---|
| Données ruban | 5 | D5 |
| Bouton gauche | 17 | TX2 |
| Bouton droit | 18 | D18 |
| LED bouton gauche / droit | 25 / 26 | D25 / D26 : **non câblées en V1** |

Broches à éviter pour de futurs ajouts : GPIO 0, 2, 12 et 15 (démarrage), 34 à 39 (entrée seule, sans pull-up), 1 et 3 (UART USB).

## Adaptations V1 à réaliser (tâches initiales)

- [ ] `platformio.ini` : ajouter un environnement `[env:esp32dev]` (`board = esp32dev`) et en faire l'environnement par défaut (`default_envs = esp32dev`), en gardant `wemos_d1_mini32` pour compatibilité.
- [ ] `include/config.h` : `NUM_LEDS 144`.
- [ ] `include/config.h` : `BRIGHTNESS` de 255 → **environ 60** pour les tests.
- [ ] `main.cpp`, dans `setup()` après `addLeds` : ajouter `FastLED.setMaxPowerInVoltsAndMilliamps(5, 1500);`. Une constante `MAX_MILLIAMPS` dans `config.h` serait préférable. Le but est de rester sous le courant du chargeur USB et du fusible.
- [ ] Remettre la vitesse à l'échelle : avec 2,6 fois plus de LEDs, `BALL_INITIAL_DELAY_MS 60` rend une traversée environ 2,6 fois plus lente. Point de départ : **~23 ms** (60 × 55 / 144). *Hypothèse, à régler en jouant.*
- [ ] Zones de frappe : `ZONE_SIZE_START` et `ZONE_SIZE_MIN` (10 et 5 pour 55 LEDs) → essayer **~26 et ~13**. *Hypothèse, à régler en jouant.*
- [ ] Ajouter un **mode test matériel** activable par un flag de build (par exemple `-D HW_TEST` dans un environnement `[env:esp32dev_test]`). Il allume les 5 premières LEDs en rouge, vert et bleu à faible luminosité, et affiche l'état des boutons dans le moniteur série (1 au repos, 0 appuyé). C'est la première vérification à faire tourner.

## Points de vigilance dans le code d'origine

- `Animation::update(CRGB* leds, uint8_t numLeds)` et plusieurs boucles utilisent `uint8_t`. **Limite : 255 LEDs.** Ça convient pour 144, mais il faudra passer en `uint16_t` pour un ruban plus long (2 m à 144 LEDs/m = 288 LEDs).
- `BRIGHTNESS 255` sans limite de puissance : risque de chute de tension ou de coupure de la source USB sur les animations lumineuses.
- Les broches LED des boutons (25 et 26) sont pilotées en PWM même si rien n'y est branché. C'est sans danger. Les désactiver proprement si une V2 n'a pas de boutons lumineux.
- Les animations s'enregistrent automatiquement : un fichier `.cpp` dans `src/animations/` avec `REGISTER_ANIMATION(Classe, "Nom")`. Gabarit : `src/animations/_template.cpp.example`.

## Feuille de route (idées, à prioriser avec François)

1. **V1 jouable** : adaptations ci-dessus, test matériel, réglage de la vitesse et des zones.
2. **Son** : buzzer piézo passif (frappe, point, victoire), sur une broche libre et sûre.
3. **Plusieurs jeux sur le même support** : ajouter un état `MENU` à la machine à états (appui long pour changer de jeu, couleur par jeu) et faire de chaque jeu un module. Candidats :
   - pong (existant) ;
   - course de LEDs, inspirée d'[ArminJo/OpenledRace](https://github.com/ArminJo/OpenledRace) ;
   - tir à la corde ;
   - jeu de réflexes.
4. **Meilleurs scores** persistants avec `Preferences` (NVS), plutôt que l'EEPROM émulée.
5. **Page web de configuration** via le Wi-Fi de l'ESP32 : choix du jeu, difficulté, luminosité.
6. **V2** : carte miniature (ESP32-C3 SuperMini ou Seeed XIAO ESP32-C6, avec un environnement PlatformIO dédié), boutons lumineux de 60 mm via MOSFET, soudure sur plaque à pastilles, boîtier. Plus tard, un TWANG avec un MPU6050, sur le modèle de [bdring/TWANG32](https://github.com/bdring/TWANG32).

## Journal des décisions

- 2026-10-24 : pong 1D, ruban WS2812B 5 V 144 LEDs/m sur 1 m, ESP32 déjà possédé, 2 boutons d'arcade.
- 2026-10-25 : V1 à coût minimal, boutons de 24 mm non lumineux.
- 2026-11-11 : ESP32 sur carte d'extension à bornier, breadboard réservée au 74AHCT125 ; bouton gauche sur TX2 (GPIO 17).
- 2026-11-11 : reprise du code d'ogermer via un fork local, développement avec Claude Code.
