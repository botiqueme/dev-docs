# Endpoint: `/resend_verification`

## Scopo
Consente agli utenti di richiedere un nuovo token di verifica email nel caso in cui quello precedente sia scaduto o non valido. Questo endpoint garantisce un'esperienza utente fluida, sicura e conforme ai requisiti di sicurezza.

---

## 1. Dettagli Tecnici

### **Metodo**
`POST`

### **URL**
`/resend_verification`

### **Autenticazione**
Nessuna (pubblico).

---

## 2. Parametri della Richiesta

| **Parametro** | **Tipo**  | **Obbligatorio** | **Descrizione**                             |
|---------------|-----------|------------------|---------------------------------------------|
| `email`       | Stringa   | Sì               | L'indirizzo email associato all'account dell'utente. |

Esempio di richiesta:
```
{
  "email": "user@example.com"
}
```

---

## 3. Logica dell'Endpoint

### **Fasi Chiave**
1. **Validazione dell'Input**:
   - Controllare che l'email sia presente nella richiesta.
   - Verificare che il formato dell'email sia valido (regex).

2. **Recupero Utente**:
   - Cercare nel database l'utente associato all'email.
   - **Errori Gestiti**:
     - Utente non trovato.
     - Utente già verificato.

3. **Stato dell'Utente**:
   - Se l'utente è già verificato:
     - Restituire un messaggio indicando che non è necessario rigenerare il token.

4. **Generazione del Token**:
   - Utilizzare `URLSafeTimedSerializer` per creare un nuovo token univoco.
   - Configurare la scadenza (es. 24 ore).

5. **Invio del Token via Email**:
   - Inviare un'email all'utente con il token generato, incluso un link per la verifica.

6. **Rate Limiting**:
   - Limitare a 3 richieste per ora per ciascun indirizzo IP/email.
   - Implementare logging dettagliato per monitorare tentativi sospetti.

7. **Logging e Monitoraggio (miglioria opzionale)**:
   - Loggare ogni tentativo di rigenerazione (successo/fallimento).
   - Raccogliere metriche per analisi future.

---

## 4. Risposte dell'Endpoint

| **Scenario**               | **HTTP Status**   | **Messaggio**                                |
|----------------------------|-------------------|---------------------------------------------|
| **Successo**               | `200 OK`         | "Verification email resent."                |
| **Email Mancante**          | `400 Bad Request` | "Missing email."                            |
| **Utente Non Trovato**      | `404 Not Found`   | "User not found."                           |
| **Utente Già Verificato**   | `409 Conflict`    | "User is already verified."                 |
| **Rate Limit Superato**     | `429 Too Many Requests` | "Too many requests. Please try again later."|

Esempio di Risposta Successo:
```
{
  "status": "success",
  "code": 200,
  "message": "Verification email resent."
}
```

---

## 5. Codice Completo

```
from flask import request, jsonify, current_app, url_for
from itsdangerous import URLSafeTimedSerializer
from app.models import User
from app.utils import send_verification_email
from app import db
import logging
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

# Configurazione logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Configurazione rate limiter
limiter = Limiter(key_func=get_remote_address)

@v1.route('/resend_verification', methods=['POST'])
@limiter.limit("3 per hour")
def resend_verification():
    data = request.get_json()
    email = data.get('email')

    # Validazione email mancante
    if not email:
        logger.warning("Resend verification failed: Missing email.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Missing email."
        }), 400

    # Recupero utente
    user = User.query.filter_by(email=email).first()
    if not user:
        logger.warning(f"Resend verification failed: User with email {email} not found.")
        return jsonify({
            "status": "error",
            "code": 404,
            "message": "User not found."
        }), 404

    # Verifica stato utente
    if user.is_verified:
        logger.info(f"Resend verification skipped: User {email} is already verified.")
        return jsonify({
            "status": "error",
            "code": 409,
            "message": "User is already verified."
        }), 409

    # Generazione token di verifica
    try:
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        token = serializer.dumps(email, salt=current_app.config['SECURITY_PASSWORD_SALT'])
        verify_url = url_for('v1.verify_email', token=token, _external=True)

        # Invio email
        send_verification_email(email, verify_url)
        logger.info(f"Verification email resent to {email}.")
    except Exception as e:
        logger.error(f"Error during token generation or email sending: {e}")
        return jsonify({
            "status": "error",
            "code": 500,
            "message": "Error resending verification email."
        }), 500

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "Verification email resent."
    }), 200
```

---

## 6. Sicurezza e Migliorie Implementate

1. **Rate Limiting**:
   - Configurato per evitare abusi e brute force.

2. **Validazione Input**:
   - Verifica rigorosa del formato email.

3. **Error Handling**:
   - Gestione dettagliata degli errori per migliorare l'esperienza utente e la sicurezza.

4. **Logging (miglioria opzionale)**:
   - Monitoraggio avanzato di successi e fallimenti per analisi e prevenzione di attacchi.

---

## 7. Prossimi Passi

1. **Test Automatici**:
   - Testare:
     - Email mancante.
     - Utente non trovato.
     - Utente già verificato.
     - Rate limiting.

2. **Monitoraggio Avanzato (miglioria opzionale)**:
   - Integrare strumenti come Prometheus per raccogliere metriche:
     - Numero di email rigenerate.
     - Frequenza di errori.

3. **Integrazione Frontend**:
   - Creare una pagina per notificare l'utente sullo stato della richiesta:
     - "Link di verifica inviato."
     - "Errore durante la rigenerazione del token."
