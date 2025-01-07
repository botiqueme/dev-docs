# Endpoint: /update_password

## 1. Dettagli
- **URL**: `/update_password`
- **Metodo**: `PATCH`
- **Autenticazione**: Richiede un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di aggiornare la propria password in modo sicuro.

## 2. Parametri Accettati

Il corpo della richiesta deve essere in formato JSON:
- **current_password** (stringa, obbligatorio): La password attuale dell’utente.
- **new_password** (stringa, obbligatorio): La nuova password da impostare.

Esempio di richiesta:
```
{
  "current_password": "OldPass123!",
  "new_password": "NewPass456!"
}
```

## 3. Comportamento

### Autenticazione
1. Recuperare il token JWT dall’intestazione `Authorization`.
2. Verificare e decodificare il token per identificare l’utente.

### Validazioni
1. **Presenza dei campi**:
   - Controllare che `current_password` e `new_password` siano presenti.
2. **Verifica password attuale**:
   - Assicurarsi che la password attuale corrisponda a quella memorizzata (hashing password).
3. **Nuova password valida**:
   - Controllare che la nuova password rispetti i criteri di sicurezza: 
     - Minimo 8 caratteri.(verifica fatta in frontend)
     - Almeno 1 maiuscola, 1 numero e 1 carattere speciale.(verifica fatta in frontend)
     - Diversa dalla password attuale.

### Aggiornamento
1. Generare un hash della nuova password con `bcrypt`.
2. Aggiornare la password hashata nel database.

### Risposta
- Confermare l’aggiornamento con un messaggio di successo.

## 4. Risposte

### Successo

| HTTP Status | Messaggio                     |
|-------------|-------------------------------|
| 200 OK      | "Password updated successfully." |

### Errori

| Causa                     | HTTP Status       | Messaggio                                   |
|---------------------------|-------------------|---------------------------------------------|
| Parametri mancanti         | 400 Bad Request  | "Both current and new passwords are required." |
| Password attuale errata    | 400 Bad Request  | "Current password is incorrect."           |
| Token non valido           | 401 Unauthorized | "Invalid token."                            |
| Utente non trovato         | 404 Not Found    | "User not found."                           |
| Internal Server Error      | 500 Error    | "Internal Server Error."                           |


## 5. Codice Aggiornato

```
@v1.route('/update_password', methods=['PATCH'])
@jwt_required()
def update_password():
    """
    Endpoint per aggiornare la password dell'utente autenticato.
    Richiede un token JWT valido.
    """
    try:
        # Ottieni l'ID dell'utente dal token JWT
        user_id = get_jwt_identity()

        # Recupera l'utente dal database
        user = User.query.get(user_id)
        if not user:
            return utils.jsonify_return_error("error", 404, "User not found"), 404

        # Recupera i dati dal corpo della richiesta
        data = request.get_json()
        if not data:
            return utils.jsonify_return_error("Bad Request", 400, "Both current and new passwords are required."), 400

        # Estrai le password dal payload
        current_password = data.get('current_password')
        new_password = data.get('new_password')

        # Verifica che tutte le password siano fornite
        if not all([current_password, new_password]):
            return utils.jsonify_return_error("Bad Request", 400, "Both current and new passwords are required."), 400
        
        # Verifica la password utilizzando bcrypt
        if not utils.verify_password(user.password_hash, current_password):  # Confronto con la password hashata
            return utils.jsonify_return_error("Bad Request", 40, "Current password is incorrect."), 400
        
        # Aggiorna la password dell'utente
        user.password_hash = utils.hash_password(new_password)

        # Salva le modifiche nel database
        db.session.commit()

        # Restituisci una risposta di successo
        return utils.jsonify_return_success("OK", 200, {"message": "Password updated successfully"}), 200

    except Exception as e:
        # Gestione degli errori imprevisti
        return utils.jsonify_return_error("error", 500, "Internal Server Error."), 500
```

## 6. Miglioramenti Futuri

- **(miglioria opzionale) Rotazione dei Token JWT**:
  - Dopo l’aggiornamento della password, invalidare i token JWT attuali e richiedere un nuovo login.
- **(miglioria opzionale) Notifica via Email**:
  - Inviare una notifica all’utente per informarlo che la password è stata modificata.
- **(miglioria opzionale) Logging Avanzato**:
  - Registrare i tentativi di aggiornamento della password per identificare attività sospette.

## 7. Prossimi Passi
1. Implementare l’endpoint seguendo le specifiche.
2. Testare i seguenti scenari:
   - Aggiornamento riuscito con dati validi.
   - Tentativi con password attuale errata.
   - Validazione di nuove password non conformi.
   - Superamento del rate limit.
