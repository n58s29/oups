# Le Grand Départ — v3

Death pool jusqu'à ~100 joueurs. Comptes utilisateurs, phase de choix chronométrée,
exclusivité au premier arrivé, vérification quotidienne des décès sur Wikidata.
Un seul fichier : `index.html`. Aucune dépendance à installer, aucun build.

---

## Ce qui a changé par rapport à Oups v2

| Oups v2 | Le Grand Départ v3 |
|---|---|
| Enchères scellées, budget 500 M, tours | Supprimé. Chacun réserve **20 têtes max**, premier arrivé premier servi |
| 15 têtes max | 20 têtes max |
| Pas de compte : prénom + code | **Compte email + mot de passe** (Supabase Auth), pseudo, multi-parties |
| Un document JSON par partie, verrou optimiste | **Schéma normalisé** (`gd_games`, `gd_participants`, `gd_picks`) : zéro contention à 100 joueurs |
| Mises cachées par le client (triche possible en console) | L'exclusivité rend tout public par construction ; l'écriture est verrouillée par **RLS** |
| « Tout le monde est prêt » → révélation | **Phase de choix à durée limitée** réglée par l'admin, compte à rebours, prolongeable |
| Partie fermée après le lancement | **On monte à bord à tout moment** : un décès antérieur à ta réservation ne rapporte rien |
| Adjudication par « le premier client qui s'en aperçoit » | Plus d'adjudication du tout. Le seul arbitre des courses au clic : `UNIQUE(game_id, qid)` dans Postgres |
| Vérif décès toutes les 4 min par chaque client | **Une fois par jour** (garde 6 h en base), barème calculé côté serveur |
| Mode simu + bots | Supprimé (conçu pour les enchères ; l'app se teste à deux navigateurs) |

Le barème est inchangé : page Wikipédia obligatoire, 1 point à 90 ans et plus,
sinon 90 − l'âge.

---

## Mise en route (10 minutes)

### 1. Supabase

1. Réutilise le projet d'Oups v2 (les tables `gd_*` cohabitent avec l'ancienne
   table `games`) ou crée un projet neuf — dans ce cas choisis une **région EU**
   (tu vas stocker des emails de collègues).
2. *SQL Editor* → colle `supabase.sql` → **Run**. La dernière requête doit
   renvoyer `4 | 12 | 3 | t`.
3. **Indispensable** : *Authentication → Sign In / Providers → Email* →
   décoche **« Confirm email »**. Sans ça, chaque inscription attend un mail de
   confirmation, et le mailer intégré de Supabase est limité à 2-4 mails/heure :
   vos 100 inscriptions du premier soir resteraient bloquées. Sans confirmation,
   zéro mail envoyé, inscriptions instantanées.
4. Si projet neuf : reporte l'URL et la *publishable key* dans le `CFG` en haut
   du script d'`index.html`.

### 2. Héberger

GitHub Pages, comme la v2 :

```bash
git add le-grand-depart/index.html
git commit -m "Le Grand Départ v3"
git push
```

HTTPS obligatoire (Auth + `navigator.share`). Wikipédia/Wikidata autorisent le
CORS depuis n'importe quelle origine.

### 3. Lancer un tournoi

1. Chacun crée son compte (email + mot de passe + pseudo). Une seule fois :
   la session persiste des mois sur le téléphone.
2. L'organisateur crée la partie → code à 4 lettres → le partage.
3. Les joueurs montent à bord. L'organisateur choisit la durée de la course
   (15 min à 24 h) et **donne le départ**.
4. Pendant la course : chacun réserve jusqu'à 20 têtes, peut en rendre, peut
   se faire griller. Tout est public, ça va vite, c'est le but.
5. Fin du compte à rebours (ou clôture manuelle par l'admin) → la saison
   commence. Les sélections sont figées, sauf places libres.
6. Chaque matin, le premier joueur qui ouvre l'app déclenche la vérification
   Wikidata pour toute la partie (au plus une fois toutes les 6 h). Les autres
   reçoivent le récap « Pendant ton absence » à l'ouverture.

---

## Comment ça marche sous le capot

**Plus de document JSON, des lignes.** Chaque réservation est une ligne de
`gd_picks`. La règle centrale du jeu — l'exclusivité — est une contrainte
`UNIQUE (game_id, qid)` : deux joueurs qui valident la même tête à la même
milliseconde, Postgres arbitre, un seul passe, l'autre reçoit « déjà pris ».
Aucun code de résolution de conflit, aucune contention à 100 joueurs.

**Les phases sont dérivées de l'horloge.** `choice_ends_at` vide = lobby ;
futur = course ; passé = saison. Aucune transition serveur, aucun cron pour
fermer la phase : chaque client regarde sa montre.

**RLS fait la police.** On ne réserve que pour soi, dans une partie où l'on est
inscrit, une fois la partie lancée. On ne rend que ses propres têtes vivantes,
et seulement pendant la course. Personne ne modifie un pick directement : les
décès passent par `gd_record_deaths()` (security definer), qui calcule le
barème côté serveur et applique la règle du retardataire
(`décès antérieur au pick = 0 point`).

**Décès sans serveur applicatif.** Le premier client du jour interroge Wikidata
(propriété P570, par lots de 40 QIDs) et transmet les décès trouvés à la
fonction Postgres. La garde de 6 h en base évite les rafales ; l'admin peut
forcer une vérification. La notification est in-app : toast en direct si tu as
l'app ouverte, récap à l'ouverture sinon.

**Realtime + filet.** Abonnement aux changements de `gd_picks` /
`gd_participants` / `gd_games`, plus une resynchronisation complète toutes les
30 s quand l'onglet est visible. Réseau capricieux = latence, jamais de blocage.

---

## Limites connues

- **La date de décès vient du client.** Le premier client qui détecte un décès
  transmet la date Wikidata à la fonction serveur. Un tricheur motivé pourrait
  forger un faux décès — mais il serait visible de tous, contredit par Wikidata
  au premier coup d'œil, et le chef de gare peut le sortir. Pour du vrai
  zéro-confiance, la suite logique est une Edge Function planifiée (pg_cron)
  qui interroge Wikidata elle-même ; l'architecture actuelle rend ce changement
  local (seule `gd_record_deaths` et son appelant bougent).
- **Pas de « mot de passe oublié »** : la confirmation email étant désactivée,
  le reset par mail est limité par le mailer Supabase (2-4/heure). Ça passe
  pour un oubli isolé, pas pour une vague. En pratique : un mot de passe simple
  et une session qui ne expire pas.
- **La notification est in-app uniquement.** Pas de push : sur iPhone le Web
  Push exige une PWA installée sur l'écran d'accueil. C'est le candidat
  naturel de la v4 si le jeu prend.
- Les parties sont visibles (lecture seule) de tout compte connecté qui en
  connaît l'existence ; le code à 4 lettres protège l'entrée, pas la lecture.
  Acceptable entre collègues, à durcir si besoin (policy sur l'appartenance).
- Une saison = une partie. Le classement au 31 décembre fait foi, à l'ancienne.

## Suites possibles

- Web Push pour les PWA installées (table `push_subscriptions` + envoi depuis
  une Edge Function).
- Edge Function + pg_cron pour un check décès 100 % serveur à 8 h.
- Export du classement en image pour le groupe WhatsApp.
- Badge « grillé ! » : compter combien de fois chaque joueur s'est fait souffler
  une tête pendant la course.
