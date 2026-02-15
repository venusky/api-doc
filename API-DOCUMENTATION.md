# 📡 Documentation API WebSocket - Système de Livraison

## 🔌 Connexion WebSocket

### URL de connexion
```
ws://tounka.vercel.app//socket.io
```

### Authentification
Le client doit envoyer le token JWT lors de la connexion :

```javascript
import io from 'socket.io-client';

const socket = io('https://tounka.vercel.app/', {
  auth: {
    token: 'VOTRE_JWT_TOKEN_ICI'
  }
});

socket.on('connect', () => {
  console.log('Connecté au serveur WebSocket');
});

socket.on('disconnect', () => {
  console.log('Déconnecté du serveur');
});
```

---

## 🚀 FLUX 1 : CRÉATION ET ACCEPTATION DE LIVRAISON

### Étape 1 : Créer une livraison (Client)

**Endpoint REST** : `POST /deliveries`

**Body** :
```json
{
  "clientId": "uuid-du-client",
  "parcelId": "uuid-du-colis",
  "pickupLocation": {
    "lat": 6.1319,
    "lng": 1.2228,
    "address": "Adresse de départ"
  },
  "deliveryLocation": {
    "lat": 6.1500,
    "lng": 1.2400,
    "address": "Adresse de livraison"
  }
}
```

**Réponse** :
```json
{
  "id": "delivery-uuid",
  "deliveryCode": "DEL-001234",
  "status": "PENDING",
  "clientId": "uuid-du-client",
  "parcelId": "uuid-du-colis",
  "createdAt": "2026-02-15T18:00:00Z"
}
```

---

### Étape 2 : Assigner un livreur (Admin ou Auto)

**Endpoint REST** : `POST /deliveries/:id/assign`

**Params** :
- `id` : UUID de la livraison

**Body** :
```json
{
  "courierId": "uuid-du-livreur"
}
```

**Réponse** :
```json
{
  "id": "delivery-uuid",
  "courierId": "uuid-du-livreur",
  "status": "ASSIGNED"
}
```

---

### Étape 3 : Livreur accepte la livraison

**Endpoint REST** : `POST /deliveries/:id/accept`

**Params** :
- `id` : UUID de la livraison

**Body** :
```json
{
  "lat": 6.1319,
  "lng": 1.2228
}
```

**Réponse** :
```json
{
  "id": "delivery-uuid",
  "status": "ACCEPTED",
  "courierId": "uuid-du-livreur"
}
```

**🔔 Événements WebSocket émis automatiquement** :

#### A. Le CLIENT reçoit :

**Événement** : `delivery-accepted`
```javascript
socket.on('delivery-accepted', (data) => {
  console.log(data);
  /*
  {
    deliveryId: "delivery-uuid",
    courierId: "uuid-du-livreur",
    message: "Votre livreur a accepté la course !"
  }
  */
});
```

**Événement** : `start-tracking`
```javascript
socket.on('start-tracking', (data) => {
  console.log('Début du suivi', data);
  /*
  {
    courierId: "uuid-du-livreur",
    deliveryId: "delivery-uuid"
  }
  */
  // Initialiser la carte de suivi ici
});
```

**Événement** : `delivery-status-update`
```javascript
socket.on('delivery-status-update', (data) => {
  console.log('Changement de statut', data);
  /*
  {
    deliveryId: "delivery-uuid",
    clientId: "uuid-du-client",
    courierId: "uuid-du-livreur",
    status: "ACCEPTED",
    location: { lat: 6.1319, lng: 1.2228 }
  }
  */
});
```

#### B. Le LIVREUR reçoit aussi :

**Événement** : `delivery-status-update` (même structure)

---

## 📍 FLUX 2 : SUIVI DE LOCALISATION EN TEMPS RÉEL

### Livreur envoie sa position

**Événement WebSocket à émettre** : `update-location`

```javascript
// Le livreur envoie sa position toutes les 5-10 secondes
setInterval(() => {
  socket.emit('update-location', {
    lat: currentLat,
    lng: currentLng
  });
}, 5000);
```

**Paramètres** :
```json
{
  "lat": 6.1319,
  "lng": 1.2228
}
```

---

### Le CLIENT reçoit la position du livreur

**Événement WebSocket reçu** : `courier-location-update`

```javascript
socket.on('courier-location-update', (data) => {
  console.log('Position du livreur mise à jour', data);
  /*
  {
    deliveryId: "delivery-uuid",
    courierId: "uuid-du-livreur",
    lat: 6.1319,
    lng: 1.2228,
    timestamp: "2026-02-15T18:30:00Z"
  }
  */
  
  // Mettre à jour le marqueur sur la carte
  updateMapMarker(data.lat, data.lng);
});
```

---

## 📦 FLUX 3 : CHANGEMENTS DE STATUT DE LIVRAISON

### 3.1 Colis récupéré

**Endpoint REST** : `POST /deliveries/:id/pickup`

**Body** :
```json
{
  "location": {
    "lat": 6.1319,
    "lng": 1.2228
  }
}
```

**🔔 Événement émis** : `delivery-status-update`
```javascript
// Client + Livreur reçoivent
{
  deliveryId: "delivery-uuid",
  clientId: "uuid-du-client",
  courierId: "uuid-du-livreur",
  status: "PICKED_UP",
  location: { lat: 6.1319, lng: 1.2228 }
}
```

---

### 3.2 En transit

**Endpoint REST** : `POST /deliveries/:id/in-transit`

**Body** :
```json
{
  "location": {
    "lat": 6.1400,
    "lng": 1.2300
  }
}
```

**🔔 Événement émis** : `delivery-status-update`
```javascript
// Client + Livreur reçoivent
{
  status: "IN_TRANSIT",
  // ... autres champs
}
```

---

### 3.3 Livraison terminée

**Endpoint REST** : `POST /deliveries/:id/delivered`

**Body** :
```json
{
  "location": {
    "lat": 6.1500,
    "lng": 1.2400
  }
}
```

**🔔 Événement émis** : `delivery-status-update`
```javascript
// Client + Livreur reçoivent
{
  status: "DELIVERED",
  // ... autres champs
}
```

---

## ❌ FLUX 4 : ANNULATION

### 4.1 Annulation par le CLIENT

**Endpoint REST** : `POST /deliveries/:id/cancel-by-client`

**Body** :
```json
{
  "clientId": "uuid-du-client",
  "reason": "Je n'ai plus besoin de la livraison"
}
```

**Réponse** :
```json
{
  "id": "delivery-uuid",
  "status": "CANCELED",
  "canceledReason": "Je n'ai plus besoin de la livraison",
  "canceledAt": "2026-02-15T18:45:00Z"
}
```

**🔔 Événement WebSocket émis** : `delivery-canceled`

```javascript
// Le LIVREUR reçoit
socket.on('delivery-canceled', (data) => {
  console.log('Livraison annulée', data);
  /*
  {
    courierId: "uuid-du-livreur",
    deliveryId: "delivery-uuid",
    reason: "Je n'ai plus besoin de la livraison"
  }
  */
  
  // Afficher une notification au livreur
  showNotification('La livraison a été annulée par le client');
});
```

---

### 4.2 Annulation par le LIVREUR

**Endpoint REST** : `POST /deliveries/:id/cancel-by-delivery`

**Body** :
```json
{
  "reason": "Problème de véhicule",
  "notifyClient": true
}
```

---

## 🟢🔴 FLUX 5 : DISPONIBILITÉ DU LIVREUR

### Changer la disponibilité

**Endpoint REST** : `PATCH /deliveries/couriers/:id/availability`

**Params** :
- `id` : UUID du livreur

**Body** :
```json
{
  "isAvailable": true
}
```

**Réponse** :
```json
{
  "id": "uuid-du-livreur",
  "isAvailable": true,
  "currentLocation": { "lat": 6.1319, "lng": 1.2228 },
  "user": {
    "id": "uuid-user",
    "fullName": "Jean Dupont"
  }
}
```

**🔔 Événement WebSocket émis** : `courier-availability-update`

```javascript
// TOUS LES UTILISATEURS connectés reçoivent cet événement
socket.on('courier-availability-update', (data) => {
  console.log('Disponibilité du livreur mise à jour', data);
  /*
  {
    courierId: "uuid-du-livreur",
    isAvailable: true,
    courierName: "Jean Dupont",
    currentLocation: { lat: 6.1319, lng: 1.2228 },
    timestamp: "2026-02-15T19:00:00Z"
  }
  */
  
  // Mettre à jour la liste des livreurs disponibles dans l'interface
  updateCourierList(data);
});
```

---

## 💝 FLUX 6 : FAVORIS (Bonus)

### Ajouter un favori

**Endpoint REST** : `POST /favorites`

**Body** :
```json
{
  "userId": "uuid-user",
  "productId": "uuid-product"
}
```

**🔔 Événement WebSocket émis** : `favorite_updated`

```javascript
socket.on('favorite_updated', (data) => {
  console.log('Favori mis à jour', data);
  /*
  {
    action: "added",
    productId: "uuid-product"
  }
  */
});
```

---

## 📊 ENDPOINTS UTILES

### Lister les livreurs disponibles

**Endpoint** : `GET /deliveries/couriers/available`

**Query params** :
- `lat` : Latitude (ex: 6.1319)
- `lng` : Longitude (ex: 1.2228)
- `radiusKm` : Rayon en km (ex: 5)

**Exemple** :
```
GET /deliveries/couriers/available?lat=6.1319&lng=1.2228&radiusKm=5
```

**Réponse** :
```json
[
  {
    "id": "courier-uuid-1",
    "isAvailable": true,
    "currentLocation": { "lat": 6.1300, "lng": 1.2200 },
    "user": {
      "fullName": "Jean Dupont",
      "phone": "+228..."
    }
  }
]
```

---

### Obtenir la position actuelle d'un livreur

**Endpoint** : `GET /deliveries/couriers/:id/location`

**Réponse** :
```json
{
  "lat": 6.1319,
  "lng": 1.2228
}
```

---

### Historique des événements d'une livraison

**Endpoint** : `GET /deliveries/:id/events`

**Réponse** :
```json
[
  {
    "id": "event-uuid",
    "deliveryId": "delivery-uuid",
    "eventType": "ACCEPTED",
    "eventTime": "2026-02-15T18:00:00Z",
    "location": { "lat": 6.1319, "lng": 1.2228 },
    "metadata": {}
  },
  {
    "id": "event-uuid-2",
    "eventType": "PICKED_UP",
    "eventTime": "2026-02-15T18:15:00Z"
  }
]
```

---

### Lister toutes les livraisons avec filtres

**Endpoint** : `GET /deliveries`

**Query params** :
- `status` (optionnel) : Filtrer par statut (PENDING, ASSIGNED, ACCEPTED, etc.)
- `courierId` (optionnel) : Filtrer par livreur
- `clientId` (optionnel) : Filtrer par client
- `dateFrom` (optionnel) : Date de début (format: YYYY-MM-DD)
- `dateTo` (optionnel) : Date de fin (format: YYYY-MM-DD)

**Exemple** :
```
GET /deliveries?status=ACCEPTED&courierId=courier-uuid
```

**Réponse** :
```json
[
  {
    "id": "delivery-uuid",
    "status": "ACCEPTED",
    "deliveryCode": "DEL-001234",
    "parcel": {...},
    "courier": {...},
    "client": {...},
    "events": [...]
  }
]
```

---

## 📝 RÉSUMÉ DES ÉVÉNEMENTS WEBSOCKET

| Événement | Qui reçoit | Quand | Données |
|-----------|------------|-------|---------|
| `delivery-accepted` | Client uniquement | Quand le livreur accepte | `{deliveryId, courierId, message}` |
| `start-tracking` | Client uniquement | Après acceptation | `{deliveryId, courierId}` |
| `delivery-status-update` | Client + Livreur | À chaque changement de statut | `{deliveryId, clientId, courierId, status, location}` |
| `courier-location-update` | Client avec livraison active | Toutes les 5-10s pendant la livraison | `{deliveryId, courierId, lat, lng, timestamp}` |
| `delivery-canceled` | Livreur uniquement | Quand client annule | `{courierId, deliveryId, reason}` |
| `courier-availability-update` | TOUS les utilisateurs | Quand livreur change disponibilité | `{courierId, isAvailable, courierName, currentLocation, timestamp}` |
| `favorite_updated` | Utilisateur concerné | Ajout/suppression favori | `{action, productId}` |

---

## 🎯 STATUTS DE LIVRAISON

```typescript
enum DeliveryStatus {
  PENDING = 'PENDING',           // En attente d'assignation
  ASSIGNED = 'ASSIGNED',         // Livreur assigné
  ACCEPTED = 'ACCEPTED',         // Livreur a accepté
  PICKED_UP = 'PICKED_UP',       // Colis récupéré
  IN_TRANSIT = 'IN_TRANSIT',     // En cours de livraison
  DELIVERED = 'DELIVERED',       // Livré
  CANCELED = 'CANCELED'          // Annulé
}
```

---

## 🔄 FLUX COMPLET - SCÉNARIO TYPIQUE

### 1. Client crée une livraison
```
POST /deliveries
```

### 2. Admin/Système assigne un livreur
```
POST /deliveries/:id/assign
```

### 3. Livreur accepte
```
POST /deliveries/:id/accept
→ WebSocket: delivery-accepted (client reçoit)
→ WebSocket: start-tracking (client reçoit)
→ WebSocket: delivery-status-update (client + livreur reçoivent)
```

### 4. Livreur envoie sa position en temps réel
```
WebSocket emit: update-location (toutes les 5-10s)
→ WebSocket: courier-location-update (client reçoit)
```

### 5. Livreur récupère le colis
```
POST /deliveries/:id/pickup
→ WebSocket: delivery-status-update
```

### 6. Livreur démarre le transport
```
POST /deliveries/:id/in-transit
→ WebSocket: delivery-status-update
```

### 7. Livraison terminée
```
POST /deliveries/:id/delivered
→ WebSocket: delivery-status-update
```

---

## ⚠️ GESTION DES ERREURS

### Connexion WebSocket refusée
- **Cause** : Token JWT invalide ou expiré
- **Solution** : Vérifier que le token est valide et bien envoyé dans `auth.token`

### Événements non reçus
- **Cause** : Socket non connecté ou nom d'événement incorrect
- **Solution** : 
  - Vérifier que `socket.connected === true` avant d'écouter
  - Vérifier l'orthographe exacte des événements (sensible à la casse)

### Déconnexion fréquente
- **Cause** : Problèmes réseau ou token expiré
- **Solution** : Implémenter une reconnexion automatique
  
```javascript
socket.on('disconnect', (reason) => {
  if (reason === 'io server disconnect') {
    // Le serveur a déconnecté le socket, il faut se reconnecter manuellement
    socket.connect();
  }
  // Sinon, socket.io reconnectera automatiquement
});
```

---

## 💡 BONNES PRATIQUES

### 1. Gestion de la connexion

```javascript
const socket = io('http://votre-domaine', {
  auth: {
    token: getUserToken()
  },
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  reconnectionAttempts: 5
});

// Gérer l'état de connexion
socket.on('connect', () => {
  console.log('✅ Connecté');
  // Réinitialiser les écoutes si nécessaire
});

socket.on('connect_error', (error) => {
  console.error('❌ Erreur de connexion:', error);
  // Afficher un message à l'utilisateur
});

socket.on('disconnect', (reason) => {
  console.log('⚠️ Déconnecté:', reason);
  // Informer l'utilisateur
});
```

### 2. Nettoyage des écouteurs

```javascript
// Dans un composant React
useEffect(() => {
  const handleLocationUpdate = (data) => {
    updateMap(data);
  };
  
  socket.on('courier-location-update', handleLocationUpdate);
  
  // Nettoyage à la destruction du composant
  return () => {
    socket.off('courier-location-update', handleLocationUpdate);
  };
}, []);
```

### 3. Envoi optimisé de la position

```javascript
// Utiliser un throttle pour éviter de surcharger le serveur
import { throttle } from 'lodash';

const sendLocation = throttle((lat, lng) => {
  socket.emit('update-location', { lat, lng });
}, 5000); // Maximum une fois toutes les 5 secondes

// Dans le composant de tracking
navigator.geolocation.watchPosition((position) => {
  sendLocation(
    position.coords.latitude,
    position.coords.longitude
  );
});
```

---

## 🔒 SÉCURITÉ

### Token JWT
- Le token doit être valide et non expiré
- Le token contient l'ID de l'utilisateur (`sub` claim)
- Le serveur vérifie automatiquement le token à la connexion

### Validation des données
- Toutes les données envoyées sont validées côté serveur
- Les coordonnées GPS doivent être des nombres valides
- Les UUID doivent être au format valide

### Isolation des données
- Chaque utilisateur ne reçoit que les événements qui le concernent
- Les rooms WebSocket sont créées automatiquement par utilisateur
- Exception : `courier-availability-update` est un broadcast global

---

## 📱 EXEMPLE D'IMPLÉMENTATION REACT NATIVE

```javascript
import { useEffect, useState } from 'react';
import io from 'socket.io-client';
import AsyncStorage from '@react-native-async-storage/async-storage';

const useDeliveryTracking = (deliveryId) => {
  const [socket, setSocket] = useState(null);
  const [courierLocation, setCourierLocation] = useState(null);
  const [deliveryStatus, setDeliveryStatus] = useState(null);

  useEffect(() => {
    const initSocket = async () => {
      const token = await AsyncStorage.getItem('jwt_token');
      
      const newSocket = io('http://your-api.com', {
        auth: { token }
      });

      newSocket.on('connect', () => {
        console.log('Connected to tracking');
      });

      newSocket.on('delivery-accepted', (data) => {
        Alert.alert('Bonne nouvelle !', data.message);
      });

      newSocket.on('courier-location-update', (data) => {
        if (data.deliveryId === deliveryId) {
          setCourierLocation({ lat: data.lat, lng: data.lng });
        }
      });

      newSocket.on('delivery-status-update', (data) => {
        if (data.deliveryId === deliveryId) {
          setDeliveryStatus(data.status);
        }
      });

      setSocket(newSocket);
    };

    initSocket();

    return () => {
      if (socket) {
        socket.disconnect();
      }
    };
  }, [deliveryId]);

  return { courierLocation, deliveryStatus };
};

export default useDeliveryTracking;
```

---

## 🧪 TESTS

### Tester la connexion WebSocket

```javascript
// test-websocket.js
const io = require('socket.io-client');

const socket = io('http://localhost:3000', {
  auth: {
    token: 'VOTRE_TOKEN_JWT'
  }
});

socket.on('connect', () => {
  console.log('✅ Connexion réussie');
  
  // Tester l'envoi de position
  socket.emit('update-location', {
    lat: 6.1319,
    lng: 1.2228
  });
  
  console.log('📍 Position envoyée');
});

socket.on('courier-location-update', (data) => {
  console.log('📍 Position reçue:', data);
});

socket.on('connect_error', (error) => {
  console.error('❌ Erreur:', error.message);
});
```

---

## 📞 SUPPORT

Pour toute question ou problème :
- Vérifiez d'abord cette documentation
- Consultez les logs du serveur
- Vérifiez que votre token JWT est valide
- Assurez-vous que les événements WebSocket sont correctement écoutés

---

**Version** : 1.0.0  
**Dernière mise à jour** : 15 février 2026  
**API Base URL** : `https://tounka.vercel.app/`
