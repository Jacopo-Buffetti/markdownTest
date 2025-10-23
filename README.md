# PERSISTENZA LOGIN TRA TAB

### ANALISI INIZIALE DEL PROBLEMA

##### Problema Principale:

Quando un utente faceva login su Olimpo e poi copiava l'URL per aprirlo in un nuovo tab, veniva reindirizzato a /not_authorized invece di accedere alla pagina desiderata.

#### Analisi:

- __sessionStorage vs localStorage__: I token erano salvati in sessionStorage
- __Isolamento dei tab__: sessionStorage è isolato per ogni tab del browser
- __AuthGuard__: Verificava la presenza del token per autorizzare l'accesso
- __Flusso di errore__: Nuovo tab → Nessun token → AuthGuard nega accesso → Redirect a not_authorized

### MODIFICA 1: SISTEMA DI STORAGE DEI TOKEN

__File__: `authentication.service.ts`

__METODO__ `setStorage()`

###### Modifica Specifica:

```typescript
// PRIMA:
sessionStorage.setItem(SessionStorage.TOKEN, tokenData.token.access_token);
sessionStorage.setItem(SessionStorage.REFRESH_TOKEN, tokenData.token.refresh_token);

// DOPO:
localStorage.setItem(SessionStorage.TOKEN, tokenData.token.access_token);
localStorage.setItem(SessionStorage.REFRESH_TOKEN, tokenData.token.refresh_token);
```

###### Motivazione Tecnica:

- __sessionStorage__: Dati isolati per ogni tab/finestra del browser
- __localStorage__: Dati condivisi tra tutti i tab dello stesso dominio
- __Persistenza__: localStorage sopravvive alla chiusura dei tab (fino a logout esplicito)
- __Sicurezza__: Mantiene la sicurezza perché limitato al dominio

###### Risultato:

Ora i token sono accessibili da qualsiasi tab dello stesso dominio.

---

__METODO__ `pulisciStorage()`

###### Modifica Specifica:

```typescript
// PRIMA:
sessionStorage.removeItem(SessionStorage.TOKEN);
sessionStorage.removeItem(SessionStorage.REFRESH_TOKEN);

// DOPO:
localStorage.removeItem(SessionStorage.TOKEN);
localStorage.removeItem(SessionStorage.REFRESH_TOKEN);
```

###### Motivazione Tecnica:

- __Consistenza__: Rimuove i token dallo stesso storage dove li salva
- __Pulizia completa__: Assicura la rimozione completa dei dati di autenticazione
- __Sicurezza__: Previene token orfani in storage diversi

__METODO__ `forceLogout()`

###### Modifica Identica a pulisciStorage()

###### Motivazione:

Stesso principio di consistenza e pulizia completa.

---

__GETTER__ `get token()` e `get getRefreshToken()`

###### Modifica Specifica:

```typescript
// PRIMA:
get token(): string | null {
    return sessionStorage.getItem(SessionStorage.TOKEN);
}

// DOPO:
get token(): string | null {
    return localStorage.getItem(SessionStorage.TOKEN);
}
```

###### Motivazione Tecnica:

- __Coerenza__: Deve leggere dallo stesso storage dove scrive
- __Accessibilità__: Permette l'accesso ai token da qualsiasi tab
- __Uniformità API__: Mantiene la stessa interfaccia pubblica

---

__METODO__ `getUserLogged()` - __PRIMA VERSIONE__

###### Modifica Specifica:

```typescript
// PRIMA:
getUserLogged() {
    const userLogged = this.appHelpers.decodeJwt(sessionStorage.getItem(SessionStorage.TOKEN)!);
    return userLogged;
}

// DOPO:
getUserLogged() {
    try {
        const token = localStorage.getItem(SessionStorage.TOKEN);
        if (!token) {
            return null;
        }
        const userLogged = this.appHelpers.decodeJwt(token);
        return userLogged;
    } catch (error) {
        console.error('Errore nel decodificare il token utente:', error);
        return null;
    }
}
```

###### Motivazione Tecnica:

- __Token Null__: Gestisce il caso in cui il token non esista
- __Gestione degli errori__: Cattura errori di decodifica JWT malformati
- __Coerenza di archiviazione__: Usa localStorage invece di sessionStorage

###### Benefici:

- Previene crash dell'applicazione
- Migliore debugging con log degli errori
- Comportamento predicibile

---

__METODO__ `getUserLabels()` - __MIGLIORAMENTO__

###### Modifica Specifica:

```typescript
// PRIMA:
getUserLabels() {
    const userLogged = this.appHelpers.decodeJwt(localStorage.getItem(SessionStorage.TOKEN)!);
    const userEntityLabels = userLogged.labels;
    const allEntityLabels = ALL_ENTITY_LABELS.filter((el) => el.visible);
    return allEntityLabels.filter((el) => userEntityLabels.find((entEl: any) => entEl === el.label));
}

// DOPO:
getUserLabels() {
    try {
        const token = localStorage.getItem(SessionStorage.TOKEN);
        if (!token) {
            return [];
        }
        const userLogged = this.appHelpers.decodeJwt(token);
        if (!userLogged || !userLogged.labels) {
            return [];
        }
        const userEntityLabels = userLogged.labels;
        const allEntityLabels = ALL_ENTITY_LABELS.filter((el) => el.visible);
        return allEntityLabels.filter((el) => userEntityLabels.find((entEl: any) => entEl === el.label));
    } catch (error) {
        console.error('Errore nel caricamento dei label utente:', error);
        return [];
    }
}
```

###### Motivazione Tecnica:

- __Valori predefiniti__: Ritorna array vuoto invece di undefined
- __Controlli null annidati__: Verifica token, userLogged, e labels
- __Tipo di ritorno coerente__: Sempre ritorna un array

---

🛡️ MODIFICA 2: MIGLIORAMENTO DELL'AUTHGUARD
File: auth.guard.ts
AGGIUNTA IMPORT PER REACTIVE PROGRAMMING
🔧 Modifica Specifica:

📋 Motivazione Tecnica:

Asynchronous Operations: Necessario per operazioni di refresh token
RxJS Integration: Integrazione con l'architettura Angular/RxJS esistente
Error Handling: Operators per gestione errori elegante
REFACTORING DEL METODO canActivate()
🔧 Modifica Specifica:

📋 Motivazione Tecnica:

Intelligent Recovery: Tenta il refresh automatico invece di fallire immediatamente
Async Support: Ritorna Observable per operazioni asincrone
Graceful Fallback: Solo se il refresh fallisce, reindirizza a not_authorized
Code Reuse: Estrae la logica di reindirizzamento in handleNoAuth()
Better UX: L'utente potrebbe non accorgersi del refresh automatico
🎯 Flusso Logico Migliorato:

Token presente? → Accesso immediato
Token assente ma refresh presente? → Tenta refresh
Refresh riuscito? → Salva nuovo token e consenti accesso
Refresh fallito? → Reindirizza a not_authorized
Nessun token? → Reindirizza a not_authorized
NUOVO METODO handleNoAuth()
🔧 Aggiunta:

📋 Motivazione Tecnica:

DRY Principle: Evita duplicazione di codice
Single Responsibility: Metodo dedicato alla gestione dell'autorizzazione negata
Maintainability: Più facile modificare la logica in futuro
🔧 MODIFICA 3: AGGIORNAMENTO COMPONENTI ESISTENTI
File: app.component.ts
METODO ngOnInit()
🔧 Modifica Specifica:

📋 Motivazione: Consistenza con il nuovo sistema di storage.

GESTIONE WEBSOCKET SICURA
🔧 Modifica Specifica:

📋 Motivazione Tecnica:

Null Safety: Previene errori se getUserLogged() ritorna null
Property Check: Verifica l'esistenza dell'email prima di usarla
WebSocket Reliability: Evita connessioni WebSocket con dati non validi
File: config.service.ts
CARICAMENTO CONDIZIONALE DELLE PREFERENZE
🔧 Modifica Specifica:

📋 Motivazione Tecnica:

Performance: Evita chiamate API non necessarie
Error Reduction: Previene errori quando l'utente non è autenticato
Logical Flow: Le preferenze utente hanno senso solo se c'è un utente
🐛 MODIFICA 4: RISOLUZIONE ERRORI RUNTIME
File: project-button.component.ts
METODO getUserById() - BUG FIX
🔧 Modifica Specifica:

📋 Analisi Tecnica del Bug:

Array.find() behavior: Ritorna undefined se nessun elemento è trovato
Property access on undefined: undefined.username genera TypeError
Template binding: L'errore si propaga al template HTML
🎯 Soluzione Implementata:

Safe Assignment: Assegna il risultato di find() a una variabile
Null Check: Verifica se l'utente esiste prima di accedere alle proprietà
Default Value: Ritorna stringa vuota se l'utente non è trovato
File: statistics.service.ts
COSTRUTTORE CON WARNING
🔧 Modifica Specifica:

📋 Motivazione:

Debugging Aid: Aiuta a identificare problemi di inizializzazione
Early Warning: Notifica potenziali problemi prima che causino errori
Development Support: Utile durante lo sviluppo e testing
🔐 MODIFICA 5: DIRETTIVE DI PERMESSI
Problema Identificato:
Le direttive *appPermission e *appExactPermission generavano errori:
Cannot read properties of null (reading 'entity_perms')
ExpressionChangedAfterItHasBeenCheckedError
File: permission.directive.ts
REFACTORING COMPLETO DELLA DIRETTIVA
🔧 Prima Versione (PROBLEMATICA):

🔧 Versione Finale (SICURA):

📋 Analisi Tecnica dei Miglioramenti:

Stable Initialization:

ngOnInit() è chiamato prima del rendering
I valori sono stabilizzati prima che il template li usi
Previene cambiamenti durante il change detection
Error Handling:

Try-catch per decodifica JWT
Gestione sicura di token corrotti o malformati
Log degli errori per debugging
Null Safety:

Controlli multipli: token, user, entity_perms
Tipo nullable: User | null
Early return invece di crash
ExpressionChangedAfterItHasBeenCheckedError Prevention:

setTimeout(() => {}, 0) posticipa l'esecuzione
Angular completa il change detection prima della logica della direttiva
Previene modifiche al DOM durante il controllo
Storage Consistency:

Usa localStorage come tutto il resto del sistema
Accesso ai token condivisi tra i tab
Flag Optimization:

hasToken boolean per controlli rapidi
Evita chiamate ripetute a localStorage
File: exact-permission.directive.ts
STESSE MODIFICHE PER CONSISTENZA
📋 Motivazione:

Entrambe le direttive hanno la stessa architettura
Stesso pattern di errori e soluzioni
Consistenza nell'interfaccia e comportamento
⚡ MODIFICA 6: ALTRI AGGIORNAMENTI DI CONSISTENZA
File: utility.ts
cleanStorage(): localStorage per TOKEN e REFRESH_TOKEN
File: users-button.component.ts
goToRbac(): localStorage per TOKEN e REFRESH_TOKEN
📋 Motivazione: Mantenere la consistenza del sistema di storage in tutta l'applicazione.

🎯 RISULTATI TECNICI DETTAGLIATI
1. Architettura dei Token:
2. Flusso di Autenticazione Migliorato:
3. Error Handling Robusto:
4. Change Detection Stabilizzato:
🔒 CONSIDERAZIONI DI SICUREZZA
localStorage vs sessionStorage:
localStorage: Persiste fino a pulizia esplicita
Sicurezza: Stessa sicurezza (same-origin policy)
Logout: Pulizia esplicita implementata
XSS Protection: Nessun cambiamento (stesso livello di esposizione)
Token Refresh Automatico:
Trasparenza: L'utente non nota il refresh
Security Window: Breve finestra di token valido
Fallback: Se fallisce, comportamento originale
