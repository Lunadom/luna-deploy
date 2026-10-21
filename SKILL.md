---
name: luna-deploy
description: Déployer ou modifier le serveur Flask de Maîtresse Luna sur PythonAnywhere (compte Furiosha) sans casser ce qui tourne. À utiliser dès qu'il faut toucher à server.py, à une page statique, ou ajouter une route.
---

# Déploiement Luna — procédure verrouillée

## Règle d'or
Le site Furiosha n'héberge pas qu'une boutique : il héberge l'espace de David.
Une erreur de déploiement ne casse pas un site vitrine, elle le coupe de son journal.
Donc : **sauvegarde → syntaxe → envoi → reload → batterie de tests → rollback si un test échoue**.
Jamais d'improvisation, jamais de test partiel.

## 1. Toujours localiser les fichiers AVANT
- Webapp : `POST /api/v0/user/Furiosha/webapps/Furiosha.pythonanywhere.com/reload/` — **le slash final est obligatoire**, sans lui : 301 et rien ne recharge.
- `source_directory` = `/home/Furiosha/portecles`
- `working_directory` = `/home/Furiosha` ← **piège majeur** : tout chemin relatif écrit dans `/home/Furiosha`, pas dans `portecles`. Un fichier créé en relatif sera introuvable là où on le cherche.
→ Dans server.py, tout chemin de fichier doit être **absolu**.

## 2. Sauvegarde horodatée (non négociable)
```
curl -s -H "Authorization: Token $T" \
  -F "content=@/tmp/server_nouveau.py" \
  "https://www.pythonanywhere.com/api/v0/user/Furiosha/files/path/home/Furiosha/portecles/server_backup_AAAAMMJJ_avant_MOTIF.py"
```
Puis récupérer l'actuel en local avant modification :
```
curl -s -H "Authorization: Token $T" \
  "https://www.pythonanywhere.com/api/v0/user/Furiosha/files/path/home/Furiosha/portecles/server.py" -o /tmp/server_actuel.py
```

## 3. Contrôle de syntaxe OBLIGATOIRE avant envoi
```
python3 -c "import ast;ast.parse(open('/tmp/server_nouveau.py',encoding='utf-8').read());print('syntaxe OK')"
```
Pour une page HTML avec JS : extraire chaque bloc `<script>` et le passer à `node --check`.

## 4. Patch par remplacement ancré, jamais par regex floue
Écrire le patch en Python avec `s.replace(ancien, nouveau, 1)` **et** un `assert ancien in s` avant.
Si l'ancre est absente, on s'arrête : c'est que le code a changé, il faut relire.

## 5. Envoi + reload
```
curl -s -H "Authorization: Token $T" -F "content=@/tmp/server_nouveau.py" \
  "https://www.pythonanywhere.com/api/v0/user/Furiosha/files/path/home/Furiosha/portecles/server.py" \
  -o /dev/null -w "upload:%{http_code}\n"
sleep 1
curl -s -X POST -H "Authorization: Token $T" \
  "https://www.pythonanywhere.com/api/v0/user/Furiosha/webapps/Furiosha.pythonanywhere.com/reload/" \
  -o /dev/null -w "reload:%{http_code}\n"
sleep 6-7   # le worker met quelques secondes à redémarrer
```
Attendre **6 à 7 secondes** avant de tester, sinon on teste l'ancienne version et on croit à un échec.

## 6. Batterie de tests — à passer EN ENTIER, à chaque fois
Ne jamais tester « juste la route qu'on vient d'ajouter ». Tester le parcours complet :

```
B=https://furiosha.pythonanywhere.com
# espace protégé : lien personnel puis cookie
curl -s -c /tmp/c.txt -o /dev/null -w "lien:%{http_code}\n" "$B/e7k2m9"        # 302
curl -s -b /tmp/c.txt -o /dev/null -w "journal:%{http_code}\n" "$B/luna/intime/journal"   # 200
curl -s -b /tmp/c.txt -o /dev/null -w "stats:%{http_code}\n"   "$B/luna/intime/stats"     # 200
curl -s -b /tmp/c.txt -o /dev/null -w "dette:%{http_code}\n"   "$B/luna/intime/dette"     # 200
curl -s -b /tmp/c.txt -o /dev/null -w "humeur:%{http_code}\n"  "$B/luna/intime/bilan"     # 200
curl -s -o /dev/null -w "sans rien:%{http_code}\n" "$B/luna/intime/journal"               # 401
# verrouillage
curl -s -o /dev/null -w "faux:%{http_code}\n" "$B/luna/intime/journal?k=zzz"              # 401
# pages publiques et boutique
for u in / /produit /vente /static/tout-8m3q.html /static/envoi.html /espace; do
  curl -s -o /dev/null -w "$u:%{http_code} " "$B$u"; done; echo
```
Puis **remettre les compteurs à zéro** si un test a incrémenté un verrou :
`curl "$B/debloquer?k=<LUNA_KEY>"`

Critère : un seul 500, ou un 401 là où un 200 est attendu, ou une page vide → **rollback**.

## 7. Rollback
Ré-uploader la sauvegarde, reload, re-tester. Trente secondes. Ne jamais « réparer sur le tas » en production sans être revenu à l'état sain.

## 8. Pièges connus, tous vérifiés
- **`/root/inspect.py` masque la stdlib** : tout script Python doit tourner depuis `/tmp`, jamais depuis `/root`.
- **Shell persistant cassé** (erreur de snapshot) → relancer en `fresh=true`.
- **API fichiers = 429 vite** : espacer les appels de 1,2 s minimum ; au-delà de ~60 opérations, lancer en arrière-plan avec log.
- **API fichiers = cache** : un fichier supprimé ou modifié peut encore apparaître plusieurs minutes. Vérifier par GET/HEAD sur l'URL publique, pas par `files/tree`.
- **Fichiers cachés** : `.lock_attempts.json` et similaires sont invisibles dans `files/tree` et parfois impossibles à supprimer. Ne pas bâtir une sécurité critique sur un fichier caché qu'on ne peut pas lever.
- **Heredoc + guillemets** : `P='"motdepasse"'` garde les guillemets littéraux → 401 trompeur. Toujours `printf '%s'` ou variable sans guillemets imbriqués.
- **TZ Paris non appliqué** dans le sandbox (`TZ=Europe/Paris date` renvoie UTC). Convertir à la main (+2 été / +1 hiver).
- **Secrets en dur dans les pages statiques** : après tout changement de jeton, `grep` le jeton (et sa version base64) dans tous les `.html` de `/home/Furiosha/portecles` **et** `/static`. Leçon du 07/10 : 5 pages gardaient un jeton mort en base64 et envoyaient « Envoyé » alors que rien ne partait.
- **Ordre du middleware** : un contrôle d'accès doit tester « déjà autorisé » AVANT « trop d'essais ». Sinon on s'enferme soi-même dehors (leçon du 21/10, vécue).

## 9. Après coup
- Purple : noter dans la mémoire ce qui a changé, l'URL, l'ID de la route, le backup.
- Si une route protégée a été ajoutée : vérifier qu'elle renvoie 401 **sans** clé, et 200 **avec**.
