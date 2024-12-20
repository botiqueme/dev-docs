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
| Parametri mancanti         | `400 Bad Request` | "Missing fields: email, password"          |
| Email non valida           | `400 Bad Request` | "Invalid email format"                     |
| Email temporanea           | `400 Bad Request` | "Disposable emails are not allowed."       |
| Password non valida        | `400 Bad Request` | "Password does not meet security criteria."|
| Email duplicata            | `409 Conflict`    | "Email already registered."                |
| Rate limit superato        | `429 Too Many Requests` | "Too many requests. Please try again later." |

---

## Flusso Dettagliato

### **1. Ricezione della Richiesta**
1. Recuperare i parametri inviati dal frontend.
2. Controllare che tutti i campi obbligatori siano presenti.

### **2. Validazione**
- **Email**:
  - Formato valido (regex).
  - Non temporanea (libreria `is_disposable_email`).
- **Password**:
  - Lunghezza minima 8 caratteri.
  - Deve contenere almeno 1 maiuscola, 1 numero, 1 carattere speciale.
- **Campi opzionali**:
  - Numero di telefono (regex internazionale).
  - Partita IVA (regex nazionale).

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

    # Validazione parametri obbligatori
    required_fields = ['email', 'password', 'name', 'surname']
    missing_fields = [field for field in required_fields if field not in data]
    if missing_fields:
        return jsonify({
            "status": "error",
            "code": 400,
            "message": f"Missing fields: {', '.join(missing_fields)}"
        }), 400

    email = data['email']
    password = data['password']
    name = data['name']
    surname = data['surname']
    phone_number = data.get('phone_number')
    vat_number = data.get('vat_number')

    # Validazioni
    if not validate_email(email):
        return jsonify({"status": "error", "code": 400, "message": "Invalid email format"}), 400
    if is_disposable_email(email):
        return jsonify({"status": "error", "code": 400, "message": "Disposable emails are not allowed."}), 400
    if not validate_password(password):
        return jsonify({"status": "error", "code": 400, "message": "Password does not meet security criteria."}), 400

    # Hash password e generazione User ID
    password_hash = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())
    user_id = str(uuid.uuid4())

    # Creazione utente
    new_user = User(
        user_id=user_id,
        email=email,
        password_hash=password_hash,
        name=name,
        surname=surname,
        phone_number=phone_number,
        vat_number=vat_number,
        is_verified=False
    )
    try:
        db.session.add(new_user)
        db.session.commit()
    except IntegrityError:
        db.session.rollback()
        return jsonify({"status": "error", "code": 409, "message": "Email already registered."}), 409

    # Generazione token verifica
    serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
    verification_token = serializer.dumps(email, salt=current_app.config['SECURITY_PASSWORD_SALT'])

    # Invio email verifica
    verification_link = url_for('v1.verify_email', token=verification_token, _external=True)
    send_verification_email(email, verification_link)

    return jsonify({"status": "success", "code": 201, "message": "User registered. Please verify your email."}), 201
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
  - Strumenti consigliati:
    - **Task Scheduler**:
      - Celery con Beat per configurare un job periodico.
      - Alternativamente, un cron job che utilizza uno script per eliminare gli utenti.
    - Query SQL per selezionare ed eliminare gli utenti non verificati:
      ```sql
      DELETE FROM users WHERE is_verified = FALSE AND created_at < NOW() - INTERVAL '7 days';
      ```
  - Aggiungere log per monitorare le operazioni di pulizia.

---

### 2. Test Automatici
- **Problema**: La documentazione non specifica i test necessari per coprire i casi limite.
- **Soluzione**:
  - Scrivere test automatici per i seguenti scenari:
    - **Registrazione valida**: Utente creato con dati corretti.
    - **Email duplicata**: Tentativo di registrare un'email già esistente.
    - **Parametri mancanti**: Risposta appropriata quando uno o più parametri sono assenti.
    - **Email temporanea**: Verificare che l'endpoint rifiuti email usa e getta.
    - **Superamento del rate limit**: Simulare più richieste dallo stesso IP per assicurarsi che il rate limiting funzioni.
  - Strumenti consigliati:
    - **Librerie di test**: `pytest`, `unittest` o `Postman` per test manuali.
    - Integrare i test nel CI/CD (es. GitHub Actions).

---

### 3. Monitoraggio e Metriche
- **Problema**: Mancano indicazioni su come monitorare l'utilizzo dell'endpoint e individuare eventuali problemi.
- **Soluzione**:
  - Aggiungere metriche per:
    - **Numero di registrazioni**: Utenti registrati con successo ogni giorno.
    - **Numero di email inviate**: Per monitorare eventuali fallimenti.
    - **Tasso di errore**: Percentuale di richieste che falliscono.
  - Strumenti consigliati:
    - Prometheus + Grafana per raccogliere e visualizzare le metriche.
    - Libreria come `statsd` per inviare metriche dal backend.

---

### 4. Scalabilità per Invio Email
- **Problema**: In caso di alto volume di registrazioni, l'invio di email potrebbe diventare un collo di bottiglia.
- **Soluzione**:
  - Implementare una coda per gestire l'invio delle email:
    - **Strumenti consigliati**:
      - RabbitMQ o Redis con Celery per gestire la coda.
  - Configurare retry in caso di fallimento durante l'invio.
  - Separare il processo di registrazione dall'invio dell'email per migliorare i tempi di risposta.

---

### 5. Sicurezza Avanzata
- **Problema**: Mancano test e strategie specifiche contro attacchi avanzati.
- **Soluzione**:
  - **Test di Sicurezza**:
    - Simulare attacchi brute force per verificare l'efficacia del rate limiting.
    - Eseguire scansioni di vulnerabilità (es. OWASP ZAP).
  - **Blacklist Token di Verifica**:
    - Creare una blacklist per invalidare i token di verifica email in caso di abuso.

---

### 6. Preparazione per il Futuro
- Integrare l'endpoint con sistemi di identità esterni (es. OAuth con Google o Microsoft) per semplificare la registrazione.
- Aggiungere localizzazione per supportare diverse lingue nei messaggi di errore e nell'email di verifica.

---

## Prossimi Passi
- Implementare un **cron job** per eliminare utenti non verificati.
- Scrivere test automatici per tutti gli scenari menzionati.
- Configurare metriche con Prometheus per monitorare l'utilizzo dell'endpoint.
- Pianificare l'introduzione di una coda per l'invio delle email in vista della scalabilità.
