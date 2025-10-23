# PERSISTENZA LOGIN TRA TAB

### ANALISI INIZIALE DEL PROBLEMA

###### Problema Principale:

Quando un utente faceva login su Olimpo e poi copiava l'URL per aprirlo in un nuovo tab, veniva reindirizzato a /not_authorized invece di accedere alla pagina desiderata.

###### Analisi:

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
