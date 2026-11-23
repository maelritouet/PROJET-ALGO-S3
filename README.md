# Simulateur d'entrepôt logistique — Projet C fin de semestre

Groupe : Mael, Este, Aurélie, Shahan, Thibault

Simulation, tick par tick, d'une flotte de robots qui préparent des commandes dans un entrepôt en grille à sens unique. Le programme lit 4 fichiers (carte, produits, robots, commandes), affecte les commandes aux robots, construit leurs trajets, les fait avancer sans collision, gère leur batterie et produit des statistiques et un score.

---

## 1. Vue d'ensemble

```
          scenario/ (carte.txt, produits.csv, robots.csv, commandes.csv)
                                  │
                                  ▼
                        ┌───────────────────┐
                        │  LECTURE + CARTE  │  parse, valide, remplit l'Entrepot
                        └─────────┬─────────┘
                                  ▼
   ┌───────────────────────────────────────────────────────────────┐
   │                     BOUCLE DE SIMULATION (1 tick)             │
   │                                                               │
   │  1. Arrivée des commandes   (à venir → en attente / refusée)  │
   │  2. Lancement + affectation (commande → station P → robot)    │
   │  3. Batterie                (besoin de recharge ?)            │
   │  4. Décision de chaque robot (prochaine case du trajet)       │
   │  5. Résolution des conflits + déplacements                    │
   │  6. Vérificateur            (collisions, sens interdits)      │
   │  7. Historique              (snapshot du tick)                │
   │  8. Affichage / stats       (ou rien en mode sans affichage)  │
   │  9. Détection de blocage    (20 ticks sans mouvement)         │
   └───────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                   Statistiques de fin + ligne SCORE
```

Toutes les données dont le nombre n'est connu qu'à la lecture vivent dans **nos propres structures** (listes, pile, file, tableau 2D dynamique). Aucun tableau de taille fixe pour les robots, produits ou commandes.

---

## 2. Arborescence

```
projet/
├── Makefile                 all, clean, test  (-Wall -Wextra -Werror)
├── README.md
├── IA.md
├── src/
│   ├── config.h             constantes : dossier du scénario, affichage on/off,
│   │                        graine, ticks max, vitesse
│   ├── types.h              tous les énumérés + structures (contrat commun)
│   │
│   ├── liste.c/.h           liste simplement chaînée
│   ├── liste_triee.c/.h     insertion à la bonne place
│   ├── liste_double.c/.h    liste doublement chaînée
│   ├── pile.c/.h            pile (bac du robot)
│   ├── file.c/.h            file (files d'attente, commandes standard)
│   │
│   ├── carte.c/.h           tableau 2D dynamique, accès aux cases,
│   │                        case de prélèvement, case d'accès
│   ├── lecture.c/.h         lecture + validation des 4 fichiers
│   │
│   ├── trajet.c/.h          construction des trajets (règle de base, raccourcis)
│   ├── commandes.c/.h       arrivée, priorités, affectation, tournées
│   ├── simulation.c/.h      boucle de tick, déplacements, conflits
│   ├── station.c/.h         files d'attente des stations P et C
│   ├── batterie.c/.h        consommation, décision de recharge
│   ├── verificateur.c/.h    contrôle indépendant de chaque tick
│   ├── stats.c/.h           statistiques, rapport, ligne SCORE
│   │
│   ├── affichage.c/.h       rendu terminal ANSI
│   ├── interaction.c/.h     pause, pas à pas, vitesse, détails, quitter
│   ├── historique.c/.h      300 derniers ticks, avancer / reculer
│   │
│   └── main.c
├── generateur/
│   └── generateur.c         programme séparé : produit des scénarios valides
├── tests/
│   └── tests.c              make test
└── scenarios/               01_exemple … 04_grand, erreurs/, nos scénarios
```

---

## 3. Dépendances entre modules

```
                        types.h  ◄── tout le monde
                           │
        structures de données (liste, liste_triee, liste_double, pile, file)
                           │
                     carte + lecture
                           │
      ┌──────────┬─────────┼──────────┬───────────┐
    trajet   commandes   station   batterie   historique
      └──────────┴────┬────┴──────────┘           │
                  simulation ◄────────────────────┘
                      │
       ┌──────────────┼───────────────┐
  verificateur      stats       affichage + interaction
  (lit l'état,      (score)
   ne modifie rien)
```

Règles :
- **`types.h` et les `.h` d'abord.** Chacun publie les signatures de son module avant d'écrire le `.c`. Les autres codent avec ces signatures sans attendre l'implémentation.
- **Le vérificateur ne dépend pas de `simulation`.** Il compare l'état avant/après d'un tick et compte les erreurs (deux robots sur une case, déplacement contre la flèche). C'est une exigence du sujet.
- Pas de variables globales : tout passe par un pointeur `Entrepot *`.

---

## 4. Qui utilise quelle structure

| Structure | Utilisée pour |
|---|---|
| Liste simple | robots, catalogue produits (recherche par référence), lignes de commande, trajet d'un robot (on retire en tête) |
| Liste triée | commandes à venir (par tick d'arrivée), commandes express en attente (par échéance), arrêts d'une tournée (ordre de passage) |
| Liste double | historique des 300 derniers ticks |
| Pile | bac du robot (fragile = pris en dernier) |
| File | file d'attente de chaque station (3 max), commandes standard en attente |
| Tableau 2D dynamique | carte |

---

## 5. Répartition

| Membre | Modules | Ce que ça couvre |
|---|---|---|
| **Mael** | `trajet`, `commandes`, `simulation` | Construction des trajets (règle de base puis raccourcis), boucle de tick, déplacements et priorités aux croisements, robots lourds, affectation robot ↔ commande, tournées multi-lignes (poids, fragile, découpage) |
| **Este** | `liste`, `liste_triee`, `liste_double`, `pile`, `file`, `tests/` | Toutes les structures de données + `make test` (ajout, insertion triée, suppression, recherche, libération) |
| **Aurélie** | `affichage`, `interaction`, `historique` | Rendu ANSI (carte, robots, stations, files), commandes interactives, navigation dans l'historique en pause |
| **Shahan** | `carte`, `lecture`, `generateur/` | Lecture des 4 fichiers, détection d'erreurs (fichier + ligne + cause), vérification de la structure de la carte, rejet des 6 scénarios `erreurs/`, générateur de scénarios |
| **Thibault** | `station`, `batterie`, `verificateur`, `stats` | Files d'attente des stations, choix de la station P, recharge et décision de partir en charge, vérificateur indépendant, statistiques, mode sans affichage, ligne SCORE |

Points d'interface à surveiller :
- **Este ↔ tout le monde** : ses `.h` sont à figer en premier.
- **Shahan → Mael / Thibault** : la carte doit fournir la case de prélèvement d'une étagère et la case d'accès d'une station.
- **Mael ↔ Thibault** : `batterie` a besoin de la longueur d'un trajet (`trajet`) ; `simulation` appelle `station` pour entrer / sortir / faire la queue.
- **Aurélie ↔ Mael** : l'historique enregistre un snapshot à la fin de chaque tick de `simulation`.

---

## 6. Planning

| Date | Objectif |
|---|---|
| **23/11** (point d'étape 1) | Architecture et répartition validées, `types.h` + `.h` des structures |
| **30/11** | Structures de données testées, lecture des fichiers, affichage de la grille, un robot qui suit les flèches |
| **07/12** (point d'étape 2) | Niveau 1 complet, premier jet du document d'architecture |
| **13/12, 23h59** | Rendu : code, Makefile, README, architecture (PDF), IA.md, générateur, scénarios |
| **14/12** | Soutenance |

Avant la soutenance, chacun relit le code des autres au moins une fois : n'importe qui peut être tiré au sort pour la modification en direct.

---

## 7. Compilation et utilisation

```
make          # compile le simulateur et le générateur
make test     # lance les tests
make clean
./entrepot
```

Les paramètres (dossier du scénario, affichage, graine, ticks max, vitesse) se changent dans `src/config.h`, puis on recompile.
