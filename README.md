# TeamSpace

Plateforme de travail multi-entreprises : projets, tâches, équipe et suivi d’activité.
Backend Spring Boot existant conservé et étendu ; interface React/TypeScript.

## Fonctionnalités implémentées
- Comptes, BCrypt, JWT d’accès de 10 minutes, refresh en cookie HttpOnly avec rotation et révocation.
- Protection CSRF des opérations de compte et CORS limité à une origine configurable.
- Récupération de mot de passe à usage unique, expiration à 30 minutes et révocation des sessions de renouvellement.
- Entreprises, memberships, rôles OWNER/ADMIN/MEMBER, protection du dernier OWNER actif.
- Projets, archivage, tâches, statuts, priorités, affectation, commentaires, recherche et pagination.
- Kanban avec persistance et maintien de l’état en cas d’échec ; alternative clavier via liste de statuts.
- Calendrier mensuel/hebdomadaire des échéances de tâches et projets.
- Invitations par lien privé et transport SMTP facultatif ; acceptation liée à l’email connecté.
- Notifications persistées d’affectation, changement de statut et commentaire ; journal d’activité.
- Statistiques calculées côté serveur pour l’entreprise active ; accès réservé aux administrateurs.
- Plans FREE/PRO/BUSINESS simulés avec limites côté serveur. Aucun paiement réel.
- Interface responsive, thèmes sombre/clair, validations, chargements et erreurs visibles.

## Démarrage Docker local isolé
Prérequis : Docker Desktop démarré (Linux containers).

```powershell
.\scripts\Initialize-Local.ps1
docker compose up --build
```

- Application : http://localhost:8088
- API et health : http://localhost:8081/actuator/health
- Emails de développement Mailpit : http://localhost:8025
- PostgreSQL Docker : localhost:55432, base `teamspace_isolated`.

Les secrets sont générés dans `.env`, ignoré par Git. Aucune donnée de démonstration n’est injectée automatiquement. Créez un compte et une entreprise depuis l’interface.
Cette procédure crée une base Docker distincte et ne touche pas à `teamspace_db`.
Si `.env` existe, le script refuse de l’écraser.
Arrêt sans effacer les données : `docker compose stop`. Ne pas utiliser `down -v` sur des données à conserver.
La configuration Compose est locale HTTP : ne pas l’exposer telle quelle sur Internet.

## Développement sans conteneuriser l’application
Java 17, Node.js 24, PostgreSQL 18, Docker Desktop pour les tests.
Configurer dans le processus backend :
- `DB_URL` : URL JDBC, par défaut `jdbc:postgresql://localhost:5432/teamspace_db`.
- `DB_USERNAME` : par défaut `postgres`.
- `DB_PASSWORD` : obligatoire ; ne pas remplacer `${DB_PASSWORD}` dans application.properties.
- `JWT_SECRET` : au moins 32 octets aléatoires encodés Base64.
- `COOKIE_SECURE=false` uniquement pour le développement local HTTP.
- `FRONTEND_ORIGIN=http://localhost:5173`.
- `MAIL_ENABLED=true`, `MAIL_HOST`, `MAIL_PORT` pour activer les emails ; sinon les invitations produisent un lien à partager et la récupération de compte indique l’absence de configuration.

Avant de démarrer contre une base existante, sauvegarder et inspecter son schéma. Les nouvelles tables sont créées par Flyway dans `teamspace`, sans baseline automatique de `public`, avec `ddl-auto=validate`. Ne pas activer `ddl-auto=create` ni `clean`.

```powershell
cd backend
.\mvnw.cmd spring-boot:run
# Deuxième terminal, depuis la racine
cd frontend
npm ci
npm run dev
```

Ouvrir http://localhost:5173. Vite transmet `/api` au backend sur 8080. `.env` de Compose n’est pas chargé automatiquement par Spring Boot ou PowerShell : exporter explicitement les variables dans le processus backend.

## Tests
```powershell
cd backend
.\mvnw.cmd test
cd ..\frontend
npm test
npm run build
```
Les tests backend lancent un PostgreSQL 18 jetable via Testcontainers et ne nécessitent pas le mot de passe de la base de développement. Docker et le téléchargement initial des images sont nécessaires. Un Docker indisponible provoque un échec visible, pas un succès artificiel.
Voir `docs/VALIDATION.md` pour les résultats effectivement exécutés et les limites de vérification.

## API
Les routes sont préfixées `/api/v1`. Les ressources d’entreprise utilisent `X-Tenant-Id` et le JWT dans `Authorization: Bearer ...`.
La valeur du header n’est jamais une autorisation : chaque service contrôle l’appartenance active et le rôle. Les Members ne voient que les projets auxquels ils sont autorisés.
- `/auth/csrf` : token CSRF à envoyer dans `X-XSRF-TOKEN` pour les opérations de compte.
- `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/me`.
- `/auth/forgot-password`, `/auth/reset-password`.
- `/tenants`, `/tenants/{id}/members`.
- `/projects`, `/projects/{id}/members`, `/tasks`, `/tasks/{id}/status`, `/tasks/{id}/comments`.
- `/invitations`, `/invitations/accept`, `/notifications`, `/activity`, `/analytics`, `/subscriptions`.

Activer `API_DOCS_ENABLED=true` pour accéder à `/swagger-ui.html` et `/v3/api-docs`. La documentation est désactivée par défaut. Les erreurs métier suivent ProblemDetail. Les pages retournent content, totalElements, totalPages, number et size (1–100).

## Architecture et sécurité
`backend/` : controllers → services transactionnels → repositories JPA, DTO explicites, contrôles tenant, migrations.
`frontend/` : pages, composants, layout, hooks TanStack Query, client Axios et session Zustand.
Les jetons d’accès restent en mémoire ; aucun JWT ou refresh token dans localStorage. Seuls le thème et l’identifiant d’entreprise sélectionné y sont conservés.
Le cookie refresh est HttpOnly, SameSite=Strict, Secure par défaut. En production, HTTPS et COOKIE_SECURE=true sont obligatoires.
Le serveur recontrôle les rôles sans se fier à un rôle stocké dans le JWT. Les FK composites empêchent les références de données entre entreprises.
Les plafonds de projets/membres et le dernier OWNER sont contrôlés dans des transactions sérialisées par verrou d’entreprise.
Voir `docs/DATA-MODEL.md` pour le diagramme et `docs/ROADMAP.md` pour le plan de réalisation.

## Déploiement et CI
Le pipeline GitHub Actions vérifie les tests PostgreSQL et le build frontend ; il n’a pas encore été exécuté sur GitHub.
Les Dockerfiles construisent le backend Java 17 et le frontend statique servi par Nginx.
Pour une mise en production : choisir un hébergement, configurer HTTPS, les secrets, une origine CORS exacte, un SMTP authentifié et des sauvegardes PostgreSQL. Ne pas exposer les ports PostgreSQL ou Mailpit. Activer Secure, désactiver la documentation si non souhaitée et conserver l’identité de la clé JWT entre redémarrages.
Aucune publication externe n’est effectuée automatiquement.

## Limites connues
- Abonnements simulés ; BUSINESS utilise actuellement 100 membres / 200 projets, sans éditeur de quotas.
- Pas de vérification d’email, WebSocket, pièces jointes ni téléversement d’avatar/logo.
- Notifications d’échéance vérifiées une fois par heure ; une notification par tâche et destinataire, sans répétition après report de date.
- Les sessions d’accès déjà émises restent valides jusqu’à expiration (10 minutes) après déconnexion/réinitialisation ; les refresh sont révoqués immédiatement.
- Limitation des connexions en mémoire : prévoir un stockage partagé derrière plusieurs instances.
- Envoi SMTP synchrone ; prévoir une outbox transactionnelle pour des garanties de livraison en production.
- Analytics : la courbe montre les tâches créées par jour et celles actuellement terminées parmi ces tâches, pas un historique journalier des événements de complétion.
- Chargement intégral paginé des tâches pour calendrier et Kanban : à remplacer par une fenêtre serveur pour les grands volumes.
- Suppression d’un projet avec tâches refusée ; l’archivage est proposé.
- Les paramètres affichent le profil en lecture seule.

## Présentation CV / portfolio
Développement de TeamSpace, plateforme collaborative full stack multi-entreprises avec Java 17/Spring Boot et React/TypeScript : authentification JWT avec rotation des sessions, autorisations par rôle, gestion de projets et tâches, Kanban persistant, calendrier et statistiques PostgreSQL. Mise en place de migrations Flyway, tests d’intégration avec Testcontainers et configuration Docker/CI.


## Captures réelles de la version locale

Les données présentées appartiennent à un environnement de démonstration isolé.

![Dashboard réel](docs/screenshots/teamspace-dashboard.jpg)
![Kanban après rechargement](docs/screenshots/teamspace-kanban.jpg)
![Accueil mobile](docs/screenshots/teamspace-mobile.jpg)

Les fonctionnalités et limites vérifiées sont détaillées dans `docs/VALIDATION.md`.

## Refonte Organic Futuristic 3D
Accueil avec bureaux circulaires WebGL interactifs, arbre central et sélection des entreprises autorisées ; nouvelle identité crème/pêche/bleu sur l’authentification et l’application. Adaptation mobile, pause, mouvement réduit et illustration de repli sans WebGL.

![Accueil 3D](docs/screenshots/teamspace-organic-3d.jpg)
![Dashboard](docs/screenshots/teamspace-organic-dashboard.jpg)

Périmètre, organisation, validations et limites : [bilan de la refonte](docs/3D-REDESIGN.md).

## Identité actuelle — Organic Wave
La refonte Organic Wave remplace l’accueil architectural précédent : dashboard HTML/CSS en perspective, vagues SVG, orange et bleu nuit, sidebar sombre et adaptation bureau/tablette/mobile. Les fonctions métier restent inchangées.

Détails : [bilan Organic Wave](docs/ORGANIC-WAVE.md). Les documents de la refonte 3D précédente décrivent une version historique.

![Accueil Organic Wave](docs/screenshots/teamspace-wave-accueil.jpg)
#   - P l a t e f o r m e - S a a S - d e - g e s t i o n - c o l l a b o r a t i v e - T e a m S p a c e -  
 