# Endpoint: `/refresh_token`

## 1. Dettagli
- **URL**: `/refresh_token`
- **Metodo**: `POST`
- **Autenticazione**: Necessario il refresh token.
- **Scopo**: Fornire un nuovo access token quando quello precedente è scaduto.

---

## 2. Comportamento

### **1. Ricezione del Refresh Token**
- Recuperare il refresh token dall’intestazione `Authorization` o da un cookie HTTP-only.

### **2. Validazione del Refresh Token**
- Decodificare il refresh token utilizzando la chiave segreta del server.
- Verificare:
  - Validità del token (non scaduto).
  - Tipo di token (`refresh`).
  - Utente associato esistente.
- Restituire errore se il token non è valido.

### **3. Generazione di un Nuovo Access Token**
- Creare un nuovo access token con scadenza breve.

### **4. Risposta al Frontend**
- Restituire il nuovo access token.

---

## 3. Parametri Accettati
- Nessun parametro nel corpo della richiesta.
- Richiede il refresh token nell’intestazione `Authorization`.

Esempio di intestazione:
```
Authorization: Bearer <refresh_token>
```

---

## 4. Risposte dell'Endpoint

### Successo
- **HTTP Status**: `200 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "access_token": "nuovo_access_token"
  }
}
```

---

### Errori

1. **Refresh Token Mancante**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Refresh token missing."
}
```

2. **Refresh Token Non Valido**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Invalid refresh token."
}
```

3. **Utente Non Trovato**:
   - **HTTP Status**: `404 Not Found`
   - **Body**:
```
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```
4. **Refresh Token Expired**:
   - **HTTP Status**: `401 Error`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Refresh token has expired."
}

---

## 5. Codice Aggiornato

```
@v1.route('/refresh_token', methods=['POST'])
def refresh_token():
    """
    Endpoint per rigenerare un access token usando un refresh token.
    """
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return utils.jsonify_return_error("error", 401, "Refresh token is missing"), 401

    try:
        refresh_token = auth_header.split('Bearer')[1].strip()

        # Decodifica e verifica il refresh token
        payload = jwt.decode(refresh_token, current_app.config['SECRET_KEY'], algorithms=["HS256"])
        if payload.get('type') != 'refresh':
            return utils.jsonify_return_error("error", 401, "Invalid refresh token."), 401

        # Verifica se l'utente esiste
        user_id = payload.get('user_id')
        user = User.query.get(user_id)
        if not user:
            return utils.jsonify_return_error("error", 404, "User not found"), 404
        # Genera un nuovo access token
        new_access_token = utils.create_jwt_token(user, token_type="access")

        # Risposta al Frontend
        data = {
            "message": "Token refreshed successfully",
            "access_token": new_access_token
        }

        return utils.jsonify_return_success("success", 200, data), 200

    except jwt.ExpiredSignatureError:
        return utils.jsonify_return_error("error", 401, "Refresh token has expired"), 401
    except jwt.InvalidTokenError:
        return utils.jsonify_return_error("error", 401, "Invalid refresh token"), 401

```

---

## 6. Validazioni da Implementare

1. **Token JWT**:
   - Verificare che il token sia valido e non scaduto.
   - Controllare che il tipo sia `refresh`.

2. **Blacklist**:
   - Implementare una blacklist per invalidare i refresh token in caso di logout.

3. **Rotazione del Refresh Token (miglioria opzionale)**:
   - Generare un nuovo refresh token quando quello attuale viene utilizzato.

4. **Sicurezza Avanzata (miglioria opzionale)**:
   - Includere `jti` (unique identifier) per tracciare i token.

---

## 7. Prossimi Passi

1. **Implementare l'endpoint** seguendo le specifiche.
2. **Testare**:
   - Richieste valide con un refresh token funzionante.
   - Richieste con token scaduto o non valido.
   - Comportamento in caso di utente non trovato.
3. **Integrare la blacklist per il refresh token**.
4. **Aggiornare la documentazione API** per includere dettagli su questo endpoint.
5. **Integrare la rotazione del token (opzionale)** per aumentare la sicurezza.

---

