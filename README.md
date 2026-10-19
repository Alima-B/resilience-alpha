# PROJECT RESILIENCE-ALPHA — backend (plateforme de paiement fictive)

Backend Python/FastAPI de la plateforme fictive. Aucune donnée réelle : tout est simulé.

## Arborescence

```
resilience-alpha/
├── docker-compose.yml          # orchestration (réseaux edge / backend / remediation / obs)
├── .env.example                # modèle ; `python scripts/gen_env.py` génère .env (secrets aléatoires)
├── gateway/nginx.conf          # seul point d'entrée exposé (à placer derrière le WAF)
├── db/init/                    # 01-roles.sh (rôle PostgreSQL par service), 02-schema.sql
├── libs/common/resilience_common/
│   ├── app.py                  # fabrique FastAPI : /health, /ready, /metrics, middleware logs
│   ├── security.py             # JWT, rôles, jeton de service interne
│   ├── logs.py                 # logs JSON sur stdout (collectés par Promtail → Loki)
│   ├── metrics.py  db.py  events.py  config.py
├── services/
│   ├── Dockerfile              # image unique, paramétrée par SERVICE
│   ├── auth/                   # inscription, login (JWT), rôles
│   ├── accounts/               # comptes + ledger (écritures), contrôle d'intégrité
│   ├── payments/               # paiements, idempotence, statuts
│   ├── notifications/          # notifications asynchrones simulées
│   ├── audit/                  # piste d'audit en ajout seul, vérifiable (chaîne de hachage)
│   ├── remediation/            # alertes → actions en liste blanche → vérification
│   └── canary/                 # transaction synthétique de bout en bout (preuve de reprise)
├── infra/                      # prometheus, promtail, grafana (datasources)
├── scripts/                    # gen_env.py, smoke.py, export_openapi.py
├── tests/                      # tests de la bibliothèque commune
└── docs/api/*.openapi.json     # documentation d'API générée
```

## Démarrage

```bash
python scripts/gen_env.py        # crée .env (ne jamais le committer)
docker compose up -d --build
python scripts/smoke.py          # parcours complet : inscription, compte, paiement, vérifications
```

Tests unitaires : `PYTHONPATH=libs/common pytest tests`
Documentation d'API : `python scripts/export_openapi.py`

## Choix de sécurité

- Seule la passerelle publie un port ; les réseaux `backend`, `remediation` et `obs` sont `internal`.
- Aucun service ne monte le socket Docker : le moteur de remédiation passe par un proxy
  (`docker-proxy-remediation`) limité à l'inspection et au redémarrage/arrêt de conteneurs.
- Conteneurs en lecture seule, `cap_drop: ALL`, `no-new-privileges`.
- Un rôle PostgreSQL distinct par service (`db/init/01-roles.sh`).
- À faire avant rendu : épingler l'image `tecnativa/docker-socket-proxy`, brancher WAF/SIEM/Vault.

## Logs : faut-il MongoDB ?

Mon avis : non, pas pour les logs techniques.

- **Le sujet impose Loki** pour la centralisation des journaux, avec Grafana pour la visualisation.
  Un MongoDB en plus ferait doublon et il faudrait justifier le second stockage devant le jury.
- **Coût mémoire** : le sujet insiste sur la charge du PC Hôte (16 Go). MongoDB consomme du cache
  et ajoute un conteneur ; Loki indexe seulement les labels et reste léger.
- **Intégration** : Promtail → Loki → Grafana est prêt et alimente le SIEM/détection ML via LogQL.
  Avec MongoDB, il faudrait écrire le collecteur, les requêtes et le datasource.
- **Ce qui est déjà couvert** : la piste d'audit métier (connexions, paiements, actions de
  remédiation) est dans PostgreSQL, en ajout seul et chaînée par hachage. Elle doit rester
  transactionnelle et vérifiable, ce que MongoDB n'apporte pas davantage.

MongoDB n'aurait de sens que pour stocker des événements de sécurité semi-structurés et très
variables (par exemple des alertes WAF brutes) si votre équipe SIEM le demande. Dans ce cas,
c'est une décision d'architecture à documenter dans le dossier, pas un prérequis du backend.
