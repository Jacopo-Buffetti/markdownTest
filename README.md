# PERSISTENZA LOGIN TRA TAB

### ANALISI INIZIALE DEL PROBLEMA

#### Problema Principale:

Quando un utente faceva login su Olimpo e poi copiava l'URL per aprirlo in un nuovo tab, veniva reindirizzato a /not_authorized invece di accedere alla pagina desiderata.

#### Analisi:

- __sessionStorage vs localStorage__: I token erano salvati in sessionStorage
- __Isolamento dei tab__: sessionStorage è isolato per ogni tab del browser
- __AuthGuard__: Verificava la presenza del token per autorizzare l'accesso
- __Flusso di errore__: Nuovo tab → Nessun token → AuthGuard nega accesso → Redirect a not_authorized

---

### MODIFICA 1: SISTEMA DI STORAGE DEI TOKEN

__File__: `authentication.service.ts`

__METODO__ `setStorage()`

#### Modifica Specifica:

```typescript
// PRIMA:
sessionStorage.setItem(SessionStorage.TOKEN, tokenData.token.access_token);
sessionStorage.setItem(SessionStorage.REFRESH_TOKEN, tokenData.token.refresh_token);

// DOPO:
localStorage.setItem(SessionStorage.TOKEN, tokenData.token.access_token);
localStorage.setItem(SessionStorage.REFRESH_TOKEN, tokenData.token.refresh_token);
```

#### Motivazione Tecnica:

- __sessionStorage__: Dati isolati per ogni tab/finestra del browser
- __localStorage__: Dati condivisi tra tutti i tab dello stesso dominio
- __Persistenza__: localStorage sopravvive alla chiusura dei tab (fino a logout esplicito)
- __Sicurezza__: Mantiene la sicurezza perché limitato al dominio

#### Risultato:

Ora i token sono accessibili da qualsiasi tab dello stesso dominio.

---

__METODO__ `pulisciStorage()`

#### Modifica Specifica:

```typescript
// PRIMA:
sessionStorage.removeItem(SessionStorage.TOKEN);
sessionStorage.removeItem(SessionStorage.REFRESH_TOKEN);

// DOPO:
localStorage.removeItem(SessionStorage.TOKEN);
localStorage.removeItem(SessionStorage.REFRESH_TOKEN);
```

#### Motivazione Tecnica:

- __Consistenza__: Rimuove i token dallo stesso storage dove li salva
- __Pulizia completa__: Assicura la rimozione completa dei dati di autenticazione
- __Sicurezza__: Previene token orfani in storage diversi

__METODO__ `forceLogout()`

#### Modifica Identica a pulisciStorage()

#### Motivazione:

Stesso principio di consistenza e pulizia completa.

---

__GETTER__ `get token()` e `get getRefreshToken()`

#### Modifica Specifica:

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

#### Motivazione Tecnica:

- __Coerenza__: Deve leggere dallo stesso storage dove scrive
- __Accessibilità__: Permette l'accesso ai token da qualsiasi tab
- __Uniformità API__: Mantiene la stessa interfaccia pubblica

---

__METODO__ `getUserLogged()` - __PRIMA VERSIONE__

#### Modifica Specifica:

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

#### Motivazione Tecnica:

- __Token Null__: Gestisce il caso in cui il token non esista
- __Gestione degli errori__: Cattura errori di decodifica JWT malformati
- __Coerenza di archiviazione__: Usa localStorage invece di sessionStorage

#### Benefici:

- Previene crash dell'applicazione
- Migliore debugging con log degli errori
- Comportamento predicibile

---

__METODO__ `getUserLabels()` - __MIGLIORAMENTO__

#### Modifica Specifica:

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

#### Motivazione Tecnica:

- __Valori predefiniti__: Ritorna array vuoto invece di undefined
- __Controlli null annidati__: Verifica token, userLogged, e labels
- __Tipo di ritorno coerente__: Sempre ritorna un array

---

### MODIFICA 2: MIGLIORAMENTO DELL'AUTHGUARD

__File__: `auth.guard.ts`

__AGGIUNTA IMPORT PER REACTIVE PROGRAMMING__

#### Modifica Specifica:

```typescript
// AGGIUNTO:
import { Observable, of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';
```

#### Motivazione Tecnica:

- __Operazioni asincrone__: Necessario per operazioni di refresh token
- __Integrazione RxJS__: Integrazione con l'architettura Angular/RxJS esistente
- __Gestione degli errori__: Gestione degli errori

---

__REFACTORING DEL METODO__ `canActivate()`

#### Modifica Specifica:

```typescript
// PRIMA:
canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot) {
    const token = this.authenticationService.token;
    let ret = false;
    if (token) {
        ret = true;
    } else {
        const tokenOtp = route.queryParams.t;
        if (tokenOtp) {
            this.router.navigate([routes.OTP], { queryParams: { t: tokenOtp } });
        } else {
            this.router.navigate([routes.NOT_AUTH]);
        }
    }
    return ret;
}

// DOPO:
canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean> | boolean {
    const token = this.authenticationService.token;
    const refreshToken = this.authenticationService.getRefreshToken;
    
    if (token) {
        return true;
    } else if (refreshToken) {
        // Tenta il refresh del token prima di reindirizzare
        return this.authenticationService.refreshToken().pipe(
            map((response) => {
                if (response && response.data) {
                    this.authenticationService.setStorage(response);
                    return true;
                } else {
                    this.handleNoAuth(route);
                    return false;
                }
            }),
            catchError(() => {
                this.handleNoAuth(route);
                return of(false);
            })
        );
    } else {
        this.handleNoAuth(route);
        return false;
    }
}
```

#### Motivazione Tecnica:

- __Recovery__: Tenta il refresh automatico invece di fallire immediatamente
- __Supporto Asincrono__: Ritorna Observable per operazioni asincrone
- __Fallback__: Solo se il refresh fallisce, reindirizza a not_authorized
- __Riutilizzo del Codice__: Estrae la logica di reindirizzamento in handleNoAuth()
- __Migliore UX__: L'utente potrebbe non accorgersi del refresh automatico

#### Flusso Logico Migliorato:

1. Token presente? → Accesso immediato
2. Token assente ma refresh presente? → Tenta refresh
3. Refresh riuscito? → Salva nuovo token e consenti accesso
4. Refresh fallito? → Reindirizza a not_authorized
5. Nessun token? → Reindirizza a not_authorized

---

__NUOVO METODO__ `handleNoAuth()`

#### Aggiunta:

```typescript
private handleNoAuth(route: ActivatedRouteSnapshot): void {
    const tokenOtp = route.queryParams.t;
    if (tokenOtp) {
        this.router.navigate([routes.OTP], { queryParams: { t: tokenOtp } });
    } else {
        this.router.navigate([routes.NOT_AUTH]);
    }
}
```

#### Motivazione Tecnica:

- __Principio DRY__: Evita duplicazione di codice
- __Responsabilità unica__: Metodo dedicato alla gestione dell'autorizzazione negata
- __Manutenibilità__: Più facile modificare la logica in futuro

---

### MODIFICA 3: AGGIORNAMENTO COMPONENTI ESISTENTI

__File__: `app.component.ts`

__METODO__ `ngOnInit()`

#### Modifica Specifica:

```typescript
// PRIMA:
ngOnInit(): void {
    const token = sessionStorage.getItem(SessionStorage.TOKEN);
    if (token) {
        this.store.dispatch(isLoggedActions({ isLogged: true }));
        this.store.dispatch(setAppReady({ isAppReady: true }));
    }	
}

// DOPO:
ngOnInit(): void {
    const token = localStorage.getItem(SessionStorage.TOKEN);
    if (token) {
        this.store.dispatch(isLoggedActions({ isLogged: true }));
        this.store.dispatch(setAppReady({ isAppReady: true }));
    }	
}
```

#### Motivazione: Consistenza con il nuovo sistema di storage.

---

#### GESTIONE WEBSOCKET SICURA

#### Modifica Specifica:

```typescript
// PRIMA:
this.isLogged$.subscribe((el) => {
    if (el === true) {
        this.userLogged = this.authService.getUserLogged();
        this.webSocketService.connect(this.userLogged.email);
    }
}

// DOPO:
this.isLogged$.subscribe((el) => {
    if (el === true) {
        this.userLogged = this.authService.getUserLogged();
        if (this.userLogged && this.userLogged.email) {
            this.webSocketService.connect(this.userLogged.email);
        }
    }
}
```

#### Motivazione Tecnica:

- __Sicurezza Null__: Previene errori se getUserLogged() ritorna null
- __Controllo della Proprietà__: Verifica l'esistenza dell'email prima di usarla
__WebSocket__: Evita connessioni WebSocket con dati non validi

---

__File__: `config.service.ts`

#### CARICAMENTO CONDIZIONALE DELLE PREFERENZE

#### Modifica Specifica:

```typescript
// PRIMA:
// load user prefs and set language
try {
    const langPreference = await this.authService.getUserPreferences();
    const language = langPreference.value;
    this.langService.setLanguage(language);
} catch (e) {
    console.error('Errore nel caricamento delle preferenze utente', e);
}

// DOPO:
// load user prefs and set language only if user is authenticated
const token = this.authService.token;
if (token) {
    try {
        const langPreference = await this.authService.getUserPreferences();
        const language = langPreference.value;
        this.langService.setLanguage(language);
    } catch (e) {
        console.error('Errore nel caricamento delle preferenze utente', e);
    }
}
```

#### Motivazione Tecnica:

- __Performance__: Evita chiamate API non necessarie
- __Riduzione degli Errori__: Previene errori quando l'utente non è autenticato
- __Flusso Logico__: Le preferenze utente hanno senso solo se c'è un utente

---

### MODIFICA 4: RISOLUZIONE ERRORI RUNTIME

__File__: `project-button.component.ts`

__METODO__ `getUserById()` - __BUG FIX__

#### Modifica Specifica:

```typescript
// PRIMA (CAUSAVA ERRORE):
getUserById(user_id: any) {
    if (this.users && this.users.length) {
        return this.users.find((el) => el.user_id === user_id).username;
        //                                                   ^
        //                              Potenziale undefined!
    } else return '';
}

// DOPO (SICURO):
getUserById(user_id: any) {
    if (this.users && this.users.length) {
        const user = this.users.find((el) => el.user_id === user_id);
        return user ? user.username : '';
    } else return '';
}
```

#### Analisi Tecnica del Bug:

- __Comportemneto Array.find()__: Ritorna undefined se nessun elemento è trovato
- __Accesso alla Proprietà su Undefined__: undefined.username genera TypeError
- __Template binding__: L'errore si propaga al template HTML

#### Soluzione Implementata:

- __Assegnazione sicura__: Assegna il risultato di find() a una variabile
- __Verifica Null__: Verifica se l'utente esiste prima di accedere alle proprietà
- __Valore di Default__: Ritorna stringa vuota se l'utente non è trovato

---

__ File__: `statistics.service.ts`
#### COSTRUTTORE CON WARNING

#### Modifica Specifica:

```typescript
// PRIMA:
constructor(
    private _http: HttpClient, 
    private _authService: AuthenticationService, 
) {
    this._userInfo = this._authService.getUserLogged();
}

// DOPO:
constructor(
    private _http: HttpClient, 
    private _authService: AuthenticationService, 
) {
    this._userInfo = this._authService.getUserLogged();
    if (!this._userInfo) {
        console.warn('Utente non autenticato nel StatisticsService');
    }
}
```
#### Motivazione:

- __Aiuto nel debug__: Aiuta a identificare problemi di inizializzazione
- __Allerta Rapida__: Notifica potenziali problemi prima che causino errori
- __Supporto allo sviluppo__: Utile durante lo sviluppo e testing

---

### MODIFICA 5: DIRETTIVE DI PERMESSI

__Problema Identificato__:
Le direttive `*appPermission` e `*appExactPermission` generavano errori:

- Cannot read properties of null (reading 'entity_perms')
- ExpressionChangedAfterItHasBeenCheckedError

__File__: `permission.directive.ts`

#### REFACTORING COMPLETO DELLA DIRETTIVA

#### Prima Versione (PROBLEMATICA):

```typescript
export class PermissionDirective {
    user: User = this.appHelpers.decodeJwt(sessionStorage.getItem(SessionStorage.TOKEN)!);
    //             ^                        ^                                           ^
    //             |                        |                                           |
    //     Inizializzazione                sessionStorage                    Non-null assertion
    //     nel costruttore                 (non condiviso)                    (pericoloso)

    ngAfterViewInit(): void {
        // Logica eseguita direttamente
        this.user.entity_perms.some((perms) => { /* ... */ });
        //        ^
        //        Potenziale null!
    }
}
```

#### Versione Finale:

export class PermissionDirective implements OnInit {
    user: User | null = null;
    private hasToken = false;
```typescript
    ngOnInit(): void {
        // Inizializzazione stabile e sicura
        const token = localStorage.getItem(SessionStorage.TOKEN);
        this.hasToken = !!token;
        
        if (this.hasToken) {
            try {
                this.user = this.appHelpers.decodeJwt(token!);
            } catch (error) {
                console.error('Errore nel decodificare il token:', error);
                this.user = null;
            }
        }
    }

    ngAfterViewInit(): void {
        // Posticipa l'esecuzione per evitare ExpressionChangedAfterItHasBeenCheckedError
        setTimeout(() => {
            this.checkPermissions();
        }, 0);
    }

    private checkPermissions(): void {
        // Controlli multipli per sicurezza
        if (!this.hasToken || !this.user || !this.user.entity_perms) {
            return; // Exit silenzioso
        }
        
        // Logica originale ma sicura
        /* ... */
    }
}
```

### Analisi Tecnica dei Miglioramenti:

1. __Stable Initialization:__

	- `ngOnInit()` è chiamato prima del rendering
	- I valori sono stabilizzati prima che il template li usi
	- Previene cambiamenti durante il change detection

2. __Gestione Errori:__

	- Try-catch per decodifica JWT
	- Gestione sicura di token corrotti o malformati
	- Log degli errori per debugging

3. __Sicurezza Null:__

	- Controlli multipli: token, user, entity_perms
	- Tipo nullable: User | null

4. __ExpressionChangedAfterItHasBeenCheckedError Prevention:__

	- `setTimeout(() => {}, 0)` posticipa l'esecuzione
	- Previene modifiche al DOM durante il controllo

5. __Coerenza Storage:__

	- Usa localStorage come tutto il resto del sistema
	- Accesso ai token condivisi tra i tab

6. __Ottimiazzazione dei flags:__

	- `hasToken` booleano per controlli rapidi
	- Evita chiamate ripetute a localStorage

--- 

__File__: `exact-permission.directive.ts`

#### STESSE MODIFICHE PER CONSISTENZA

#### Motivazione:

- Entrambe le direttive hanno la stessa architettura
- Stesso pattern di errori e soluzioni
- Consistenza nell'interfaccia e comportamento

---

### MODIFICA 6: ALTRI AGGIORNAMENTI DI CONSISTENZA

__File__: `utility.ts`

- `cleanStorage()`: localStorage per TOKEN e REFRESH_TOKEN

__File__: `users-button.component.ts`

- `goToRbac()`: localStorage per TOKEN e REFRESH_TOKEN

#### Motivazione: Mantenere la consistenza del sistema di storage in tutta l'applicazione.

---
