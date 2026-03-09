# TOUNKA EXPRESS — WebSocket API
### Guide d'implémentation Frontend
 
> **Base URL** : `https://api.tounkaexpress.com`  
> **Dernière mise à jour** : Mars 2026

---

## importants

- Toute connexion **sans token JWT valide est immédiatement rejetée** côté serveur
- Chaque utilisateur rejoint automatiquement sa **room personnelle** `user_<userId>` à la connexion — pas d'action manuelle requise
- Les événements sont **case-sensitive** : respectez l'orthographe exacte
- **Bug connu** : dans `DeliveriesService`, l'événement émis est `deliveryStatusUpdate` (camelCase) mais le gateway écoute `deliveryStatusUpdated` — à corriger côté back ou à surveiller

---

## Connexion

```typescript
import io, { Socket } from 'socket.io-client';

const socket: Socket = io('https://api.tounkaexpress.com', {
  auth: {
    token: 'JWT_TOKEN', // obligatoire
  },
  // Alternative : headers: { token: 'JWT_TOKEN' }
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  reconnectionAttempts: 5,
});

socket.on('connect',       () => console.log('Connecté, id:', socket.id));
socket.on('connect_error', (err) => console.error('Refusé:', err.message));
socket.on('disconnect',    (reason) => {
  console.warn('Déconnecté:', reason);
  if (reason === 'io server disconnect') socket.connect(); // token expiré → reconnecter
});
```

---

## Événements envoyés par le client

### `update-location` — Livreur uniquement

Le livreur envoie sa position GPS. Le serveur persiste la position en base **et** redistribue aux clients ayant une livraison active avec ce livreur.

```typescript
// À appeler toutes les 5-10 secondes (utiliser un throttle)
socket.emit('update-location', {
  lat: 6.1319,
  lng: 1.2228,
});
```

**Payload :**
```typescript
interface UpdateLocationDto {
  lat: number;
  lng: number;
}
```

**Exemple avec throttle (recommandé) :**
```typescript
import { throttle } from 'lodash';

const sendLocation = throttle((lat: number, lng: number) => {
  socket.emit('update-location', { lat, lng });
}, 5000);

navigator.geolocation.watchPosition(({ coords }) => {
  sendLocation(coords.latitude, coords.longitude);
});
```

> **Chaîne interne** :  
> `update-location` → `livreur.location.update` → BDD mise à jour → `livreur.location.broadcast` → `courier.location.for.client` → `courier-location-update` envoyé au(x) client(s)

---

## Événements reçus par le client

---

### Module Livraison

---

#### `delivery-accepted`
**Destinataire :** Client  
**Déclencheur :** Le livreur appelle `POST /deliveries/:id/accept`

```typescript
socket.on('delivery-accepted', (data: {
  deliveryId: string;
  courierId:  string;
  message:    string; // "Votre livreur a accepté la course !"
}) => {
  showToast(data.message);
});
```

---

#### `start-tracking`
**Destinataire :** Client  
**Déclencheur :** Émis automatiquement juste après `delivery-accepted`

C'est le signal pour **initialiser la carte** et commencer à afficher la position du livreur.

```typescript
socket.on('start-tracking', (data: {
  courierId:  string;
  deliveryId: string;
}) => {
  initMap();
  setActiveDeliveryId(data.deliveryId);
});
```

---

#### `delivery-status-update`
**Destinataire :** Client **ET** Livreur  
**Déclencheur :** Tout changement de statut via les endpoints REST

```typescript
socket.on('delivery-status-update', (data: {
  deliveryId: string;
  clientId:   string;
  courierId?: string;
  status:     DeliveryStatus;
  location?:  { lat: number; lng: number };
}) => {
  updateDeliveryUI(data.status);
});
```

**Statuts et transitions autorisées :**

```
REQUESTED → ASSIGNED → ACCEPTED → PICKED_UP → IN_TRANSIT → DELIVERED
     ↓            ↓          ↓
  CANCELED    CANCELED   CANCELED
```

| Statut | Description | Action UI suggérée |
|--------|-------------|-------------------|
| `REQUESTED` | En attente d'assignation | Spinner "Recherche d'un livreur…" |
| `ASSIGNED` | Livreur assigné | "Un livreur est en route vers vous" |
| `ACCEPTED` | Livreur a accepté | Afficher la carte de suivi |
| `PICKED_UP` | Colis récupéré | "Votre colis est pris en charge" |
| `IN_TRANSIT` | En livraison | Suivi temps réel actif |
| `DELIVERED` | Livré | Confirmation + notation |
| `CANCELED` | Annulé | Message d'annulation |

---

#### `courier-location-update`
**Destinataire :** Client (uniquement si livraison active avec ce livreur)  
**Déclencheur :** Chaque `update-location` émis par le livreur

```typescript
socket.on('courier-location-update', (data: {
  deliveryId: string;
  courierId:  string;
  lat:        number;
  lng:        number;
  timestamp:  Date;
}) => {
  if (data.deliveryId === currentDeliveryId) {
    updateMarkerOnMap(data.lat, data.lng);
  }
});
```

---

#### `delivery-canceled`
**Destinataire :** Livreur uniquement  
**Déclencheur :** Client annule via `POST /deliveries/:id/cancel-by-client`

```typescript
socket.on('delivery-canceled', (data: {
  courierId:  string;
  deliveryId: string;
  reason?:    string;
}) => {
  showAlert(`Livraison annulée${data.reason ? ' : ' + data.reason : ''}`);
  resetCourierState();
});
```

> **Note** : quand ce cas se produit, le serveur rend automatiquement le livreur `isAvailable: true` en base et crée une notification `in_app`.

---

#### `courier-availability-update`
**Destinataire :** **Tous les utilisateurs connectés** (broadcast global)  
**Déclencheur :** `PATCH /deliveries/couriers/:id/availability`

```typescript
socket.on('courier-availability-update', (data: {
  courierId:        string;
  isAvailable:      boolean;
  courierName?:     string;
  currentLocation?: { lat: number; lng: number };
  timestamp:        Date;
}) => {
  updateCourierListUI(data);
});
```

---

### Module Messagerie

---

#### `new-message`
**Destinataire :** Destinataire du message

```typescript
socket.on('new-message', (message: MessageObject) => {
  appendToConversation(message);
  playNotificationSound();
});
```

---

#### `message-read`
**Destinataire :** Expéditeur du message

```typescript
socket.on('message-read', (data: {
  messageId:      string;
  conversationId: string;
  readBy:         string; // userId
}) => {
  setMessageStatus(data.messageId, 'read'); // ✓✓ double coche
});
```

---

#### `conversation-marked-read`
**Destinataire :** Utilisateur qui a lu

```typescript
socket.on('conversation-marked-read', (data: {
  conversationId: string;
  readBy:         string;
}) => {
  resetUnreadBadge(data.conversationId);
});
```

---

### Module Appels

---

#### `incoming-call`
**Destinataire :** Destinataire de l'appel

```typescript
socket.on('incoming-call', (call: {
  callerId:   string;
  receiverId: string;
  startedAt?: Date;
  endedAt?:   Date;
  type?:      'video' | 'audio';
}) => {
  showIncomingCallModal(call);
});
```

---

#### `call-status-update`
**Destinataire :** Caller **ET** Receiver

```typescript
socket.on('call-status-update', (data: {
  callId: string;
  status: string;
  call: {
    callerId:   string;
    receiverId: string;
    startedAt?: Date;
    endedAt?:   Date;
    type?:      'video' | 'audio';
  };
}) => {
  updateCallUI(data.status);
});
```

---

### Module Favoris

#### `favorite_updated`
**Destinataire :** Utilisateur concerné  
**Déclencheur :** Ajout ou suppression d'un favori (REST)

```typescript
socket.on('favorite_updated', (data: {
  action:    'added' | 'removed';
  productId: string;
}) => {
  syncFavoritesUI(data);
});
```

---

## Tableau de référence rapide

| Événement | ← / → | Destinataire | Module | Déclencheur |
|-----------|:-----:|-------------|--------|-------------|
| `update-location` | → | Serveur | Livraison | Livreur |
| `delivery-accepted` | ← | Client | Livraison | `POST /accept` |
| `start-tracking` | ← | Client | Livraison | `POST /accept` |
| `delivery-status-update` | ← | Client + Livreur | Livraison | Tout changement de statut |
| `courier-location-update` | ← | Client | Livraison | `update-location` du livreur |
| `delivery-canceled` | ← | Livreur | Livraison | `POST /cancel-by-client` |
| `courier-availability-update` | ← | **Tous** | Livraison | `PATCH /availability` |
| `new-message` | ← | Destinataire | Messagerie | Envoi message |
| `message-read` | ← | Expéditeur | Messagerie | Lecture message |
| `conversation-marked-read` | ← | Utilisateur | Messagerie | Lecture conversation |
| `incoming-call` | ← | Destinataire | Appels | Initiation appel |
| `call-status-update` | ← | Caller + Receiver | Appels | Changement état appel |
| `favorite_updated` | ← | Utilisateur | Favoris | Ajout/suppression favori |

---


## Sécurité

- Le token JWT est lu depuis `auth.token` ou `headers.token`
- L'`userId` est extrait du token via `RealtimeAuthService.verifyToken()`
- Chaque utilisateur ne reçoit **que les événements de sa room** `user_<userId>`
- Exception : `courier-availability-update` est un **broadcast global non filtré**

---