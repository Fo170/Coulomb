# Coulomb

Compteur de charge électrique en **coulombs** pour ESP8266 — **intégrateur pur**.

Version **2.0.1** · Licence **GPL-3.0** · Auteur : Olivier FOURNET (olivier.fournet@free.fr) · Dépôt : [github.com/Fo170/Coulomb](https://github.com/Fo170/Coulomb)

> ## Cette librairie ne fait que COMPTER
>
> ```
> ΔC = I · Δt
> ```
>
> Aucune physique : ni rendement faradique, ni Peukert, ni état de charge, ni
> capacité batterie, ni dégagement gazeux. Ces politiques appartiennent au
> **projet** qui utilise la librairie.
>
> ```cpp
> Coulomb c;
> c.addMeasurement(I * eta);     // le projet pondère s'il le souhaite
> ```
>
> Dépendance unique : `<Arduino.h>`.

## Contenu

| Header | Classe | Rôle |
|---|---|---|
| `Coulomb.h` | `Coulomb` | intégration et cumul de la charge en coulombs (1 C = 1 A·s) |

> ### 📦 Dépôt séparé
>
> Ce header a vécu dans `Fo170/CoulombsAh` jusqu'à la 3.0.1, aux côtés de
> `AmpereHeure.h` et `CoulombsAh.h`. Les trois étaient **indépendants** (aucun
> include croisé) : ils sont désormais dans trois dépôts distincts.
>
> | Dépôt | Header | Classe | Version |
> |---|---|---|---|
> | [Fo170/CoulombsAh](https://github.com/Fo170/CoulombsAh) | `CoulombsAh.h` | `Ah_GD` | 3.1.0 |
> | **Fo170/Coulomb** | **`Coulomb.h`** | **`Coulomb`** | **2.0.1** |
> | [Fo170/AmpereHeure](https://github.com/Fo170/AmpereHeure) | `AmpereHeure.h` | `AmpereHeure` | 2.0.1 |

---

## ⚠️ Précision — pourquoi préférer `CoulombsAh` pour un cumul long

`Coulomb` cumule en **`float32`**. Un `float32` ne peut pas incrémenter une
grande valeur par petits pas : l'ULP croît avec la magnitude.

| `total` | ULP float32 | Incrément 0,1 A × 0,1 s | Conservé ? |
|---|---|---|---|
| 1 C | 1,19e-07 C | 0,01 C | oui |
| 36 000 C (10 Ah) | 3,8e-03 C | 0,01 C | oui (2,6 ULP) |
| **360 000 C (100 Ah)** | **3,05e-02 C** | 0,01 C | **non** |
| **3 276 000 C (910 Ah)** | **2,44e-01 C** | 0,01 C | **non (24× sous l'ULP)** |

Conséquence : au-delà de quelques dizaines d'ampères-heures, chaque échantillon
est **arrondi à zéro** et le compteur se fige silencieusement.

`Coulomb` reste parfaitement adapté à un **compteur remis à zéro
périodiquement** (par exemple pour mesurer un delta entre deux synchronisations),
où la valeur reste petite. Pour un **odomètre** qui grandit indéfiniment,
utiliser [`Fo170/CoulombsAh`](https://github.com/Fo170/CoulombsAh) : accumulateur
`int64` à deux étages, exact à toute magnitude.

---

## Utilisation

```cpp
#include "Coulomb.h"

Coulomb coulomb;

void setup() { coulomb.begin(); }

void loop() {
    float I = lireCourant();                 // A, > 0 = charge
    coulomb.addMeasurement(I);               // timestamp = micros()
    // ou, si l'intervalle est connu :
    coulomb.addMeasurementWithInterval(I, 0.1f);
}
```

### Injection d'une charge déjà intégrée

```cpp
float addCharge(float coulombsDelta);        // signé : < 0 = décharge
```

Utile quand le projet calcule lui-même son incrément (rendement, Peukert,
gazage…) sans passer par un courant. Ne touche ni `getCurrent()` ni l'horloge
interne — aucune notion de temps. Les NaN/Inf sont rejetés.

---

## API

| Méthode | Rôle |
|---|---|
| `begin(initialCoulomb = 0, resetTimestamp = true)` | Initialise |
| `addMeasurement(I)` | Intègre avec `micros()` |
| `addMeasurement(I, microsTimestamp)` | Intègre avec horodatage explicite |
| `addMeasurementWithInterval(I, dt_s)` | Intègre sur un intervalle donné (0 < Δt ≤ 3600 s) |
| `addCharge(dC)` | Ajoute une charge déjà intégrée |
| `reset()` / `reset(bool)` | Remet à zéro |
| `setTotalCoulombs(C)` | Fixe la **base** (persistance) ; l'intégration s'y ajoute |
| `getCoulombs()` | Coulombs depuis le dernier reset |
| `getTotalCoulombs()` | Base + intégration |
| `getAmpereHours()` | `getTotalCoulombs() / 3600` |
| `getAmpereHoursSinceReset()` | `getCoulombs() / 3600` |
| `getCoulombsPerSecond()` / `getAverageCoulombsPerSecond()` | Débit instantané / lissé |
| `getCurrent()` / `getAverageCurrent()` | Courant brut / lissé |
| `getLastInterval()` / `getAverageInterval()` | Intervalles |
| `getSampleCount()` | Nombre d'échantillons |
| `getMaxCoulombs()` / `getMinCoulombs()` | Bornes (si `enableMinMaxTracking(true)`) |
| `setFilterAlpha(α)` | Coefficient EMA du courant (0–1) |
| `enableMinMaxTracking(bool)` | Active le suivi min/max |
| `setAverageCurrent(I)` | Force le courant moyen lissé |
| `printDebug()` | État interne sur le port série |

### Garde-fous

- **Δt hors `]0 ; 3600]`** rejeté (pas d'intégration).
- **`I` = NaN ou ±Inf** rejeté : un seul échantillon corrompu (trame I²C)
  empoisonnerait définitivement le cumul.
- **Débordement de `micros()`** (~70 min) géré dans `calculateInterval()`.
- **Filtrage exponentiel** : `filterAlpha` par défaut 0,99 ; plus proche de 1 =
  lissage plus fort. La première mesure ne fait qu'amorcer l'état.

---

## Persistance

Déléguée à [`Fo170/Persistance`](https://github.com/Fo170/Persistance)
(EEPROM, FRAM ou LittleFS). Le header **ne l'inclut pas** — c'est le projet qui
choisit le support.

```cpp
// restauration
float saved;
if (persistance.loadFloat("coulomb", &saved)) coulomb.setTotalCoulombs(saved);

// sauvegarde
persistance.saveFloat("coulomb", coulomb.getTotalCoulombs());
```

⚠️ `setTotalCoulombs()` définit une **base** sur laquelle l'intégration
s'ajoute — il n'écrase pas la valeur courante.

---

## Migration

| Avant | Après |
|---|---|
| `#include "Coulomb.h"` (depuis `Fo170/CoulombsAh`) | `#include "Coulomb.h"` (depuis `Fo170/Coulomb`) — **aucun changement de code** |
| `getChargeEfficiency(I)` / `setEfficiencies()` / `setGassingModel()` / `setBatteryVoltage()` | supprimés en 2.0.0 — la physique est dans le projet |
| `setPeukertExponent()` / `variableEfficiency()` | supprimés (code mort depuis la 1.3.1) |
| `getLastInterval()` | **conservé**, mais attention : c'est le `dt` du dernier `addMeasurement()`. Préférer passer le `dt` explicitement. |

Pour passer à l'accumulateur exact, voir la table de migration de
[`Fo170/CoulombsAh`](https://github.com/Fo170/CoulombsAh).

---

## Sources et références

Consultées le **2026-10-19**. Toutes répondent HTTP 200 à cette date.

| Sujet | Source | Ce qui en est tiré |
|---|---|---|
| Rendement faradique | [Wikipedia — Faraday efficiency](https://en.wikipedia.org/wiki/Faraday_efficiency) | η = rapport des charges échangées sur un cycle — grandeur physique, **pas** un facteur d'intégration |
| Loi de Peukert | [Wikipedia — Peukert's law](https://en.wikipedia.org/wiki/Peukert%27s_law) | `C/C₀ = (I₀/I)^(k−1)` : la capacité **disponible décroît** avec le courant. Ne s'applique jamais comme rendement |
| Représentation IEEE 754 (ULP) | [Wikipedia — Unit in the last place](https://en.wikipedia.org/wiki/Unit_in_the_last_place) | L'ULP d'un float32 croît avec la magnitude ⇒ justifie de préférer un accumulateur `int64` pour un cumul long |

---

## Historique synthétique des versions

| Version | Changement majeur |
|---|---|
| **2.0.1** | `getLastInterval()` n'était pas mis à jour par `addMeasurementWithInterval()` → pas de temps du modèle 100× trop petit. |
| **2.0.0** | **RUPTURE** — la librairie devient un **intégrateur pur**. Toute la physique (η, Peukert, SoC, gazage) est supprimée et migre dans le projet. `addCharge()` ajouté. |
| 1.3.1 | η découplé de Peukert (constant par direction) ; garde `isfinite` ; `reset()` complet ; `setPeukertExponent` borné `[1.0, 2.0]` |
| 1.3.0 | η appliqué à l'intégration (auparavant seul `AmpereHeure` en avait un) |
| 1.2.0 | Peukert dynamique + rendement η variable |
| 1.1.0 | Persistance déléguée à `Fo170/Persistance` (EEPROM/FRAM/LittleFS) |
| 1.0.0 | Version initiale |

> 📦 **Séparation des dépôts (2026-10-19)** : `Coulomb.h` était dans
> `Fo170/CoulombsAh` aux côtés de `AmpereHeure.h` et `CoulombsAh.h`. Les trois
> headers étant indépendants, chacun a désormais son dépôt. L'historique de ce
> fichier est conservé dans `Fo170/CoulombsAh` jusqu'au tag `v3.0.1`.
