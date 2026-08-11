# MP Optimizer

Application web mono-page qui répond à une seule question, pour Hypixel SkyBlock :

> **Avec mon budget, quel accessoire dois-je acheter en premier pour gagner le plus de Magical Power ?**

Le joueur saisit son pseudo Minecraft. L'app lit son sac à accessoires, calcule ce qu'il
lui manque, et classe les achats par **coût réel d'un point de Magical Power**.

Aucun backend, aucune clé API, aucune dépendance. Un seul fichier `index.html`,
déployable tel quel sur GitHub Pages.

---

## Ce que fait l'app

- **Classement des accessoires manquants** par ratio coins/MP (tri par défaut), prix, ou MP gagnée.
  Toutes les colonnes sont triables au clic.
- **Gain net, pas MP brute.** Les accessoires vont par lignes (Talisman → Ring → Artifact → Relic)
  et seul le tier le plus haut compte. Posséder le Bat Talisman (8 MP) et acheter le Bat Ring
  (12 MP) rapporte **4 MP**, pas 12.
- **Tableaux séparés selon la façon dont l'accessoire s'obtient** :
  - **Nouveaux accessoires** — vous ne possédez aucun tier de la ligne, chaque achat occupe un slot.
  - **Améliorations** — vous portez déjà un tier inférieur, l'achat le remplace et ne consomme
    aucun slot. Quand le sac est plein, c'est le seul tableau qui compte encore.
  - **Accessoires du Rift** — repérés par `origin === "RIFT"` dans le catalogue Hypixel. Ils
    s'achètent en motes à l'intérieur du Rift et non en coins : un ratio coins/MP n'aurait aucun
    sens pour eux. Ils sont donc exclus du classement principal **et de la sélection par budget**,
    et triés par MP gagnée. Toute la ligne Crux (Talisman → Ring → Artifact → Relic → Heirloom →
    Chronomicon → Celestial Starstone) en fait partie.
  - **À obtenir en jeu** — soulbound ou non échangeables, avec la méthode d'obtention déduite.
- **Budget.** Saisissez `50m`, `1.2b` ou `250000000` : l'app sélectionne gloutonnement par ratio
  la meilleure combinaison d'achats dans ce budget, en piochant dans les deux tableaux, et met
  les lignes retenues en évidence. **Un seul achat par ligne** : les gains d'une même ligne ne
  s'additionnent pas. Si le budget le permet, la sélection monte vers un tier supérieur de la
  même ligne en ne payant que la différence de prix.
- **Encadré récapitulatif** : MP actuelle, MP après achats, et multiplicateur de stats avant/après
  avec la formule officielle `29.97 × ln(0.0019 × MP + 1)^1.2`, plus le gain en pourcentage.
  C'est ce pourcentage qui montre quand la MP cesse d'être rentable.
- **Section séparée « à obtenir en jeu »** pour les accessoires soulbound ou non échangeables,
  avec la méthode d'obtention déduite (Rift, donjons, soulbound…).
- **Choix du profil** quand le joueur en a plusieurs, avec le profil actif présélectionné.
- Interface en **français**, bascule **anglais** en un clic. Thème sombre, responsive mobile.
- Dernier pseudo, langue et prix mis en cache dans `localStorage`.

---

## Sources de données retenues

Les quatre sources ont été testées avant l'écriture du code : code HTTP, absence de clé,
en-têtes CORS, et structure réelle de la réponse.

| Besoin | Source retenue | État |
|---|---|---|
| Catalogue des accessoires | `api.hypixel.net/v2/resources/skyblock/items` | ✅ 200, CORS `*`, sans clé |
| Profil et sac du joueur | `sky.coflnet.com/api/profile/{uuid}/{profileId}` | ✅ 200, CORS `*`, sans clé |
| Liste des profils | `sky.coflnet.com/api/profile/{uuid}` | ✅ 200, sans clé |
| Profil actif | `soopy.dev/api/v2/player_skyblock/{uuid}` | ✅ 200, CORS `*` (best-effort) |
| Prix bazaar | `api.hypixel.net/v2/skyblock/bazaar` | ✅ 200, CORS `*`, sans clé |
| Prix hôtel des ventes | `sky.coflnet.com/api/item/price/{tag}/current` | ✅ 200, CORS, sans clé |
| Pseudo → UUID | `playerdb.co`, repli `minetools.eu` puis `ashcon.app` | ✅ CORS `*` |

### Pourquoi pas SkyCrypt

`sky.shiiyu.moe/api/v2/profile/<pseudo>` était la source prévue. **Elle est fermée.**
Les tests menés :

- La page d'accueil `sky.shiiyu.moe/` répond `200`, mais tout `/api/*` répond `403`,
  depuis la même IP et le même navigateur — le blocage vise le chemin, pas le client.
- Un `fetch()` **same-origin, depuis sky.shiiyu.moe lui-même**, dans un vrai navigateur,
  renvoie également `403`.
- Le corps de la réponse est un `403 Forbidden` **nginx**, donc un refus du serveur d'origine
  et non un challenge anti-bot Cloudflare.
- L'instance de développement `cupcake.shiiyu.moe` est bloquée de la même façon.
- Le dépôt SkyCrypt v1 est archivé au profit de SkyCryptV2. Le site actuel n'appelle plus
  `/api/v2/` : il utilise des remote functions SvelteKit internes
  (`/_app/remote/<hash>/getCombined`) dont le hash change à chaque déploiement.

Conclusion : l'API publique de SkyCrypt a été retirée volontairement. S'appuyer sur son
endpoint interne aurait cassé au premier redéploiement, en contournant un blocage explicite.

Autres pistes testées et écartées : `soopy.dev` (fonctionne et expose le profil, mais
**supprime les inventaires** — pas de `talisman_bag`), `api.slothpixel.me` (521),
`skyblock.matdoes.dev` (500), `api.pixelic.de` (DNS mort), `hysky.de` (404).
`api.hypixel.net/v2/skyblock/profiles` répond `{"cause":"Missing API-Key header"}`.

**Coflnet** relaie le profil Hypixel brut, sans clé et avec CORS, y compris
`inventory.bag_contents.talisman_bag` — exactement ce qui manquait. C'est le rôle que
jouait SkyCrypt.

### Pourquoi Coflnet plutôt que la pagination des enchères

L'énoncé prévoyait en repli `api.hypixel.net/v2/skyblock/auctions` paginé. Mesuré :
**50 pages × 2,3 Mo ≈ 110 Mo** par rafraîchissement, inacceptable sur mobile. De plus les
enchères n'exposent pas l'identifiant SkyBlock de l'objet, seulement `item_bytes` (NBT
gzippé) : il faudrait décoder ~49 000 objets pour les rattacher au catalogue.

`sky.coflnet.com/api/item/price/{tag}/current` renvoie une centaine d'octets en ~120 ms pour
un objet précis. L'app n'interroge que les accessoires réellement manquants, avec un cache
`localStorage` de **10 minutes**.

`moulberry.codes/lowestbin.json`, la source historique, est morte (525).

### `/current` plutôt que `/bin`

`/bin` ne renvoie que les annonces **actives à la seconde près**. Un accessoire dont personne
ne vend d'exemplaire à cet instant en ressort à `lowest: 0` et paraît invendable, alors que son
prix de marché est parfaitement connu : le Junk Talisman ressortait vide quand `/current` le
donne à 500k, l'Artifact of Coins à 67,6m, le Small Fish Bowl à 110k. Sur un profil de test,
une dizaine d'accessoires étaient ainsi faussement marqués « aucune vente ».

`/current` renvoie en une seule requête `buy` (le prix payé) et `available` (le nombre
d'annonces en ligne), et donne la même valeur que `/bin` lorsque des annonces existent. Un
accessoire sans annonce active affiche donc son dernier prix connu préfixé de « ≈ », et reste
**exclu de la sélection par budget** puisqu'il n'est pas achetable dans l'immédiat.

### Quota de l'API

Coflnet plafonne à **100 requêtes par minute et par IP**, ce que son message d'erreur indique
explicitement : `API calls quota exceeded! maximum admitted 100 per 1m`. Un régulateur à
fenêtre glissante limite l'app à 85 par minute, et les accessoires au plus fort gain sont
demandés en premier pour que le haut du classement soit exploitable avant la fin du chargement.

Un `429` n'est jamais confondu avec « objet absent de l'hôtel des ventes » : il est retenté
quatre fois avec repli exponentiel en respectant `Retry-After`, n'est jamais mis en cache, et
s'affiche « Prix indisponible ». Comme les échecs ne sont pas mémorisés, relancer la recherche
ne redemande que les manquants et les complète en quelques secondes.

### Note sur `api.mojang.com`

L'endpoint officiel Mojang n'envoie **aucun en-tête CORS** et est donc inutilisable depuis
un navigateur. D'où le passage par des miroirs qui, eux, en envoient.

---

## Règles métier implémentées

**Magical Power par rareté** — uniquement la rareté, jamais les stats de l'objet :

| COMMON | UNCOMMON | RARE | EPIC | LEGENDARY | MYTHIC | SPECIAL | VERY SPECIAL |
|---|---|---|---|---|---|---|---|
| 3 | 5 | 8 | 12 | 16 | 22 | 3 | 5 |

**Cas particuliers codés explicitement :**

- **Hegemony Artifact** : MP doublée (32 au lieu de 16).
- **Rift Prism** : +11 MP une fois imbué chez Erihann, et ne prend aucun slot. Détecté via
  `rift.access.consumed_prism` — le prisme étant consommé à l'imbuage, il ne se trouve pas dans
  le sac. S'il est déjà imbué, les 11 MP sont comptées et l'objet disparaît des candidats ;
  sinon il apparaît dans la section Rift à 11 MP et sans slot.
- **Abicase** : +1 MP par tranche de 2 contacts de l'Abiphone, lu depuis
  `nether_island_player_data.abiphone.active_contacts`. Ignorée si le joueur n'a aucun contact.
- **Accessoires de donjon** : MP doublée en donjon uniquement. Le tableau affiche la valeur
  hors donjon et la valeur doublée sur un badge.
- **Objets recombobulés** : un accessoire recombobulé monte d'un cran de rareté, sa MP réelle
  suit. Pris en compte dans le calcul de la MP actuelle via `ExtraAttributes.rarity_upgrades`.
- **Le Recombobulator 3000 n'est jamais proposé** comme optimisation. Il monte un accessoire
  d'un seul cran pour un prix qui dépasse systématiquement celui d'un accessoire neuf offrant
  le même gain : c'est toujours le pire coût par MP. Il est mentionné uniquement dans l'aide.

**Construction des lignes d'accessoires** — trois règles successives :

1. **Préfixe + suffixe** : `Badge`/`Talisman` → `Ring` → `Artifact` → `Relic` → `Heirloom`.
   Couvre 55 lignes automatiquement. `Badge` est le premier échelon de certaines lignes
   (Pesthunter Badge → Ring → Artifact → Relic) et `Heirloom` l'échelon au-dessus de Relic,
   confirmé par la ligne Crux où *Crux Heirloom* (LEGENDARY) suit *Crux Relic* (EPIC).
2. **Motif « X of Y »**, où le suffixe est en tête de nom :
   *Talisman of Coins* → *Ring of Coins* → *Artifact of Coins* → *Relic of Coins*.
   Récupère les lignes Coins, Power, Century et Space, invisibles pour la règle 1.
3. **Table de correction manuelle** (`FAMILY_FIX`) pour les séries qu'aucune règle de nommage
   ne peut déduire : Master Skull, Personal Compactor/Deletor, Wedding Ring, Crux, Campfire et
   Soul Campfire Badges, Beastmaster Crest, colliers de dents de requin, **accessoires Chocolate**
   (Nibble Stick → Smooth Bar → Rich Chunk → Ganache Slab → Prestige Realm), Voter's Badge,
   Fish Bowl, Kuudra Core, et Piggy Bank — ce dernier se dégradant au lieu de s'améliorer,
   l'exemplaire intact est le meilleur état.

**Raretés manquantes** — 38 accessoires du catalogue Hypixel n'ont **aucun champ `tier`**
(les anciens talismans : Zombie, Speed, Scavenger, Master Skull I/II…). Une table
`TIER_FIX` les renseigne, avec repli sur `COMMON`.

### Hypothèses assumées

- **`SUPREME`** apparaît dans l'API sur un seul accessoire (Celestial Starstone) et n'est pas
  documenté par Hypixel. Il est compté **22 MP**, aligné sur `MYTHIC`.
- **La capacité du sac n'est pas exposée de façon fiable.** `bag_upgrades_purchased` vaut 99
  pour un sac contenant 113 accessoires, et la longueur de la liste NBT (271) est un tampon
  d'allocation. Plutôt que d'afficher un nombre de slots libres faux, l'app affiche le nombre
  d'accessoires portés et indique, pour une sélection donnée, combien d'achats occuperont un
  nouveau slot.
- **La MP calculée est la MP courante**, dérivée du contenu réel du sac. Elle peut différer de
  `highest_magical_power` renvoyé par Hypixel, qui est un **maximum historique**.

---

## Gestion des erreurs

Chaque cas produit un message clair et actionnable, jamais une page blanche :

- **Pseudo inexistant** — message distinguant explicitement pseudo Minecraft et pseudo Discord.
- **API du joueur désactivée** — le sac n'est pas lisible ; l'app donne la marche à suivre
  (Menu SkyBlock → Paramètres → API Settings → activer *Inventory*, puis rejoindre un lobby).
- **Service indisponible / réseau injoignable** — le service concerné est nommé.
- **Objet sans prix** — affiché « aucune vente » et exclu du classement par ratio, avec un
  compteur sous le tableau.
- **Navigateur sans `DecompressionStream`** — message explicite.
- Indicateur de chargement à chaque étape, barre de progression pour les prix, et
  **affichage progressif** : le tableau apparaît dès que le sac est décodé, les prix se
  remplissent au fur et à mesure.

---

## Déployer sur GitHub Pages

1. Créez un dépôt sur GitHub (par exemple `mp-optimizer`), public.

2. Poussez le contenu de ce dossier :

```bash
git remote add origin https://github.com/<votre-pseudo>/mp-optimizer.git
git branch -M main
git push -u origin main
```

3. Sur GitHub, ouvrez **Settings → Pages**.

4. Sous **Build and deployment**, choisissez :
   - **Source** : `Deploy from a branch`
   - **Branch** : `main`, dossier `/ (root)`
   - Cliquez **Save**.

5. Attendez une à deux minutes. Le site sera servi à l'adresse :

```
https://<votre-pseudo>.github.io/mp-optimizer/
```

Aucune étape de build, aucune GitHub Action, aucune variable d'environnement.
`index.html` se suffit à lui-même.

### Tester en local

Ouvrir `index.html` directement en `file://` fonctionne, mais un serveur local reproduit
fidèlement les conditions de GitHub Pages :

```bash
python -m http.server 8000
```

Puis rendez-vous sur `http://localhost:8000`.

---

## Limites connues

- L'app dépend de la disponibilité de **Coflnet**. C'est aujourd'hui le seul relais public
  du profil Hypixel sans clé ; s'il ferme comme SkyCrypt, il faudra une nouvelle source.
- Les prix de l'hôtel des ventes sont des **lowest BIN instantanés**. Un objet peut être
  vendu entre le chargement et l'achat.
- Les accessoires jamais mis en vente n'ont pas de prix et sont exclus du classement par ratio.
- Le profil actif est déterminé au mieux ; en cas d'échec l'app prend le premier profil et
  laisse le sélecteur disponible.

---

## Licence

MIT — voir [LICENSE](LICENSE).

Projet non officiel, sans affiliation avec Hypixel Inc. ou Mojang Studios.
