Hai assolutamente ragione, l’User ID è fondamentale per un sistema sicuro e scalabile, e ora dobbiamo assicurarci che l’endpoint di update password sia progettato correttamente. Ecco una soluzione dettagliata e completa per l’implementazione dell’endpoint.

# Endpoint: `/update_password`

## 1. Dettagli
- **URL**: `/update_password`
- **Metodo**: `PUT`
- **Autenticazione**: Richiede un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di aggiornare la propria password in modo sicuro.

---

## 2. Comportamento

### **1. Autenticazione**
- Recuperare il token JWT dall’intestazione `Authorization`.
- Decodificare il token per ottenere l’`user_id`.
- Verificare che l’utente esista nel database.

### **2. Parametri Accettati**
Il corpo della richiesta deve essere in formato JSON e contenere i seguenti parametri:
| **Parametro**       | **Tipo**  | **Obbligatorio** | **Descrizione**                                  |
|----------------------|-----------|------------------|------------------------------------------------|
| `current_password`  | Stringa   | Sì               | La password attuale dell’utente.               |
| `new_password`      | Stringa   | Sì               | La nuova password da impostare.                |

Esempio di richiesta:
```
{
  "current_password": "old_password123",
  "new_password": "new_secure_password456"
}
```

---

### **3. Validazioni**
1. **Password Attuale**:
   - Verificare che la password fornita corrisponda a quella salvata nel database.
   - Restituire un errore se la password non è corretta.

2. **Nuova Password**:
   - Assicurarsi che la nuova password rispetti i criteri di sicurezza (es. lunghezza minima, caratteri speciali, ecc.).
   - Evitare che la nuova password sia uguale a quella attuale.

---

### **4. Aggiornamento**
- Aggiornare la password dell’utente nel database utilizzando un hash sicuro (`bcrypt`).
- Invalidate eventuali token JWT attivi se richiesto (opzionale, basato sulle politiche di sicurezza).

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

---

### Errori
1. **Utente Non Autenticato**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required."
}
```

2. **Password Attuale Non Corretta**:
   - **HTTP Status**: `400 Bad Request`
   - **Body**:
```
{
  "status": "error",
  "code": 400,
  "message": "Current password is incorrect."
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

## 4. Codice Aggiornato

```
@v1.route('/update_password', methods=['PUT'])
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

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "Password updated successfully."
    }), 200
```

---

## 5. Validazioni da Implementare

1. **Criteri di Sicurezza**:
   - Lunghezza minima di 8 caratteri.
   - Almeno una lettera maiuscola, un numero e un carattere speciale.

2. **Prevenzione Password Riutilizzate** (Opzionale):
   - Salvare un hash delle ultime 3 password e verificare che la nuova password non coincida con queste.

3. **Scadenza dei Token** (Opzionale):
   - Invalidate i token JWT attivi dopo un cambio di password.

---

## 6. Prossimi Passi

1. **Implementare l'endpoint** seguendo le specifiche.
2. **Testare**:
   - Cambio password con dati validi.
   - Tentativi con password attuale errata.
   - Nuova password che non rispetta i criteri.
3. **Aggiornare la documentazione API** per includere dettagli su questo endpoint.
