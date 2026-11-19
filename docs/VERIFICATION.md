# Vérifications et captures

Vérifications effectuées le 19 novembre 2026, sur l’application du dépôt et des données locales de démonstration. Les résultats ci-dessous distinguent le test Spring existant des appels manuels de l’API.

## Environnement et construction

| Vérification | Résultat |
| --- | --- |
| Java | OpenJDK 21.0.13.0.3 |
| Maven | 3.8.7 |
| `mvn -o -q -DskipTests package` | Réussi avec les dépendances présentes dans le cache Maven |
| `mvn -o test`, avant démarrage de PostgreSQL | Échec du chargement de contexte, connexion PostgreSQL indisponible |
| `docker compose up -d postgres axonserver` | Réussi après une nouvelle tentative de téléchargement des images |
| `mvn -o -q spring-boot:run` | Application démarrée sur `http://localhost:8066` |
| `mvn -o test`, infrastructure disponible | 1 test, 0 échec, 0 erreur |
| `GET /v3/api-docs` | HTTP 200, les huit chemins correspondent aux contrôleurs et à la référence API |
| `GET /` de l’application | HTTP 404 ; aucune page d’accueil bancaire n’est fournie |
| Console `http://localhost:8024/` | HTTP 200 |

La console de l’image Axon Server téléchargée affiche `2026.1.8`, avec un bandeau de mode non enregistré et une limite d’accès de 12 heures. Aucune inscription ni modification de licence n’a été effectuée.

L’option `-o` utilise uniquement le cache Maven local. Le test `contextLoads` ne constitue pas une couverture des règles métier.

## Parcours API observé

Un compte de démonstration a été créé avec `1000 MAD`, crédité de `250`, puis débité de `100`.

| Action | Résultat observé |
| --- | --- |
| Création | HTTP 200, identifiant UUID renvoyé ; projection activée |
| Crédit et débit | HTTP 200, corps vide |
| Liste des comptes après projection | HTTP 200 ; solde `1150.0`, statut `ACTIVATED`, devise du compte `null` |
| Événements après ces opérations | HTTP 200 ; création, activation, crédit et débit présents |
| Suivi SSE ouvert avant un crédit | Opération `CREDIT` de `250.0` reçue pour le compte suivi |
| Débit supérieur au solde | HTTP 200 avec le texte `Balance non suffisant` |
| Passage à `SUSPENDED`, puis `ACTIVATED` | HTTP 200 ; retour au statut activé confirmé dans la projection |
| Relevé `accountsStatement` | HTTP 500, incompatibilité XStream / record Java |

Le relevé échoue sur la désérialisation de `AccountStatementReponseDTO` avec XStream 1.4.20 : `ConversionException`, causée par `UnsupportedOperationException: can't get field offset on a record class`. Le solde correctement projeté ne résout pas cette erreur de transport de la réponse. La documentation ne prétend pas que ce parcours est entièrement réussi.

Les données de démonstration restent dans l’instance locale. Aucun test d’authentification, de charge, de reprise après panne ou de replay des projections n’a été effectué.

## Captures de l’application

La [galerie des captures](SCREENSHOTS.md) regroupe les douze images dans une page dédiée. Les anciens fichiers `img*.png` supprimés par l’utilisateur ne sont pas réutilisés.

Les douze captures ont été obtenues, décodées et ouvertes pour inspection visuelle. Les dix captures web mesurent **1920 × 1040 pixels**, en mode natif avec la vraie barre d’adresse Chrome et sans bannière d’automatisation. Les trois premières montrent Swagger UI, la liste JSON des comptes et la console Axon Server ; les sept suivantes documentent les tests API. Les deux nouvelles captures de GNOME Terminal mesurent **1962 × 465** et **1962 × 621 pixels**.

Les captures du terminal montrent l’exécution réelle de `docker compose up -d --pull never postgres axonserver`, l’état `running` des deux services, leurs journaux de démarrage et les contrôles `pg_isready` / HTTP 200. Les services étaient déjà démarrés ; les commandes ont été exécutées sur les conteneurs existants. Les fenêtres dédiées ont été capturées avec `gnome-screenshot --window` puis fermées. Ces deux images sont des captures de terminal natif, sans barre Chrome.

Le [parcours des tests en images](SCREENSHOTS.md) présente la création, le crédit, le débit, le rejet pour solde insuffisant, la suspension, la lecture des événements et l’erreur du relevé. La création illustrée est indépendante ; les six autres tests partagent un compte local dédié initialisé à `1000 MAD`. Après crédit de `250`, débit de `100` et tentative de débit excessif rejetée, la projection a confirmé un solde de `1150` et le statut `SUSPENDED`. L’Event Store contient les cinq événements attendus, dont le changement de statut.

Pour reproduire les vues navigables avec une session Chrome :

```bash
capture-chrome http://localhost:8066/swagger-ui/index.html docs/screenshots/01-swagger.png
capture-chrome http://localhost:8024/ docs/screenshots/03-axonserver.png
capture-chrome http://localhost:8066/query/accounts/all docs/screenshots/02-comptes.png
```

## Historique Git

Les dates auteur et committer des 14 commits ont été redistribuées à la demande de l’utilisateur, du 10 novembre 2026 à 09:30 au 12 novembre 2026 à 17:30, dans le fuseau `Africa/Casablanca`. Ce calendrier réorganisé ne constitue pas une preuve de travail effectué à ces dates.

- Ancienne tête : `413dac4ed828d6ba6ba675a57edd0735e229ec33`.
- Nouvelle tête après redistribution : `687a42409a8990fe0f74ee09b85be183001063d3`.
- Sauvegarde locale : `backup-redistribute-7c797e896de9`.

La comparaison des arbres, auteurs et messages des 14 commits est identique. `git diff 413dac4ed828d6ba6ba675a57edd0735e229ec33 687a42409a8990fe0f74ee09b85be183001063d3` ne produit aucune différence. Les suppressions de l’ancien README et des anciennes images ont ensuite été réappliquées à la copie de travail. Aucune publication ni réécriture du dépôt distant n’a été effectuée.

## Contrôle documentaire

Les routes ont été comparées au code et à l’OpenAPI servi par l’application. Les liens relatifs ont été contrôlés et pointent vers des fichiers existants. La configuration Playwright est restée identique à son empreinte de référence. Les diagrammes Mermaid ont été inspectés au niveau de leur syntaxe ; aucun moteur Mermaid de rendu n’est disponible dans l’environnement utilisé.

[Retour au README](../README.md) · [API](API.md) · [Architecture](ARCHITECTURE.md)
