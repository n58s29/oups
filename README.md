# Oups. — v2

Death pool entre potes. Chacun sur son téléphone, mises secrètes, synchro temps réel.
Un seul fichier : `index.html`. Aucune dépendance à installer, aucun build.

---

## Ce qui a changé par rapport à la v1

| v1 | v2 |
|---|---|
| Un seul appareil passé de main en main, avec un « sas de confidentialité » | Un téléphone par joueur, code de partie à 4 lettres |
| 5 onglets (Enchères / Écuries / Classement / Joueurs / Règles) | 3 onglets (Enchères / Écurie / Classement) + règles en feuille |
| Sélection du pupitre, bris de sceau, fermeture du carnet | Supprimé : ton téléphone, c'est ton carnet |
| Bouton « ✝ Vérifier » sur chaque tête | Vérification Wikidata automatique, en une requête pour toute la partie |
| Mise saisie à l'aveugle dans un champ nombre | Valeur affichée + pastilles de mise (valeur, +25 %, +60 %, ×2) + champ libre |
| Quota d'enchères = « autant que d'enchères perdues » (difficile à expliquer) | Quota = **places libres dans ton écurie**. Même dynamique, une phrase de moins |
| Clôture du tour par un bouton que n'importe qui pouvait presser | Chacun appuie sur « Je suis prêt ». Tout le monde prêt → révélation automatique |
| Cibles invendues qui traînent dans l'état | Nettoyées à chaque clôture |
| Écran dense pensé desktop | Mobile-first : cibles tactiles 48 px, barre d'action collante, safe-area iPhone |

Le principe et le barème sont inchangés : 500 M de budget, 15 têtes max, page Wikipédia
obligatoire, exclusivité, 1 point au-delà de 90 ans sinon 90 − l'âge.

---

## Tester tout de suite sur ton téléphone (mode simu)

Tel quel, sans clé Supabase, l'app tourne **100 % en local** dans le navigateur :

1. Ouvre `index.html`.
2. Prénom → **Créer une partie**.
3. **＋ Inviter 4 potes fictifs** — ils misent tout seuls, contrent parfois tes cibles, et se déclarent prêts.
4. **Lancer le tour 1**, ajoute 1 ou 2 cibles réelles via la recherche Wikipédia, puis **Je suis prêt**.
5. La révélation tombe dès que les bots sont prêts (quelques secondes).
6. Onglet Écurie → **☠ Simuler un décès** pour voir les points tomber et le classement bouger.

Un second joueur humain sur le même téléphone : ouvre un autre onglet sur la même URL,
mets un autre prénom, saisis le code de la partie. Les deux onglets se synchronisent.

Le badge `simu` / `live` en haut à droite indique toujours dans quel mode tu es.

---

## Héberger sur GitHub Pages

```bash
git init
git add index.html
git commit -m "Oups. v2"
git branch -M main
git remote add origin git@github.com:<toi>/oups.git
git push -u origin main
```

Puis *Settings → Pages → Source : Deploy from a branch → main / (root)*.
L'URL sera `https://<toi>.github.io/oups/`.

Deux points à savoir :

- Wikipédia et Wikidata autorisent le CORS (`origin=*`), donc la recherche marche depuis
  GitHub Pages sans proxy.
- GitHub Pages sert en HTTPS, indispensable pour `navigator.share` et pour l'API Supabase.

---

## Passer en vrai multi-joueurs (Supabase)

1. Crée un projet sur [supabase.com](https://supabase.com) (free tier, largement suffisant).
2. *SQL Editor* → colle `supabase.sql` → **Run**.
3. *Project Settings → API* → récupère **Project URL** et la clé **anon public**.
4. Dans `index.html`, en haut du script :

```js
const CFG = {
  url : 'https://xxxxxxxxxxxx.supabase.co',
  key : 'eyJhbGciOi...'
};
```

5. Commit, push. Le badge passe de `simu` à `live`, les potes fictifs disparaissent,
   et chacun rejoint depuis son propre téléphone avec le code à 4 lettres.

La clé `anon public` est faite pour être publiée dans du code client — c'est son rôle.
La sécurité repose sur les policies RLS du fichier SQL, pas sur le secret de la clé.

---

## Comment ça marche sous le capot

**Un document, une version.** Toute la partie tient dans un seul JSON
(`joueurs`, `celebs`, `bids`, `log`). Chaque écriture est une transaction optimiste :
on lit `version`, on modifie, on écrit avec `where version = <lu>`. Si un autre
téléphone est passé avant, on relit et on rejoue — jusqu'à 7 tentatives.
Résultat : pas de serveur applicatif, pas de fonction edge, pas de race condition
sur les mises simultanées.

**Deux adaptateurs, une interface.** `LocalNet` (localStorage + BroadcastChannel)
et `SupaNet` (Postgres + Realtime) exposent exactement `create / load / commit / subscribe / close`.
Le reste du code ne sait pas lequel tourne. C'est ce qui permet au mode simu d'être
un vrai test du jeu et pas une maquette parallèle.

**Realtime + filet.** On s'abonne aux `postgres_changes` sur la ligne de la partie,
et on interroge quand même la table toutes les 5 secondes. Si le Realtime est mal
configuré ou coupé par un réseau capricieux, la partie continue avec un peu de latence
au lieu de se figer.

**Adjudication.** Dès que le dernier joueur se déclare prêt, le premier client qui
s'en aperçoit exécute la résolution dans un `commit`. Les deux gardes (`phase === 'bids'`
et « tout le monde est prêt ») font que si trois téléphones tentent en même temps,
un seul passe et les autres constatent que c'est déjà fait.

**Décès.** Une seule requête `wbgetentities` couvre jusqu'à 40 QIDs. On lit la propriété
`P570` (date de mort). Au chargement, puis toutes les 4 minutes, plus un bouton manuel.

---

## Limites connues

- **Les mises « scellées » sont cachées par le client, pas par le serveur.** Un joueur
  motivé qui ouvre la console peut lire le JSON de la partie avant la révélation.
  Pour une partie entre potes c'est acceptable ; si tu veux du vrai scellé, la suite
  logique est de déplacer `bids` dans une table séparée avec une policy RLS
  `player_id = auth.uid()` et l'adjudication dans une fonction Postgres (`security definer`).
  L'architecture actuelle rend ce changement local : il ne touche que `SupaNet`.
- Pas d'authentification : celui qui a le code entre. Le code fait 4 lettres sur 24
  (331 776 combinaisons), suffisant contre le hasard, pas contre l'acharnement.
- Un joueur qui vide le stockage de son navigateur perd son identité et devra rejoindre
  comme nouveau joueur (l'ancien restera dans la partie avec son écurie).
- Une saison = une partie. Pas de clôture automatique au 31 décembre, c'est le classement
  au 31 décembre qui fait foi, à l'ancienne.

## Suites possibles

- Notification quand une tête de ton écurie décède (Web Push, ou simple toast au retour sur l'app).
- Export du classement en image pour le groupe WhatsApp.
- Enchères à budget engagé plutôt que dépensé (les perdants immobilisent leur mise un tour).
- Mode « draft » alternatif : chacun choisit à son tour, sans enchère, pour les parties rapides.
