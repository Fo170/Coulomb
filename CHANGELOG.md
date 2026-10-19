# Changelog - Coulomb

## [Séparation des dépôts] - 2026-10-19

Le dépôt historique `Fo170/CoulombsAh` contenait **trois** headers
(`Coulomb.h`, `AmpereHeure.h`, `CoulombsAh.h`) alors qu'ils sont parfaitement
indépendants : aucun include croisé, dépendance unique `<Arduino.h>`.

Chacun est désormais dans **son propre dépôt**, avec son historique propre :

| Dépôt | Header | Classe | Version |
|---|---|---|---|
| [Fo170/CoulombsAh](https://github.com/Fo170/CoulombsAh) | `CoulombsAh.h` | `Ah_GD` | 3.1.0 |
| **Fo170/Coulomb** | **`Coulomb.h`** | **`Coulomb`** | **2.0.1** |
| [Fo170/AmpereHeure](https://github.com/Fo170/AmpereHeure) | `AmpereHeure.h` | `AmpereHeure` | 2.0.1 |

Ce dépôt démarre à la version **2.0.1**, qui était la dernière version publiée de
`Coulomb.h` dans l'ancien dépôt commun. L'historique antérieur (1.x → 2.x) reste
consultable dans `Fo170/CoulombsAh` jusqu'au tag `v3.0.1`.

Les entrées ci-dessous sont reprises de l'ancien CHANGELOG commun, **filtrées**
pour ne garder que ce qui concerne `Coulomb.h`.

## [2.0.1] - 2026-10-19

### 🐛 Fixed — `getLastInterval()` n'était jamais mis à jour par `addMeasurementWithInterval()`

`_lastIntervalSec` n'était assigné QUE dans `addMeasurement(current, micros)`.
Un appelant qui utilise `addMeasurementWithInterval()` laissait donc
`getLastInterval()` à sa valeur d'initialisation (**0,001 s**) pour toujours.

**Impact en aval.** Un consommateur qui lit `getLastInterval()` comme pas de
temps de modèle obtenait 0,001 s au lieu de 0,1 s, soit **100× trop petit**.
Toutes les constantes de temps étaient dilatées d'autant :

| Constante | Valeur voulue | Valeur effective |
|---|---|---|
| τ_pol | 12 s | **1200 s** |
| τ2 (diffusion) | 300 s | **8,3 h** |
| τ_hyst | 2000 s | **55 h** |

La polarisation et l'hystérésis ne s'établissaient donc jamais : `V_pol_state`
restait ≈ 0 et l'OCV calculé était biaisé.

**Correctif** : `addMeasurementWithInterval()` mémorise l'intervalle qu'il a
réellement utilisé.

### 🏷️ Divers

- Version `2.0.0` → `2.0.1` (`library.json` + macro)

## [2.0.0] - 2026-10-19

### 💥 BREAKING — `Coulomb` redevient un INTÉGRATEUR PUR

**Principe.** Une librairie de comptage doit savoir faire `Σ = Σ + I·Δt` et rien
d'autre. Toute **politique physique** — rendement faradique, Peukert, état de
charge, capacité batterie, autonomie, dégagement gazeux — est une décision du
**projet** qui utilise la librairie. Elle a donc été **supprimée**.

### ❌ API SUPPRIMÉE

| Méthode retirée | Remplacée par (côté projet) |
|---|---|
| `setEfficiencies(charge, discharge)` | `CoulombFct::setEfficiencyBase()` |
| `getChargeEfficiency(I)` | `CoulombFct::efficiencyFor(I)` |
| `getDischargeEfficiency(I)` | `CoulombFct::etaDis` |
| `getChargeEfficiencyBase()` / `getDischargeEfficiencyBase()` | `CoulombFct::etaChg` / `.etaDis` |
| `getAppliedEfficiency()` | `CoulombFct::etaApplied` |
| `setPeukertExponent()` / `getPeukertExponent()` | supprimés (code mort) |
| `setPeukertReferenceRate()` / `getPeukertReferenceRate()` | supprimés (code mort) |
| `variableEfficiency(base, I)` | supprimée (renvoyait `base` depuis la 1.3.1) |
| `setGassingModel(...)` | `CoulombFct::setGassingModel(...)` |
| `setBatteryVoltage(V, T)` | `CoulombFct::setBatteryVoltage(V, T)` |
| `getGassingThreshold()` ×2, `getGassingCurrent()`, `isGassingModelEnabled()`, `getGassingIRef()` | `CoulombFct::gassingThreshold()`, `.gasI` |

**Conséquence sur l'intégration** : `addMeasurementWithInterval()` n'applique
plus aucun rendement. `ΔC = I · Δt`, point. Le projet pondère le courant avant
l'appel (`addMeasurement(I * eta)`) ou injecte une charge déjà intégrée.

### ✅ API AJOUTÉE

```cpp
float addCharge(float coulombsDelta);   // charge déjà intégrée, signée
```

Ne touche ni `_current` ni `_lastMicros` : aucune notion de temps. Rejette
NaN/Inf. Sert aux projets qui calculent eux-mêmes leur incrément.

### 🧪 Ce qui est CONSERVÉ

`begin`, `addMeasurement` (×2), `addMeasurementWithInterval`, `addCharge`,
`reset` (×2), `setTotalCoulombs`, `getCoulombs`, `getTotalCoulombs`,
`getCoulombsPerSecond`, `getAverageCoulombsPerSecond`, `getCurrent`,
`getAverageCurrent`, `getLastInterval`, `getAverageInterval`, `getSampleCount`,
`getMaxCoulombs`, `getMinCoulombs`, `getAmpereHours`,
`getAmpereHoursSinceReset`, `setFilterAlpha`, `enableMinMaxTracking`,
`setAverageCurrent`, `printDebug`.

### ⚠️ Migration

```cpp
// AVANT (1.3.x)
Coulomb c;
c.setEfficiencies(0.90f, 1.00f);
c.setGassingModel(true, 2.35f, 11.7f, 0.45f, 6.0f, -0.005f);
c.setBatteryVoltage(V, T);
c.addMeasurement(I);

// APRÈS (2.0.0) — la physique est dans le projet
CoulombFct f;                       // src/Coulomb_fct.h du projet
f.setEfficiencyBase(0.90f, 1.00f);
f.setGassingModel(true, 2.35f, 11.7f, 0.45f, 6.0f, -0.005f);
f.addSample(I, V, T, dt_s);         // → applique η puis intègre

// Ou, sans le projet, en pondérant soi-même :
Coulomb c;
c.addMeasurement(I * eta);
```

### 🏷️ Divers

- `library.json` : `1.3.1` → `2.0.0`
- `__COULOMB_H_VERSION__` : `1.3.1` → `2.0.0`
- La v1.3.2 (rendement de charge variable) n'a **jamais été publiée** : son
  contenu a directement migré dans le projet côté application.
