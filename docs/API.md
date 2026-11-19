# Référence API — AxonBank

> Ce document fournit la spécification exhaustive des endpoints REST et du flux réactif SSE exposés par le backend **AxonBank**.

- **URL de base (Développement) :** `http://localhost:8066`
- **Formats d'échange :** `application/json` (REST) et `text/event-stream` (SSE)
- **Documentation interactive (Swagger UI) :** `http://localhost:8066/swagger-ui/index.html`
- **Spécification OpenAPI v3 :** `http://localhost:8066/v3/api-docs`

---

## 1. Endpoints de Commande (Command Side)

Les commandes modifient l'état du domaine. Elles sont validées par l'agrégat et enregistrées dans l'Event Store d'Axon Server.

### 1.1. Créer un compte bancaire

Initialise un nouveau compte avec un solde de départ et l'active immédiatement.

- **Méthode :** `POST`
- **URL :** `/commands/accounts/add`
- **En-tête :** `Content-Type: application/json`

#### Corps de la requête (Request Body)

```json
{
  "initialBalance": 1000.0,
  "currency": "MAD"
}
```

#### Réponse de succès (`200 OK`)

Retourne l'identifiant UUID unique généré pour l'agrégat :

```text
"e756cabc-99b5-439f-a1a7-2cf5ed99e263"
```

---

### 1.2. Créditer un compte

Ajoute un montant spécifié au solde d'un compte actif.

- **Méthode :** `POST`
- **URL :** `/commands/accounts/credit`
- **En-tête :** `Content-Type: application/json`

#### Corps de la requête

```json
{
  "accountId": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
  "amount": 250.0,
  "currency": "MAD"
}
```

#### Réponse de succès (`200 OK`)

Corps de réponse vide (exécution asynchrone confirmée par le bus de commande).

---

### 1.3. Débiter un compte

Retire un montant du solde d'un compte, sous réserve de provision suffisante.

- **Méthode :** `POST`
- **URL :** `/commands/accounts/debit`
- **En-tête :** `Content-Type: application/json`

#### Corps de la requête

```json
{
  "accountId": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
  "amount": 100.0,
  "currency": "MAD"
}
```

#### Réponse de succès (`200 OK`)

Corps de réponse vide.

---

### 1.4. Mettre à jour le statut d'un compte

Modifie le cycle de vie du compte bancaire (`ACTIVATED`, `SUSPENDED`, `BLOCKED`).

- **Méthode :** `PUT`
- **URL :** `/commands/accounts/updateStatus`
- **En-tête :** `Content-Type: application/json`

#### Corps de la requête

```json
{
  "accountId": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
  "status": "SUSPENDED"
}
```

#### Réponse de succès (`200 OK`)

Corps de réponse vide.

---

### 1.5. Consulter l'historique des événements de l'Event Store

Retourne la totalité des messages d'événements enregistrés pour un agrégat donné.

- **Méthode :** `GET`
- **URL :** `/commands/accounts/events/{accountId}`

#### Réponse de succès (`200 OK`)

```json
[
  {
    "type": "AccountAggregate",
    "aggregateIdentifier": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
    "sequenceNumber": 0,
    "payload": {
      "id": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
      "initialBalance": 1000.0,
      "currency": "MAD"
    }
  },
  {
    "type": "AccountAggregate",
    "aggregateIdentifier": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
    "sequenceNumber": 1,
    "payload": {
      "id": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
      "status": "ACTIVATED"
    }
  }
]
```

---

## 2. Endpoints de Consultation (Query Side)

Les requêtes interrogent la base de lecture relationnelle PostgreSQL issue des projections.

### 2.1. Lister tous les comptes bancaires

Retourne la liste complète des comptes projetés.

- **Méthode :** `GET`
- **URL :** `/query/accounts/all`

#### Réponse de succès (`200 OK`)

```json
[
  {
    "id": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
    "balance": 1150.0,
    "createdAt": "2026-11-19T09:25:34.120Z",
    "status": "ACTIVATED",
    "currency": null
  }
]
```

---

### 2.2. Obtenir le relevé bancaire d'un compte

Fournit les informations du compte ainsi que l'ensemble des opérations enregistrées.

- **Méthode :** `GET`
- **URL :** `/query/accounts/accountsStatement/{accountId}`

#### Modèle de contrat attendu (`200 OK`)

```json
{
  "account": {
    "id": "e756cabc-99b5-439f-a1a7-2cf5ed99e263",
    "balance": 1150.0,
    "createdAt": "2026-11-19T09:25:34.120Z",
    "status": "ACTIVATED",
    "currency": "MAD"
  },
  "operations": [
    {
      "id": 1,
      "date": "2026-11-19T09:26:00Z",
      "amount": 250.0,
      "type": "CREDIT",
      "currency": "MAD"
    },
    {
      "id": 2,
      "date": "2026-11-19T09:27:00Z",
      "amount": 100.0,
      "type": "DEBIT",
      "currency": "MAD"
    }
  ]
}
```

> [!NOTE]
> En raison d'une limitation de sérialisation XStream sur les records Java, cet appel peut retourner une erreur 500 dans la configuration par défaut.

---

### 2.3. Flux réactif en streaming (Server-Sent Events)

Ouvre un flux unidirectionnel temps réel pour recevoir les nouvelles transactions associées à un compte.

- **Méthode :** `GET`
- **URL :** `/query/accounts/watch/{accountId}`
- **En-tête attendu :** `Accept: text/event-stream`

#### Exemple de flux reçu (`text/event-stream`)

```text
data:{"id":3,"date":"2026-11-19T09:30:15Z","amount":150.0,"type":"CREDIT","currency":"MAD"}

data:{"id":4,"date":"2026-11-19T09:32:00Z","amount":50.0,"type":"DEBIT","currency":"MAD"}
```

---

## 3. Codes de Statut et Gestion des Erreurs

| Code HTTP | Scénario | Message / Comportement |
| :--- | :--- | :--- |
| `200 OK` | Succès d'une commande ou consultation | Traitement validé par le bus ou payload JSON renvoyé. |
| `400 Bad Request` | Payload JSON malformé | Corps de requête invalide ou non parsable par Jackson. |
| `200 OK` (Message d'exception) | Rejet métier (Contrôleur de commande) | Ex : `Balance doit etre positive`, `Balance non suffisant`. |
| `500 Internal Server Error` | Erreur interne de sérialisation | Ex : Incompatibilité XStream sur le record du relevé. |

---

[Retour au README principal](../README.md)
