# Endpoint: `/update_password`

## 1. Dettagli
- **URL**: `/update_password`
- **Metodo**: `PUT`
- **Autenticazione**: Richiede un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di aggiornare la propria password in modo sicuro.

---

## 2. Comportamento dell'Endpoint

### 1. Parametri Accettati
Il corpo della richiesta deve essere in formato JSON e includere:
- `current_password` (stringa, obbligatorio): La password attuale dell’utente.
- `new_password` (stringa, obbligatorio): La nuova password scelta dall’utente.

Esempio di richiesta:
```
{
  "current_password": "mypassword123",
  "new_password": "newpassword456"
}
```

### 2. Validazioni
1. **Autenticazione**:
   - Verificare il token JWT per identificare l’utente.
2. **Presenza dei campi**:
   - Entrambi i campi `current_password` e `new_password` devono essere forniti.
3. **Confronto delle Password**:
   - La nuova password deve essere diversa da quella attuale.

### 3. Rate Limiting
- Configurare un limite di **3 tentativi per minuto per IP** utilizzando Flask-Limiter.
- Bloccare ulteriori richieste oltre il limite per prevenire abusi.

### 4. Aggiornamento della Password
- Verificare che la password attuale fornita corrisponda a quella hashata nel database.
- Generare l’hash della nuova password e aggiornare il database.

---

## 3. Risposte dell'Endpoint

### Successo
- **HTTP Status**: `200 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "message": "Password updated successfully."
}
```

### Errori

1. **Autenticazione Non Riuscita**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required."
}
```

2. **Parametri Mancanti**:
   - **HTTP Status**: `400 Bad Request`
   - **Body**:
```
{
  "status": "error",
  "code": 400,
  "message": "Both current and new passwords are required."
}
```

3. **Password Errata**:
   - **HTTP Status**: `400 Bad Request`
   - **Body**:
```
{
  "status": "error",
  "code": 400,
  "message": "Current password is incorrect."
}
```

4. **Rate Limiting Superato**:
   - **HTTP Status**: `429 Too Many Requests`
   - **Body**:
```
{
  "status": "error",
  "code": 429,
  "message": "Too many requests. Please try again later."
}
```

---

## 4. Codice Aggiornato

```
@v1.route('/update_password', methods=['PUT'])
@limiter.limit("3 per minute")  # Rate limiting specifico per l'endpoint
def update_password():
    # Autenticazione tramite token JWT
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

    # Aggiornamento della password
    user.password_hash = hash_password(new_password)
    db.session.commit()

    return jsonify_return_success("success", 200, {"message": "Password updated successfully"})
```

---

## 5. Considerazioni Importanti

### 1. **Configurabilità del Rate Limiting**
   - Limite predefinito: **3 tentativi per minuto per IP**.
   - Può essere personalizzato in base ai requisiti di sicurezza.

### 2. **Logging Dettagliato**
   - Loggare tutte le richieste per monitorare:
     - Tentativi di aggiornamento riusciti.
     - Tentativi di aggiornamento falliti.

### 3. **Protezione Avanzata (miglioria opzionale)**
   - Utilizzare un sistema di notifiche per avvisare l’utente di modifiche alla password non autorizzate.

---

## 6. Prossimi Passi

1. **Testare il comportamento del rate limiting**:
   - Simulare più tentativi di aggiornamento password in breve tempo.
2. **Integrare la protezione avanzata (miglioria opzionale)**:
   - Notifiche per aggiornamenti password sospetti.
3. **Aggiornare la documentazione API**:
   - Documentare il comportamento dell’endpoint e i limiti configurati.
