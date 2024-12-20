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
- Assicurarsi che l’utente abbia un avatar associato (miglioria opzionale).

### 3. Rimozione
- Resettare il campo `avatar` nel database (es. impostandolo a `NULL` o un valore di default).
- **Eliminazione del File Immagine (miglioria opzionale)**:
  - Se il file è memorizzato localmente, eliminarlo dalla directory di upload.
  - Se il file è memorizzato su uno storage cloud, utilizzare le API del provider per eliminarlo.

### 4. Risposta
- Restituire uno stato di successo con un messaggio che confermi l’eliminazione.

---

## Esempio di Risposte

### Richiesta
- Nessun parametro, solo autenticazione JWT.

### Risposta in caso di successo
```
{
  "status": "success",
  "code": 200,
  "message": "Avatar removed successfully"
}
```

### Risposta in caso di errore (utente senza avatar associato)
```
{
  "status": "error",
  "code": 404,
  "message": "No avatar found for this user"
}
```

---

## Esempio di Codice

```
@v1.route('/delete_avatar', methods=['DELETE'])
def delete_avatar():
    # Autenticazione tramite token JWT
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify_return_error("error", 401, "Authorization token required"), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        email = payload['email']
    except Exception:
        return jsonify_return_error("error", 401, "Invalid token"), 401

    # Recupero dell'utente
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify_return_error("error", 404, "User not found"), 404

    # Verifica che l'utente abbia un avatar (miglioria opzionale)
    if not user.avatar:
        return jsonify_return_error("error", 404, "No avatar found for this user"), 404

    # Eliminazione dell'avatar
    try:
        # Resetta il campo avatar nel database
        avatar_path = user.avatar  # Per eventuale eliminazione del file
        user.avatar = None
        db.session.commit()

        # (Miglioria opzionale) Rimuovi il file dal server o dallo storage cloud
        if avatar_path:
            if avatar_path.startswith("/path/to/upload"):  # Per storage locale
                os.remove(avatar_path)
            else:  # Per storage cloud
                # Integra le API del provider cloud per eliminare il file
                pass
    except Exception as e:
        logger.error(f"Error removing avatar: {e}")
        return jsonify_return_error("error", 500, "Error removing avatar"), 500

    return jsonify_return_success("success", 200, {"message": "Avatar removed successfully"})
```

---

## Considerazioni Importanti

### 1. **Validazioni Avanzate (miglioria opzionale)**
   - Verifica che l'utente abbia effettivamente un avatar associato per evitare richieste inutili.

### 2. **Eliminazione del File Immagine (miglioria opzionale)**
   - Configurare correttamente la gestione dello storage per garantire che i file inutilizzati vengano eliminati.

### 3. **Logging Dettagliato**
   - Loggare ogni richiesta per monitorare:
     - Utente coinvolto.
     - Esito (successo/fallimento).
     - Timestamp.

---

## Prossimi Passi

1. **Implementare l'endpoint** seguendo le specifiche.
2. **Testare** i seguenti scenari:
   - Rimozione di un avatar esistente.
   - Tentativo di rimozione senza avatar associato.
   - Gestione degli errori durante l'eliminazione del file (se applicabile).
3. **Aggiornare la documentazione API** per includere i dettagli di questo endpoint.
