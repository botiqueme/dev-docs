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

---

## 5. Codice Aggiornato

```
@v1.route('/refresh_token', methods=['POST'])
def refresh_token():
    # Recupero del refresh token dall'intestazione
    refresh_token = request.headers.get('Authorization')
    if not refresh_token:
        return jsonify({"error": "Refresh token missing"}), 401

    try:
        # Decodifica del token
        token = refresh_token.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        if payload['type'] != 'refresh':
            return jsonify({"error": "Invalid token type"}), 401

        # Recupero utente tramite User ID
        user_id = payload['user_id']
        user = User.query.filter_by(user_id=user_id).first()
        if not user:
            return jsonify({"error": "User not found"}), 404

        # Generazione di un nuovo access token
        new_access_token = jwt.encode({
            "user_id": user.user_id,
            "email": user.email,
            "exp": datetime.utcnow() + timedelta(minutes=15),
            "type": "access"
        }, current_app.config['SECRET_KEY'], algorithm='HS256')

        return jsonify({
            "status": "success",
            "code": 200,
            "data": {
                "access_token": new_access_token
            }
        }), 200

    except jwt.ExpiredSignatureError:
        return jsonify({"error": "Refresh token expired"}), 401
    except Exception:
        return jsonify({"error": "Invalid token"}), 401
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

