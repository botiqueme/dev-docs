# Endpoint: Rimozione dell'Avatar

## Dettagli
- **Endpoint**: `/delete_avatar`
- **Metodo**: `DELETE`
- **Autenticazione**: Richiedere un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di rimuovere il proprio avatar.

---

## Comportamento

### 1. Autenticazione
- Recuperare il token JWT dall’intestazione `Authorization`.
- Verificare e decodificare il token per identificare l’utente.

### 2. Validazioni
- Verificare che l’utente esista nel database.
- Assicurarsi che l’utente abbia un avatar associato (se necessario).

### 3. Rimozione
- Resettare il campo `avatar` nel database (es. impostandolo a `NULL` o un valore di default).
- (Opzionale) Eliminare il file immagine dallo storage (locale o cloud).

### 4. Risposta
- Restituire uno stato di successo con un messaggio che confermi l’eliminazione.

---

## Esempio di Risposte

### Richiesta
- Nessun parametro, solo autenticazione JWT.

### Risposta in caso di successo
```json
{
  "status": "success",
  "code": 200,
  "message": "Avatar removed successfully"
}
