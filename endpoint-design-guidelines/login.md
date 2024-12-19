# Endpoint: Login Normale

## 1. Dettagli
- **Endpoint**: `/login`
- **Metodo**: `POST`
- **Autenticazione**: Nessuna (è l'endpoint di autenticazione iniziale).
- **Scopo**: Consentire agli utenti registrati di accedere alla piattaforma utilizzando email e password.

---

## 2. Parametri Accettati
Il body della richiesta deve essere in formato JSON:
- `email` (stringa, obbligatorio): L'email registrata dell'utente.
- `password` (stringa, obbligatorio): La password dell'utente.

Esempio di richiesta:
```
{
  "email": "user@example.com",
  "password": "mypassword"
}
```

---

## 3. Validazioni
1. **Email**:
   - Verificare che sia presente e in un formato valido (es. `user@example.com`).
2. **Password**:
   - Assicurarsi che sia presente.
3. **Rate Limiting**:
   - Limitare il numero di tentativi consecutivi per prevenire attacchi brute force.

Esempio con Flask-Limiter:
```
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@v1.route('/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    # Logica del login
    pass
```

---

## 4. Logica dell'Endpoint
1. **Recupero Utente**:
   - Cercare nel database l'utente corrispondente all'email fornita.
   - Restituire errore se l'utente non esiste.

2. **Verifica dello Stato dell'Utente**:
   - Controllare se l'utente ha verificato l'email (`is_verified`).
   - Restituire errore se l'email non è verificata.

3. **Verifica della Password**:
   - Utilizzare `bcrypt` per confrontare la password fornita con quella hashata nel database.
   - Restituire errore se la password non corrisponde.

4. **Generazione del Token JWT**:
   - Creare un token JWT con le seguenti informazioni:
     - `user_id`: L'ID univoco dell'utente.
     - `email`: L'email dell'utente.
     - `exp`: Scadenza del token (es. 1 ora).
   - Firmare il token con la chiave segreta del backend.

5. **Risposta al Frontend**:
   - Restituire il token JWT e un messaggio di successo.

---

## 5. Sicurezza
1. **HTTPS**:
   - Obbligatorio per tutte le richieste per proteggere credenziali e token JWT.

2. **Sicurezza del Token JWT**:
   - Utilizzare una chiave segreta robusta (`SECRET_KEY`).
   - Configurare una scadenza adeguata (es. 1 ora).
   - Opzionale: Aggiungere `iat` (issued at) e `jti` (unique identifier) al payload del JWT per maggiore sicurezza.

3. **Blacklist dei Token**:
   - Implementare una blacklist per invalidare i token in caso di logout o compromissione.

4. **Protezione contro Brute Force**:
   - Implementare rate limiting sull'endpoint `/login`.

---

## 6. Risposte dell'Endpoint
### Successo
- **HTTP Status**: `200 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "message": "Login successful",
    "jwt_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "email": "user@example.com"
  }
}
```

### Errori
1. **Utente non trovato**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Incorrect email or password."
}
```

2. **Email non verificata**:
   - **HTTP Status**: `403 Forbidden`
   - **Body**:
```
{
  "status": "error",
  "code": 403,
  "message": "Email not verified."
}
```

3. **Rate Limiting**:
   - **HTTP Status**: `429 Too Many Requests`
   - **Body**:
```
{
  "status": "error",
  "code": 429,
  "message": "Too many login attempts. Please try again later."
}
```

---

## 7. Codice Esempio
```
@v1.route('/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    data = request.get_json()

    # Validazione dei parametri
    if not data or 'email' not in data or 'password' not in data:
        return jsonify_return_error("error", 400, "Missing email or password"), 400

    email = data['email']
    password = data['password']

    # Recupero utente
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify_return_error("error", 401, "Incorrect email or password"), 401

    # Verifica stato utente
    if not user.is_verified:
        return jsonify_return_error("error", 403, "Email not verified"), 403

    # Verifica password
    if not verify_password(user.password_hash, password):
        return jsonify_return_error("error", 401, "Incorrect email or password"), 401

    # Generazione token JWT
    jwt_token = jwt.encode(
        {
            "user_id": user.id,
            "email": user.email,
            "exp": datetime.utcnow() + timedelta(hours=1)
        },
        current_app.config['SECRET_KEY'],
        algorithm='HS256'
    )

    # Risposta
    return jsonify_return_success("success", 200, {
        "message": "Login successful",
        "jwt_token": jwt_token,
        "email": user.email
    })
```

---

## 8. Prossimi Passi
1. **Implementare l'endpoint** seguendo le specifiche sopra.
2. **Testare i seguenti scenari**:
   - Login corretto con credenziali valide.
   - Login con credenziali errate.
   - Login con un account non verificato.
   - Eccesso di tentativi di login.
3. **Configurare HTTPS** e testare l'accesso solo tramite connessioni sicure.
4. **Integrare il frontend** per inviare credenziali all'endpoint `/login`.

Se hai bisogno di dettagli aggiuntivi o vuoi espandere questa implementazione, fammi sapere! 😊
