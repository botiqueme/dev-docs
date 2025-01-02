# Endpoint: `/register`

## Scopo
Registrare un nuovo utente, assicurandosi che i dati siano validi, sicuri e pronti per l'uso in una piattaforma che richiede verifica dell'email. Questo endpoint è progettato per essere sicuro, scalabile e integrabile senza ambiguità.

---

## Specifiche Tecniche

### **Metodo**
`POST`

### **URL**
`/register`

### **Autenticazione**
Nessuna (pubblico).

---

## Parametri

### **Richiesti**
| **Nome**      | **Tipo**  | **Descrizione**                   | **Validazione**                 |
|---------------|-----------|-----------------------------------|---------------------------------|
| `email`       | Stringa   | Email dell'utente.               | Formato valido, non temporanea.|
| `password`    | Stringa   | Password da hashare.             | Minimo 8 caratteri, almeno 1 maiuscola, 1 numero, 1 speciale. |
| `name`        | Stringa   | Nome dell'utente.                | Non vuoto.                     |
| `surname`     | Stringa   | Cognome dell'utente.             | Non vuoto.                     |

### **Opzionali**
| **Nome**       | **Tipo**  | **Descrizione**                   | **Validazione**                 |
|-----------------|-----------|-----------------------------------|---------------------------------|
| `phone_number` | Stringa   | Numero di telefono.              | Regex per formati internazionali.|
| `vat_number`   | Stringa   | Partita IVA.                     | Regex per formati nazionali.   |

### Esempio di Richiesta
```
POST /register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "name": "John",
  "surname": "Doe",
  "phone_number": "+1234567890",
  "vat_number": "IT12345678901"
}
```

---

## Risposte

### **Successo**
| **HTTP Status** | **Messaggio**                          |
|-----------------|---------------------------------------|
| `201 Created`   | "User registered. Please verify your email." |

### **Errori**
| **Causa**                  | **HTTP Status** | **Messaggio**                                |
|----------------------------|-----------------|---------------------------------------------|
| Email temporanea           | `400 Bad Request` | "Disposable emails are not allowed."       |
| Email duplicata            | `409 Conflict`    | "Email already registered."                |
| Rate limit superato        | `429 Too Many Requests` | "Too many requests. Please try again later." |
| Integrita' del database    | `500 Error` | "Internal Server Error, please contact the admin" |

---

## Flusso Dettagliato

### **1. Ricezione della Richiesta**
1. Recuperare i parametri inviati dal frontend.
2. Controllare che tutti i campi obbligatori siano presenti (gestito dal frontend).

### **2. Validazione**
- **Email**:
  - Formato valido (gestito dal frontend).
  - Non temporanea (libreria `is_disposable_email`).
- **Password**:
  - Lunghezza minima 8 caratteri (gestito dal frontend)
  - Deve contenere almeno 1 maiuscola, 1 numero, 1 carattere speciale. (gestito dal frontend)
- **Campi opzionali**:
  - Numero di telefono (regex internazionale - gestito dal frontend).
  - Partita IVA (regex nazionale - gestito dal frontend).

### **3. Creazione dell'Utente**
1. Hash della password (`bcrypt`).
2. Generazione di un ID univoco per l'utente (`UUID`).
3. Salvataggio nel database:
   - Verificare che l'email non sia duplicata.

### **4. Invio dell'Email di Verifica**
1. Creazione di un token di verifica con `URLSafeTimedSerializer`.
2. Invio del token tramite un'email contenente il link per verificare l'account.

### **5. Risposte**
- Successo: Confermare che l'utente è stato registrato e richiedere la verifica email.
- Errore: Restituire un messaggio appropriato per ogni problema riscontrato.

---

## Codice Implementazione

```
@v1.route('/register', methods=['POST'])
@limiter.limit("3 per minute")
def register():
    data = request.get_json()
    email = data['email']
    password = data['password']
    name = data['name']
    surname = data['surname']
    phone_number = data.get['phone_number']
    vat_number = data.get['vat_number']

    # Verifica se l'email è un'email usa e getta
    disposable = is_disposable_email.check(email)
    if disposable:
        return jsonify_return_error("Error", 400, "Disposable emails are not allowed."), 400
    

    # Hash della password
    password_hash = utils.hash_password(password)


    # Creazione nuovo utente
    new_user = User(
        email=email,
        password_hash=password_hash,
        name=name,
        surname=surname,
        is_verified=False
    )

    # Aggiunta dei campi opzionali solo se non sono vuoti
    if phone_number:
        user_data['phone_number'] = phone_number
    if vat_number:
        user_data['vat_number'] = vat_number
    

    # Verifica se l'email è già registrata
    existing_user = User.query.filter_by(email=email).first()
    if existing_user:
        return jsonify_return_error("Conflict", 409, "Email already registered"), 409

    # Aggiungi l'utente al database
    try:
        db.session.add(new_user)
        db.session.commit()
    except IntegrityError as e:
        db.session.rollback()
        return jsonify_return_error("Error", 500, "Internal (Integrity) Server Error, please contact the admin"), 500
    except Exception as e:
        db.session.rollback()
        return jsonify_return_error("Error", 500, "Internal (Databse) Server Error, please contact the admin"), 500

    try:
        # Generazione token di verifica
        token = utils.generate_verification_token(email)
        verify_url = url_for('v1.verify_email', token=token, email=email, _external=True)

        response = send_verification_email(email, verify_url)
        if response.status_code == 404:
            callback_refresh()
            response = send_verification_email(email, verify_url)
    except Exception as e:
        return jsonify_return_error("Error", 500, "Internal (Verification Email) Server Error, please contact the admin"), 500


    if response.status_code == 200:
        return jsonify_return_success("success", 201, {"message": "User registered. Please verify your email."}), 201
    else:
        return jsonify_return_error("Error", 500, "Internal (Generic) Server Error, please contact the admin"), 500

@v1.route('/verify/<token>', methods=['GET'])
def verify_email(token):
    email = request.args.get('email')
    print(email)

    try:
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        email = serializer.loads(token, salt=current_app.config['SECURITY_PASSWORD_SALT'], max_age=86400)
    except SignatureExpired:
        # Il token è scaduto
        # Restituisci un JSON o reindirizza in base al tipo di richiesta
        if request.accept_mimetypes['application/json']:
            return jsonify_return_error("error", 400, "Expired token."), 400
        else:
            return redirect(url_for('v1.expired_token', email=email))
    except BadSignature:
        # Il token non è valido
        if request.accept_mimetypes['application/json']:
            return jsonify_return_error("error", 401, "Invalid token."), 401
        else:
            return redirect(url_for('v1.invalid_token', email=email))

    # Verifica utente nel database
    user = User.query.filter_by(email=email).first()
    if user is None:
        return redirect(url_for('v1.user_not_found'))
    if user.is_verified:
        # Restituisci un JSON o reindirizza in base al tipo di richiesta
        if request.accept_mimetypes['application/json']:
            return jsonify_return_success("success", 200, {"message": "Email confirmed successfully."}), 200
        else:
            return redirect(url_for('v1.login_page'))

    user.is_verified = True
    db.session.commit()

    # Restituisci un JSON o reindirizza in base al tipo di richiesta
    if request.accept_mimetypes['application/json']:
        return jsonify_return_success("success", 200, {"message": "Email confirmed successfully."}), 200
    else:
        return redirect(url_for('v1.login_page'))

@v1.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    email = data['email']
    password = data['password']

    # Cerca l'utente nel database
    user = User.query.filter_by(email=email).first()
    if user is None:
        return jsonify_return_error("error", 401, "Incorrect email or password."), 401

    if not user.is_verified:
        return jsonify_return_error("error", 402, "Email not verified, yet"), 402

    # Verifica la password utilizzando bcrypt
    if not verify_password(user.password_hash, password):  # Confronto con la password hashata
        return jsonify_return_error("error", 401, "Incorrect email or password."), 401

    jwt_token = create_jwt_token(user)

    # Se tutto va bene, logica per il login (es. creazione di un token JWT o gestione della sessione)
    # Ritorna il token JWT insieme a email e username
    data = {
        "message": "Successfull login",
        "jwt_token": jwt_token,
        "email": user.email
    }

    response = jsonify_return_success("success", 200, data)
    return response, 200


@v1.route('/resend_verification', methods=['POST'])
@limiter.limit("3 per hour")  # Limita a 3 richieste all'ora per utente
def resend_verification():
    """
    Resends a verification email to a user.

    Returns:
        Response: A response with a message and a status code.
    """
    email = request.form.get('email')
    print(email)

    # Verifica che l'email esista nel database
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify({"message": "User not found"}), 404
    if user.is_verified:
        return redirect(url_for('v1.login_page'))

    try:
        # Generazione token di verifica
        token = generate_verification_token(email)
        verify_url = url_for('v1.verify_email', token=token, email=email, _external=True)
    except Exception as e:
        return jsonify({"error in verif token": str(e)}), 500

    # Invia email di verifica
    try:
        response = send_token_verification_email(email, verify_url)
        if response.status_code == 404:
            callback_refresh()
            response = send_token_verification_email(email, verify_url)

    except Exception as e:
        return jsonify({"error in email": str(e)}), 500

    return jsonify({"message": "A new verification email has been sent."}), 200
```

---

## Dettagli di Sicurezza
1. **Rate Limiting**:
   - 3 richieste al minuto per IP con `Flask-Limiter`.
2. **Password Sicura**:
   - Hashata con `bcrypt`.
3. **Email Temporanee Bloccate**:
   - Validazione tramite libreria specifica.
4. **Token Sicuro per Verifica Email**:
   - Utilizzo di `URLSafeTimedSerializer`.

---

## Considerazioni Future e Migliorie Suggerite

### 1. Gestione degli Utenti Non Verificati
- **Problema**: Gli utenti che non verificano l'email possono rimanere nel database indefinitamente.
- **Soluzione**:
  - Implementare un sistema per eliminare automaticamente gli utenti non verificati dopo un periodo (es. 7 giorni).

### 2. Test Automatici
- Scrivere test automatici per i seguenti scenari:
  - **Registrazione valida**
  - **Email duplicata**
  - **Parametri mancanti**

### 3. Scalabilità per Invio Email
- Implementare una coda per gestire l'invio delle email.

### 4. Monitoraggio e Metriche
- Aggiungere metriche per monitorare il comportamento dell'endpoint.
