# Quando accedo ad olimpo ma voglio aprire un nuovo tab e copio l'url, mi esce la pagina https://olimpo-stage.telsy.com/#/not_authorized.

## Analisi del Problema
### Il problema era causato dal fatto che i token di autenticazione erano memorizzati nel sessionStorage, che non è condiviso tra tab diversi del browser (a differenza del localStorage).

---

## 1: Migrazione del Sistema di Storage dei Token

#### 1.1 AuthenticationService - Modifica del metodo setStorage()

File: authentication.service.ts

Cambiato da:

```typescript
sessionStorage.setItem(SessionStorage.TOKEN, tokenData.token.access_token);
sessionStorage.setItem(SessionStorage.REFRESH_TOKEN, tokenData.token.refresh_token);
```

Cambiato a:

```typescript
localStorage.setItem(SessionStorage.TOKEN, tokenData.token.access_token);
localStorage.setItem(SessionStorage.REFRESH_TOKEN, tokenData.token.refresh_token);
```


#### 1.2 AuthenticationService - Modifica dei metodi di pulizia storage

Metodi modificati: pulisciStorage(), forceLogout()

Cambiato da: sessionStorage.removeItem()
Cambiato a: localStorage.removeItem() per TOKEN e REFRESH_TOKEN

#### 1.3 AuthenticationService - Modifica dei getter dei token
Metodi modificati: get token(), get getRefreshToken()

Cambiato da: sessionStorage.getItem()
Cambiato a: localStorage.getItem()

#### 1.4 AuthenticationService - Modifica dei metodi utente
Metodi modificati: getUserLogged(), getUserLabels()

Aggiunto:

- Gestione degli errori con try-catch
- Controlli per token null/undefined
- Ritorno di valori sicuri (null, []) invece di crash

---


## 2: Miglioramento dell'AuthGuard

#### 2.1 Aggiunta di import per Observable

File: auth.guard.ts

Aggiunto:
```typescript
import { Observable, of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';
```

#### 2.2 Miglioramento del metodo canActivate()

##### Funzionalità aggiunta:

- Tentativo automatico di refresh del token prima di reindirizzare
- Supporto per operazioni asincrone con Observable
- Gestione degli errori del refresh token

##### Logica implementata:

1. Se c'è un token valido → Accesso consentito
2. Se non c'è token ma c'è refresh token → Tenta refresh automatico
3. Solo se anche il refresh fallisce → Reindirizza a

---


## 3: Aggiornamento di Altri Componenti
#### 3.1 AppComponent

File: app.component.ts

Modifiche:

- ngOnInit(): Cambiato da sessionStorage a localStorage
- Aggiunta gestione sicura per webSocketService.connect()

#### 3.2 Utility Service

File: utility.ts

Modifiche:

- cleanStorage(): Aggiornato per usare localStorage per i token

#### 3.3 UsersButtonComponent

File: users-button.component.ts

Modifiche:

- goToRbac(): Aggiornato per usare localStorage per i token

#### 3.4 ConfigService

File: config.service.ts

Modifiche:

- Aggiunto controllo token prima di caricare preferenze utente
- Previene chiamate API quando l'utente non è autenticato

---


## 4: Risoluzione Errori Runtime

#### 4.1 ProjectButtonComponent - Fix getUserById()

File: project-button.component.ts

Problema: Cannot read properties of undefined (reading 'username')

Soluzione:
```typescript
// PRIMA (causava errore)
return this.users.find((el) => el.user_id === user_id).username;

// DOPO (gestione sicura)
const user = this.users.find((el) => el.user_id === user_id);
return user ? user.username : '';
```

#### 4.2 StatisticsService

File: statistics.service.ts

Aggiunto: Warning nel costruttore per utente non autenticato

---


## 5: Risoluzione Errori delle Direttive di Permessi

#### 5.1 PermissionDirective

File: permission.directive.ts

Problema: Cannot read properties of null (reading 'entity_perms')

Soluzioni implementate:

- Inizializzazione stabile: Token e utente inizializzati in ngOnInit
- Gestione errori: Try-catch per decodifica token
- Controlli null: Verifiche per token/utente/permessi
- Migrazione storage: Da sessionStorage a localStorage

#### 5.2 ExactPermissionDirective

File: exact-permission.directive.ts

Stesse modifiche della PermissionDirective per consistenza

---


## 6: Risoluzione Errore ExpressionChangedAfterItHasBeenCheckedError

#### 6.1 Problema:

Angular rilevava cambiamenti nelle espressioni dopo il change detection

#### 6.2 Soluzioni implementate:

- Inizializzazione in ngOnInit: I valori vengono stabilizzati prima del rendering
- setTimeout: La logica di visualizzazione viene posticipata al prossimo tick
- Flag hasToken: Controllo stabile per evitare cambiamenti di stato

Codice aggiunto alle direttive:

```typescript
ngOnInit(): void {
    const token = localStorage.getItem(SessionStorage.TOKEN);
    this.hasToken = !!token;
    // ... inizializzazione utente
}

ngAfterViewInit(): void {
    setTimeout(() => {
        this.checkPermissions(); // o updateView()
    }, 0);
}
```

---


## Risultati

- Nuovo tab funzionante: URL copiabili e apribili in nuovi tab
- Errore username undefined risolto
- Token condivisi tra tab
- Refresh automatico
- Gestione errori completa
