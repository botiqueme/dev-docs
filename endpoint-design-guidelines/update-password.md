# Endpoint: /update_password

## 1. Dettagli
- **URL**: `/update_password`
- **Metodo**: `PUT`
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
| Password non valida        | 400 Bad Request  | "New password does not meet security criteria." |
| Token non valido           | 401 Unauthorized | "Invalid token."                            |
| Utente non trovato         | 404 Not Found    | "User not found."                           |

## 5. Codice Aggiornato

```
@v1.route('/update_password', methods=['PUT'])
@limiter.limit("3 per minute")  # Rate limiting per prevenire abusi
def update_password():
    # Recupero del token JWT
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify_return_error("error", 401, "Authorization token required."), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        user_id = payload['user_id']
    except Exception:
        return jsonify_return_error("error", 401, "Invalid token."), 401

    # Recupero dell'utente
    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify_return_error("error", 404, "User not found."), 404

    # Recupero dati dal body della richiesta
    data = request.get_json()
    current_password = data.get('current_password')
    new_password = data.get('new_password')

    if not current_password or not new_password:
        return jsonify_return_error("error", 400, "Both current and new passwords are required."), 400

    # Verifica password attuale
    if not verify_password(user.password_hash, current_password):
        return jsonify_return_error("error", 400, "Current password is incorrect."), 400

    # Validazione della nuova password
    if current_password == new_password:
        return jsonify_return_error("error", 400, "New password must be different from the current password."), 400
    if not validate_password(new_password):
        return jsonify_return_error("error", 400, "New password does not meet security criteria."), 400

    # Aggiornamento della password
    user.password_hash = hash_password(new_password)
    db.session.commit()

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "Password updated successfully."
    }), 200
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
