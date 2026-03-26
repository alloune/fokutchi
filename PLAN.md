# Fokutchi — Plan Directeur

> App de focalisation gamifiée style Tamagotchi × Forest, multijoueur, avec monétisation ads + IAP.
> Ce document trace TOUTES les décisions : ce qui a été proposé, retenu, écarté, et pourquoi.

---

## Table des matières

1. [Vision & Concept](#1-vision--concept)
2. [Game Design Document (GDD)](#2-game-design-document-gdd)
3. [Architecture Technique](#3-architecture-technique)
4. [Phases d'implémentation](#4-phases-dimplémentation)
5. [Décisions & Alternatives écartées](#5-décisions--alternatives-écartées)
6. [Questions ouvertes](#6-questions-ouvertes)

---

## 1. Vision & Concept

### Pitch
"Concentre-toi dans le monde réel, tes créatures progressent dans le monde virtuel."

### Core Loop
```
FOCUS (timer réel) → ÉNERGIE DE FOCUS → ACTIONS CRÉATURES → RÉCOMPENSES → MOTIVATION → FOCUS
```

### Inspirations
| Jeu | Ce qu'on prend | Ce qu'on ne prend PAS |
|-----|---------------|----------------------|
| Forest | Timer de focus, récompense pour concentration | Arbres statiques (on veut des créatures vivantes) |
| Tamagotchi | Créatures à nourrir/faire évoluer, attachement émotionnel | Mort permanente (trop punitif pour un outil de productivité) |
| Idle RPG | Progression pendant le focus, donjons, évolution | Gameplay actif (le joueur ne doit PAS toucher son tel) |
| Clash Royale | PvP, classements, saisons | Gameplay temps réel (incompatible avec le focus) |

### Proposition de valeur unique
- Le focus **N'EST PAS** du temps mort — c'est le moteur du jeu
- Les créatures sont **autonomes** pendant le focus (idle mechanics)
- Le joueur **choisit** ce que font ses créatures avant de lancer le focus
- Plus le focus est long/de qualité → meilleures récompenses

---

## 2. Game Design Document (GDD)

### 2.1 Les Créatures (Fokutchis)

#### Attributs de base
| Attribut | Description |
|----------|-------------|
| PV (Points de Vie) | Santé de la créature |
| Énergie | Consommée par les actions pendant le focus |
| Force | Dégâts en combat/donjon |
| Endurance | Durée de focus supportée avant fatigue |
| Concentration | Bonus pour les sessions longues et ininterrompues |
| Résistance | Capacité à encaisser les donjons profonds / combats difficiles |
| Vivacité | Bonus pour les sessions multiples et variées |
| Persévérance | Bonus de streak et de régularité quotidienne |
| Intelligence | Vitesse d'apprentissage de compétences |
| Charisme | Bonus en récolte de ressources |

#### Système d'évolution
```
Œuf → Bébé → Juvénile → Adulte → Éveillé → Légendaire
```
- Chaque stade nécessite un cumul de **minutes de focus**
- L'évolution débloque de nouvelles compétences et slots d'équipement

#### Évolutions divergentes (CRITIQUE)

L'évolution est déterminée par le **pattern de focus du joueur**, pas juste le temps cumulé.

| Pattern de focus | Stats favorisées | Archétype d'évolution | Exemple |
|-----------------|-----------------|----------------------|---------|
| Sessions longues continues (2h+) | Concentration, Résistance | Gardien / Titan | Créature massive, stoïque, tanky |
| Plusieurs sessions courtes (3×1h) | Vivacité, Persévérance | Éclaireur / Assassin | Créature rapide, agile, multi-hit |
| Mix équilibré | Stats équilibrées | Sage / Paladin | Créature polyvalente |
| Focus nocturne fréquent | Intelligence, mystique | Sorcier / Ombre | Affinité infernale (donjon bas) |
| Focus matinal régulier | Charisme, lumière | Prêtre / Séraphin | Affinité divine (donjon haut) |

**Le joueur ne choisit PAS directement l'évolution** — elle émerge de ses habitudes de focus.
C'est la mécanique centrale qui rend chaque créature unique et personnelle.

> Implications techniques : le serveur doit tracker non seulement la durée cumulée mais aussi
> la distribution temporelle des sessions (heure, durée, fréquence, streaks).

#### Compétences (Skills)

Chaque compétence est définie par un modèle `skills` et est **restreinte par archétype/classe**.
Un tank ne pourra pas apprendre un sort de soin ou d'accélération — chaque skill a une liste d'archétypes autorisés.

- Acquises par focus cumulé dans une activité spécifique
- Arbre de compétences par créature, filtré selon l'archétype de la créature
- Restriction de classe : chaque skill définit les archétypes qui peuvent l'apprendre
- Exemples :
  - "Bouclier de concentration" → Gardien, Paladin
  - "Frappe rapide" → Éclaireur, Assassin
  - "Drain vital" → Sorcier, Ombre
  - "Bénédiction" → Prêtre, Séraphin
  - "Récolte rapide" → Tous
  - "Pillage de donjon" → Éclaireur, Assassin, Ombre

#### Collection
- **3 créatures starter** au lancement
- Chaque starter donne accès à un **arbre d'évolution branché** : 3 branches × 3 branches = **27 archétypes Légendaires** possibles
- Système de rareté : Commun, Rare, Épique, Légendaire, Mythique
- Obtention : œufs trouvés en donjon, récompenses de saison, événements
- **Système de gacha** pour les œufs : OUI, mais **impossible d'acheter des œufs avec de l'argent réel**. L'argent investi n'impacte jamais le gacha.
- Chaque créature a un **design unique** + variantes (skins)

### 2.2 Système de Focus

#### Lancement d'une session
1. Le joueur choisit la **durée** (10min, 25min, 45min, 1h, 2h, personnalisé — **minimum 10 minutes**)
2. Le joueur **assigne des missions** à ses créatures :
   - 🏕️ Récolte de ressources (bois, pierre, cristaux, nourriture)
   - ⚔️ Exploration de donjon
   - 📖 Apprentissage de compétence
   - 💤 Régénération (récupère PV/énergie)
   - 🛡️ Défense de la base (PvP défensif)
3. Le joueur verrouille son téléphone et se concentre

#### Pendant le focus
- Les créatures exécutent leurs missions en **temps réel**
- Animation idle visible si le joueur revient (mais pas d'interaction possible)
- **Bonus de streak** : sessions consécutives sans interruption = multiplicateur
- **Interruption** = réduction du multiplicateur de qualité (pas de perte directe de ressources, mais les récompenses finales sont diminuées)

#### Qualité du focus
| Qualité | Condition | Multiplicateur |
|---------|-----------|---------------|
| Parfait | 0 interruption, durée complète | ×2.0 |
| Bon | 1-2 consultations rapides (<5s) | ×1.5 |
| Moyen | 3-5 consultations | ×1.0 |
| Faible | >5 consultations ou abandon | ×0.5 |

#### Récompenses de focus
- **Énergie de Focus (EF)** : monnaie principale, gagnée à chaque session
- **Ressources** : selon la mission assignée
- **XP Créature** : progression vers l'évolution
- **XP Joueur** : débloque des fonctionnalités (donjons, PvP, etc.)

### 2.3 Économie du jeu

#### Monnaies
| Monnaie | Obtention | Utilisation |
|---------|-----------|-------------|
| Énergie de Focus (EF) | Focus sessions | Actions créatures, crafting, améliorations |
| Cristaux | Donjons rares, classements, achats IAP | Cosmétiques premium, accélérateurs |
| Ressources (bois, pierre, etc.) | Missions de récolte | Construction, crafting |

#### Crafting & Construction
- **Maison/Habitat** : améliorable, influe sur la régénération des créatures
- **Ateliers** : débloquent des objets à crafter (potions, équipements)
- **Décorations** : cosmétiques pour l'habitat (IAP + EF)

### 2.4 La Tour — Donjons (PvE)

#### Concept visuel
- **Style pixel art** — une tour verticale avec les étages visibles autour du niveau actuel
- **Pas de combat graphique complexe** :
  - L'équipe entre dans l'étage → animation de secousse/tremblement de la tour
  - Nuage de poussière / éclairs à l'intérieur
  - Résumé textuel rapide de ce qui s'est passé (dégâts, loot, événements)
- Minimaliste mais satisfaisant — l'accent est sur le résultat, pas l'animation

#### Structure bidirectionnelle

La tour se parcourt dans **deux directions**, chacune avec son identité propre :

```
    ☁️ Étage +∞  SOMMET DIVIN
    ⛪ Étage +20  Paradis — Boss Archange
    ✨ Étage +15  Ciel étoilé — Mobs célestes
    🕊️ Étage +10  Nuages — Mobs angéliques
    🌤️ Étage +5   Ciel bas — Mobs lumineux
    ━━━━━━━━━━━━━ ÉTAGE 0 — ENTRÉE ━━━━━━━━━━━━━
    🌑 Étage -5   Souterrains — Mobs sombres
    🔥 Étage -10  Cavernes — Mobs chaotiques
    💀 Étage -15  Abysses — Mobs démoniaques
    👹 Étage -20  Enfer — Boss Seigneur Démon
    🌋 Étage -∞   FOND INFERNAL
```

| Direction | Ambiance | Type de mobs | Type de loot | Œufs trouvables |
|-----------|----------|-------------|-------------|-----------------|
| **Vers le haut** (Divin) | Lumineux, céleste, sacré | Anges, séraphins, esprits | Loot sacré, stable, prévisible | Créatures divines/lumineuses |
| **Vers le bas** (Infernal) | Sombre, chaotique, brûlant | Démons, ombres, bêtes | Loot chaotique (haute variance : rien OU jackpot) | Créatures infernales/sombres |

**Pas de bien/mal** — les deux directions sont neutres. C'est juste une orientation de gameplay :
- Divin = stable, défensif, régulier
- Infernal = chaotique, high-risk/high-reward, offensif

#### Mécanique de progression
- **Chaque session de focus** = exploration d'étages (plus le focus est long → plus d'étages)
- **Boss tous les 5 étages** → loot rare + chance d'œuf spécial
- Le joueur **choisit la direction** avant de lancer le focus
- La progression est **persistante** (on reprend où on s'est arrêté)

#### Équipe de donjon
- 1 à 3 créatures par expédition
- Composition stratégique (tank, DPS, support)
- Les compétences des créatures déterminent l'issue
- **Affinité** : une créature divine sera plus forte en montant, une infernale en descendant

#### Progression de difficulté
```
Étages ±1-10 : Normal (accessible dès le début)
Étages ±11-20 : Difficile (nécessite créatures évoluées)
Étages ±21-50 : Héroïque (nécessite stratégie + équipement)
Étages ±51+ : Mythique (endgame, nécessite créatures Légendaires)
```

### 2.5 PvP & Multijoueur

#### Combat PvP (asynchrone)
- Le joueur configure son **équipe de défense** (3 créatures + stratégie)
- Pendant un focus, il peut envoyer ses créatures **attaquer** un autre joueur
- Le combat se résout automatiquement (idle)
- Résultat visible à la fin du focus

#### Animation PvP (minimaliste)
- Les deux groupes de créatures se rencontrent face à face
- **Nuage de poussière** classique style cartoon (secousses, étoiles, éclairs)
- Le nuage se dissipe → résumé rapide du résultat
- Pas d'animation de combat détaillée — l'émotion vient du résultat, pas de l'animation

#### Classements
| Classement | Récompense |
|------------|-----------|
| Top 1% | Skin légendaire + cristaux ×5 |
| Top 10% | Cristaux ×3 + œuf rare |
| Top 25% | Cristaux ×2 |
| Top 50% | Cristaux ×1 |

#### Guildes / Clans (EN EXPLORATION)

> **Statut** : Concept à creuser. Le défi principal est que demander à des gens de focus
> en même temps est contraignant. Mais c'est aussi une opportunité pour des cas d'usage
> spécifiques (sessions d'exam, BU en groupe, sprints de travail en équipe).

**Piste 1 — Focus synchrone (session de groupe)**
- Un membre lance une "session de groupe" avec un créneau horaire
- Les autres rejoignent et focus ensemble
- Bonus de synergie si tout le monde termine
- **Use case idéal** : révisions d'exam, sessions BU, sprints d'équipe
- **Risque** : contraignant, difficulté de coordination

**Piste 2 — Contributions asynchrones (raid boss)**
- Un raid boss a une barre de PV énorme
- Chaque session de focus de chaque membre retire des PV au boss
- Deadline de 7 jours pour battre le boss
- Pas besoin de focus en même temps, juste de contribuer régulièrement
- **Avantage** : flexible, chacun son rythme

**Piste 3 — Hybride**
- Raids asynchrones en semaine + événements synchrones le week-end
- Les événements synchrones donnent des bonus plus importants

> **Décision** : À trancher après la Phase 5 (PvP), quand on aura des retours utilisateurs
> sur les mécaniques multijoueur de base.

#### Saisons
- Durée : 4 semaines
- Thème unique (environnement, créatures spéciales, cosmétiques)
- Battle Pass gratuit + premium
- Reset du classement PvP

### 2.6 Monétisation

#### Publicités (obligatoire, non-intrusif)
| Type | Moment | Bonus |
|------|--------|-------|
| Rewarded Video | Après un focus | ×1.5 sur les récompenses de cette session |
| Rewarded Video | Avant un donjon | +1 vie supplémentaire |
| Rewarded Video | Quotidien | Coffre bonus |
| Interstitial | Entre les menus (max 1/5min) | — |

**Règle d'or** : JAMAIS de pub pendant un focus. Le focus est sacré.

#### Achats In-App (cosmétiques uniquement, PAS de pay-to-win)
| Catégorie | Exemples | Prix indicatif |
|-----------|----------|---------------|
| Skins de créature | Variantes visuelles, effets | 1.99€ - 4.99€ |
| Habitats/Maisons | Thèmes (forêt, espace, océan) | 2.99€ - 9.99€ |
| Environnements | Fonds de focus (montagne, plage, etc.) | 0.99€ - 2.99€ |
| Bannières de profil | Cadres, titres, badges | 0.99€ - 1.99€ |
| Battle Pass Premium | Contenu saisonnier exclusif | 4.99€/saison |
| Pack de Cristaux | Pour cosmétiques uniquement | 0.99€ - 19.99€ |

**Règle absolue** : Aucun achat ne donne un avantage compétitif. Cosmétiques et confort uniquement.

#### Modèle de revenus projeté
```
Revenus = Ads (60%) + Battle Pass (25%) + IAP cosmétiques (15%)
```

### 2.7 Onboarding & Rétention

#### Premier lancement
1. Choix d'un œuf starter (3 options, chacune mignonne)
2. Tutoriel de focus (5 minutes)
3. Première éclosion de l'œuf
4. Mission guidée de récolte
5. Première construction dans l'habitat

#### Boucles de rétention
| Fréquence | Mécanique |
|-----------|-----------|
| Quotidien | Récompense de connexion, mission quotidienne, énergie des créatures |
| Hebdomadaire | Défis hebdo, reset du classement intermédiaire |
| Mensuel | Nouvelle saison, événements spéciaux |
| Long terme | Évolutions, collection, progression de guilde |

---

## 3. Architecture Technique

### 3.1 Stack retenu

| Couche | Technologie | Justification |
|--------|-------------|---------------|
| **Frontend** | React Native + Expo | Cross-platform, productivité dev, large écosystème |
| **Rendu 2D** | react-native-skia | Animations fluides, shaders, particules, rendu custom |
| **Animations** | react-native-reanimated + Skia | 60fps, animations thread UI séparé |
| **State Management** | Zustand | Léger, performant, immutable-friendly |
| **Navigation** | Expo Router | File-based routing, deep links |
| **Backend** | Supabase (PostgreSQL + Auth + Realtime) | Temps réel, auth, DB, stockage, pas de serveur à gérer |
| **Game Logic Server** | Supabase Edge Functions (Deno) | Validation anti-triche, résolution des combats |
| **Queue / Cache** | Redis + BullMQ | Queue de résolution de combats, matchmaking, classements temps réel |
| **Push Notifications** | Expo Notifications + FCM/APNs | Rappels de focus, résultats de combat |
| **Ads** | Google AdMob (react-native-google-mobile-ads) | Standard industrie, rewarded videos |
| **IAP** | react-native-iap | Achats in-app cross-platform |
| **Analytics** | Mixpanel ou Amplitude | Suivi rétention, funnels, A/B tests |
| **CI/CD** | EAS Build + EAS Submit | Build cloud, soumission stores automatisée |

### 3.2 Architecture serveur

```
┌─────────────────────────────────────────────────────┐
│                    Client (React Native)             │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Skia    │  │  Game    │  │  Network Layer   │  │
│  │  Canvas  │  │  State   │  │  (Supabase SDK)  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│                Supabase Backend                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Auth    │  │  Realtime│  │  Edge Functions   │  │
│  │  (JWT)   │  │  (WS)   │  │  (Game Logic)    │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  PostGIS │  │  Storage │  │  Row Level       │  │
│  │  (DB)    │  │  (Assets)│  │  Security (RLS)  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 3.3 Anti-triche

**Critique** pour un jeu compétitif :
- Le **timer de focus** est validé côté serveur (start/end timestamps)
- Les récompenses sont **calculées côté serveur** (Edge Functions)
- Le client envoie : `{ session_id, duration, interruptions[] }`
- Le serveur valide : durée cohérente, pas de manipulation de timestamps
- Les combats PvP sont **résolus côté serveur**
- Rate limiting sur les appels API

### 3.4 Modèle de données (schéma simplifié)

```sql
-- Joueurs
players: id, username, avatar, level, xp, energy_focus, crystals, created_at

-- Créatures
creatures: id, player_id, species_id, name, level, xp, stage, hp, energy,
           strength, endurance, intelligence, charisma, skin_id

-- Compétences (définition)
skills: id, name, description, type(active|passive), cooldown,
        allowed_archetypes[], element, tier, unlock_stage,
        base_power, scaling_stat

-- Compétences (acquises par créature)
creature_skills: creature_id, skill_id, level, xp

-- Sessions de focus
focus_sessions: id, player_id, started_at, ended_at, planned_duration,
                actual_duration, quality, interruptions, rewards_json,
                hour_of_day, is_continuous, streak_count

-- Profil de focus (pour évolutions divergentes)
focus_profiles: player_id, creature_id,
                total_long_sessions, total_short_sessions,
                avg_session_duration, longest_streak,
                morning_ratio, night_ratio,
                concentration_score, vivacity_score,
                perseverance_score, resistance_score

-- Missions
missions: id, session_id, creature_id, type, result_json

-- Tour (Donjons bidirectionnels)
tower_runs: id, player_id, direction(up|down), floor_reached,
            creatures[], status, loot_json, boss_defeated

-- PvP
pvp_battles: id, attacker_id, defender_id, result, season_id, elo_change

-- Guildes
guilds: id, name, level, members_count
guild_members: guild_id, player_id, role

-- Inventaire
inventory: id, player_id, item_id, quantity

-- Achats
purchases: id, player_id, product_id, price, platform, purchased_at
```

---

## 4. Phases d'implémentation

### Phase 0 — Fondations (Semaines 1-2)
**Objectif** : Projet bootable, CI/CD, navigation de base

- [ ] Init Expo + React Native
- [ ] Setup react-native-skia
- [ ] Setup Supabase (auth, DB, migrations)
- [ ] Navigation de base (Expo Router)
- [ ] Écrans placeholder : Home, Focus, Créatures, Profil
- [ ] CI/CD avec EAS Build
- [ ] Design system (couleurs, typographie, composants de base)

### Phase 1 — Core Focus Loop (Semaines 3-5)
**Objectif** : Le joueur peut faire un focus et être récompensé

- [ ] Timer de focus (countdown, background timer)
- [ ] Détection d'interruption (app state changes)
- [ ] Calcul de qualité de focus
- [ ] Récompenses de base (EF, XP)
- [ ] Affichage résumé post-focus
- [ ] Validation serveur du focus
- [ ] Première créature (statique, pas encore animée)
- [ ] Assignation de mission simple avant focus

### Phase 2 — Créatures & Animations (Semaines 6-9)
**Objectif** : Les créatures sont vivantes et évoluent

- [ ] Système d'évolution (stages, XP requis)
- [ ] Animations Skia des créatures (idle, happy, tired, evolving)
- [ ] Système de compétences (arbre, apprentissage)
- [ ] Habitat de base (vue de la maison)
- [ ] Système de récolte de ressources
- [ ] Régénération des créatures
- [ ] 5-10 espèces de créatures designées
- [ ] Collection et bestiaire

### Phase 3 — Économie & Crafting (Semaines 10-12)
**Objectif** : Boucle économique complète

- [ ] Système de ressources (bois, pierre, cristaux, nourriture)
- [ ] Crafting d'objets (potions, équipements)
- [ ] Amélioration de l'habitat
- [ ] Boutique in-game (EF + cristaux)
- [ ] Récompenses quotidiennes
- [ ] Missions quotidiennes et hebdomadaires

### Phase 4 — Donjons PvE (Semaines 13-16)
**Objectif** : Contenu PvE jouable

- [ ] Génération procédurale de donjons
- [ ] Système de combat idle (résolution automatique)
- [ ] Composition d'équipe (1-3 créatures)
- [ ] Système de loot
- [ ] Boss de donjon
- [ ] Difficulté progressive
- [ ] Animations de combat Skia

### Phase 5 — PvP & Multijoueur (Semaines 17-20)
**Objectif** : Compétition entre joueurs

- [ ] Système de matchmaking (Elo-based)
- [ ] Combat PvP asynchrone
- [ ] Configuration d'équipe de défense
- [ ] Classements (quotidien, hebdo, saison)
- [ ] Récompenses de classement
- [ ] Supabase Realtime pour les notifications de combat
- [ ] Profils publics

### Phase 6 — Social & Guildes (Semaines 21-24)
**Objectif** : Communauté et engagement social

- [ ] Création/gestion de guildes
- [ ] Raids coopératifs
- [ ] Chat de guilde (basique)
- [ ] Classement de guilde
- [ ] Système d'amis
- [ ] Échange de ressources entre amis

### Phase 7 — Monétisation (Semaines 25-27)
**Objectif** : Revenus

- [ ] Intégration AdMob (rewarded videos)
- [ ] Pub post-focus pour bonus
- [ ] Pub pré-donjon pour vie supplémentaire
- [ ] IAP : skins de créatures
- [ ] IAP : thèmes d'habitat
- [ ] IAP : environnements de focus
- [ ] IAP : bannières et cadres
- [ ] Battle Pass (gratuit + premium)
- [ ] Validation des achats côté serveur

### Phase 8 — Saisons & Events (Semaines 28-30)
**Objectif** : Contenu vivant

- [ ] Système de saisons (4 semaines)
- [ ] Créatures saisonnières exclusives
- [ ] Événements temporaires
- [ ] Donjons spéciaux saisonniers
- [ ] Cosmétiques saisonniers

### Phase 9 — Polish & Lancement (Semaines 31-34)
**Objectif** : Qualité production

- [ ] Optimisation performances (60fps constant)
- [ ] Tests de charge serveur
- [ ] Localisation (FR, EN, JP, KR minimum)
- [ ] Accessibilité
- [ ] Onboarding complet
- [ ] Analytics et A/B testing
- [ ] Beta fermée → Beta ouverte → Lancement
- [ ] ASO (App Store Optimization)

---

## 5. Décisions & Alternatives écartées

### Stack technique

| Option | Verdict | Raison |
|--------|---------|--------|
| **React Native + Skia** | ✅ RETENU | Cross-platform, bon pour 2D, productivité dev, écosystème mature |
| Flutter + Flame | ❌ Écarté | Flame est moins mature que Skia pour les animations custom, écosystème de packages gaming plus limité |
| Unity + C# | ❌ Écarté | Overkill pour un jeu principalement UI-driven, package lourd, complexité inutile |
| Kotlin natif + Compose | ❌ Écarté | Pas cross-platform, devrait refaire tout pour iOS |
| Godot + GDScript | ❌ Écarté | Moins bon pour les parties "app" (profil, social, IAP), communauté mobile plus petite |

### Backend

| Option | Verdict | Raison |
|--------|---------|--------|
| **Supabase** | ✅ RETENU | PostgreSQL, Auth, Realtime, Edge Functions, RLS — tout-en-un, gratuit pour commencer, scalable |
| Firebase | ❌ Écarté | NoSQL (Firestore) moins adapté aux requêtes complexes de classement/matchmaking, vendor lock-in Google |
| Custom Node.js + PostgreSQL | ❌ Écarté | Trop de travail d'infra pour un MVP, on peut migrer plus tard si besoin |
| Appwrite | ❌ Écarté | Moins mature que Supabase, communauté plus petite |

### Game Design

| Décision | Verdict | Raison |
|----------|---------|--------|
| **Mort permanente des créatures** | ❌ Écarté | Trop punitif pour un outil de productivité, risque de frustrer et décourager le focus |
| **Créatures fatiguées (pas mortes)** | ✅ RETENU | Conséquence légère mais motivante, la créature se régénère avec du focus |
| **Combat temps réel** | ❌ Écarté | Incompatible avec le concept de focus (le joueur ne doit PAS toucher son tel) |
| **Combat idle/asynchrone** | ✅ RETENU | Se résout pendant le focus, résultat visible après |
| **Pay-to-win** | ❌ Écarté | Tue la compétitivité et la motivation, les joueurs détestent ça |
| **Cosmétiques only** | ✅ RETENU | Standard industrie éthique, les joueurs acceptent et achètent volontiers |
| **Pub pendant le focus** | ❌ Écarté | Contre-productif, détruit la proposition de valeur |
| **Pub rewarded optionnelle** | ✅ RETENU | Le joueur choisit de regarder une pub pour un bonus, non-intrusif |
| **Animations de combat détaillées** | ❌ Écarté | Trop coûteux en dev, pas le core du jeu. Animations symboliques suffisantes (secousse donjon, nuage de poussière PvP) |
| **Animations minimalistes + résumé** | ✅ RETENU | Nuage de poussière pour PvP, secousse de tour pour donjons + résumé textuel. Simple, efficace, peu coûteux |
| **Donjons horizontaux classiques** | ❌ Écarté | Remplacé par la Tour bidirectionnelle (haut/divin, bas/infernal) — plus original, plus de rejouabilité |
| **Tour bidirectionnelle Divin/Infernal** | ✅ RETENU | Vector art, choix de direction, loot différencié, pas de morale bien/mal |
| **Pixel art / Sprite-based** | ❌ Écarté | Fichiers binaires opaques, non modifiables par IA, chaque frame dessinée manuellement |
| **Vector-based (SVG/paths Skia)** | ✅ RETENU | Code lisible par IA, skeleton animation, itération rapide, IA-driven development |
| **Lottie (JSON After Effects)** | ❌ Écarté | JSON verbeux/fragile, nécessite outil externe, animations figées après export |
| **Skia natif (paths + reanimated)** | ✅ RETENU | Code TypeScript standard maîtrisé par l'IA, animations dynamiques liées aux stats à runtime |
| **Supabase seul (pas de Redis)** | ❌ Écarté | Risque de surcharge DB si beaucoup de combats simultanés en fin de focus |
| **Supabase + Redis/BullMQ** | ✅ RETENU | Queue de résolution de combats, matchmaking, classements — absorbe les pics de charge |
| **Évolution par choix du joueur** | ❌ Écarté | L'évolution doit ÉMERGER du pattern de focus, pas être choisie manuellement |
| **Évolution par pattern de focus** | ✅ RETENU | Sessions longues → Concentration/Résistance. Sessions courtes multiples → Vivacité/Persévérance. Rend chaque créature unique |

---

## 6. Questions ouvertes

> À discuter et trancher ensemble avant/pendant l'implémentation.

### Game Design
- [x] **Nom du jeu** : ~~FocusMon ? FocusPets ? ZenBeasts ?~~ → **Fokutchi** (décidé 2026-03-26)
- [x] **Nombre de créatures au lancement** : ~~10 ? 20 ? 30 ?~~ → **3 starters → 27 archétypes Légendaires** (arbre branché 3×3×3) (décidé 2026-03-26)
- [x] **Arbre d'évolution** : ~~Linéaire (A→B→C) ou branché~~ → **Branché, basé sur le pattern de focus** (décidé 2026-03-26)
- [x] **Timer minimum de focus** : ~~15min ? 10min ? 5min ?~~ → **10 minutes minimum** (décidé 2026-03-26)
- [x] **Pénalité d'interruption** : ~~Perte de ressources ?~~ → **Réduction du multiplicateur uniquement** (diminue les récompenses finales) (décidé 2026-03-26)
- [x] **Équilibrage PvP** : ~~Elo seul ou aussi basé sur le niveau ?~~ → **Elo seul**. Si un joueur envoie un bébé contre des Légendaires, c'est son choix. (décidé 2026-03-26)
- [x] **Limite de créatures actives** : ~~3 ? 5 ? Illimité ?~~ → **3 slots au lancement**, slots supplémentaires achetables. Les **équipes restent toujours composées de 3 créatures**. (décidé 2026-03-26)
- [x] **Système de gacha pour les œufs** : ~~Oui/Non ?~~ → **Oui, MAIS impossible d'acheter des œufs avec de l'argent réel**. L'argent n'impacte jamais le gacha. (décidé 2026-03-26)

### Technique
- [x] **Offline-first ou online-required** : ~~Le focus marche-t-il sans internet ?~~ → **Online obligatoire**. Le focus ne fonctionne pas sans connexion internet. (décidé 2026-03-26)
- [x] **Sprite-based ou vector-based** : ~~Sprite/Pixel ?~~ → **Vector-based (SVG/paths Skia)**. Les formes vectorielles sont du code, directement lisible et modifiable par l'IA. Skeleton-based animation = 1 modèle + transformations codées, au lieu de centaines de sprites dessinés. Approche IA-driven development. (décidé 2026-03-26)
- [x] **Lottie vs Skia natif** : ~~Lottie ?~~ → **Skia natif**. Le code est du TypeScript standard que l'IA maîtrise parfaitement, pas de JSON Lottie verbeux/fragile exporté depuis un outil externe. Permet des animations dynamiques liées aux stats des créatures à runtime. (décidé 2026-03-26)
- [x] **Supabase seul ou ajouter Redis** : ~~Supabase seul ?~~ → **Supabase + Redis (BullMQ)**. Supabase pour auth/data/realtime. Redis pour le matchmaking, les classements, et une **queue de résolution de combats** (évite la surcharge DB si beaucoup de combats simultanés en fin de focus). (décidé 2026-03-26)

### Business
- [x] **Plateforme de lancement** : ~~Android d'abord ou les deux ?~~ → **Android uniquement au lancement**. iOS seulement si ça marche. (décidé 2026-03-26)
- [x] **Modèle freemium** : ~~Quelles limitations ?~~ → **Aucune limitation pour les joueurs gratuits**. Ils auront simplement moins de récompenses et moins de cosmétiques. (décidé 2026-03-26)
- [x] **Âge cible** : ~~13-25 ? 18-35 ?~~ → **13-35 ans** (étudiants + jeunes travailleurs) (décidé 2026-03-26)
- [x] **Partenariats** : ~~Pomodoro apps, écoles, entreprises ?~~ → **Aucun partenariat pour le moment** (décidé 2026-03-26)

---

## Changelog

| Date | Modification | Décidé par |
|------|-------------|------------|
| 2026-03-26 | Création du plan initial | Claude + User |
| 2026-03-26 | Refonte donjons → Tour bidirectionnelle Divin/Infernal (pixel art) | User |
| 2026-03-26 | Animations minimalistes : secousse donjon + nuage poussière PvP + résumé texte | User |
| 2026-03-26 | Évolutions divergentes basées sur le pattern de focus (durée, fréquence, horaire) | User |
| 2026-03-26 | Ajout stats Concentration, Résistance, Vivacité, Persévérance | User |
| 2026-03-26 | Guildes/Raids marquées "EN EXPLORATION" — 3 pistes documentées | User + Claude |
| 2026-03-26 | Nom du jeu : Fokutchi | User |
| 2026-03-26 | 3 starters → 27 archétypes Légendaires (arbre branché 3×3×3) | User |
| 2026-03-26 | Timer minimum de focus : 10 minutes | User |
| 2026-03-26 | Interruptions = réduction du multiplicateur uniquement | User |
| 2026-03-26 | PvP Elo seul, pas de restriction par niveau | User |
| 2026-03-26 | 3 slots créatures au lancement, slots achetables, équipes de 3 | User |
| 2026-03-26 | Gacha pour les œufs, mais achat d'œufs impossible avec argent réel | User |
| 2026-03-26 | App online-only, focus sans internet impossible | User |
| 2026-03-26 | Lancement Android uniquement, iOS si succès | User |
| 2026-03-26 | Aucune limitation pour joueurs gratuits (moins de récompenses/cosmétiques) | User |
| 2026-03-26 | Âge cible : 13-35 ans | User |
| 2026-03-26 | Ajout modèle de données `skills` avec restriction par archétype | User |
| 2026-03-26 | Vector-based (SVG/paths Skia) au lieu de pixel art — approche IA-driven | User |
| 2026-03-26 | Skia natif au lieu de Lottie — code TS manipulable par IA, animations dynamiques | User |
| 2026-03-26 | Ajout Redis + BullMQ pour queue de combats, matchmaking et classements | User |
| | | |

---

> **STATUT** : VALIDÉ — PRÊT POUR IMPLÉMENTATION
>
> Toutes les questions ouvertes ont été tranchées (16/16).
>
> **Prochaine étape** :
> 1. Commencer Phase 0 (fondations)
