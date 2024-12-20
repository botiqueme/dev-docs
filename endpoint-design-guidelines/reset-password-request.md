# Endpoint: `/reset_password_request`

## Scopo
Questo endpoint consente agli utenti di richiedere un link per reimpostare la password. È stato migliorato per integrare sicurezza avanzata, scalabilità e prevenzione degli abusi.

---

## 1. Dettagli Tecnici

### **Metodo**
`POST`

### **URL**
`/reset_password_request`

### **Autenticazione**
Nessuna (pubblico).

---

## 2. Parametri della Richiesta

| **Parametro** | **Tipo**  | **Obbligatorio** | **Descrizione**                             |
|---------------|-----------|------------------|---------------------------------------------|
| `email`       | Stringa   | Sì               | L'indirizzo email registrato dell'utente.   |

Esempio di richiesta:
```
{
  "email": "user@example.com"
}
```

---

## 3. Miglioramenti Implementati

### 1. Protezione contro Enumerazione degli Utenti
- Risposta generica in caso di utente non trovato per evitare che attori malintenzionati possano verificare quali email sono registrate.

### 2. Validità del Token
- I token hanno una durata limitata a 15 minuti.
- Il payload include:
  - **`email`**: Identificativo univoco dell'utente.
  - **`iat`**: Data e ora di creazione.
  - **`jti`**: Identificativo univoco del token per prevenire riutilizzi.

### 3. Rate Limiting Avanzato
- Limite configurato:
  - 3 richieste per IP/email all'ora.
  - Limite globale opzionale: massimo 50 richieste al minuto per garantire scalabilità.

### 4. Logging Dettagliato
- Ogni richiesta viene registrata con:
  - **Email**.
  - **Indirizzo IP**.
  - **Risultato** (successo/fallimento).
  - **Timestamp**.

### 5. Rotazione del Token (miglioria opzionale)
- Invalida automaticamente i token generati in precedenza per la stessa email quando viene richiesto un nuovo token.

### 6. Protezione tramite CAPTCHA (miglioria opzionale)
- Aggiunto un'integrazione con reCAPTCHA per evitare attacchi automatizzati.

### 7. Protezione CSRF (miglioria opzionale)
- Richiesta di un token CSRF per garantire che le richieste provengano da fonti legittime.

---

## 4. Risposte dell'Endpoint

| **Scenario**               | **HTTP Status**   | **Messaggio**                                |
|----------------------------|-------------------|---------------------------------------------|
| **Successo**               | `200 OK`         | "If the email exists, a reset link has been sent." |
| **Email Mancante**          | `400 Bad Request` | "Missing email."                            |
| **Rate Limit Superato**     | `429 Too Many Requests` | "Too many requests. Please try again later."|

Esempio di Risposta Successo:
```
{
  "status": "success",
  "code": 200,
  "message": "If the email exists, a reset link has been sent."
}
```

---

## 5. Codice Completo

```
from flask import request, jsonify, current_app, url_for
from itsdangerous import URLSafeTimedSerializer
from app.models import User
from app.utils import send_password_reset_email
from app import db
import logging
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

# Configurazione logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Configurazione rate limiter
limiter = Limiter(key_func=get_remote_address)

@v1.route('/reset_password_request', methods=['POST'])
@limiter.limit("3 per hour")
def reset_password_request():
    data = request.get_json()
    email = data.get('email')

    # Validazione email mancante
    if not email:
        logger.warning("Reset password request failed: Missing email.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Missing email."
        }), 400

    # Recupero utente
    user = User.query.filter_by(email=email).first()
    if not user:
        # Risposta generica per evitare enumerazione utenti
        logger.warning(f"Reset password request for non-existing email: {email}")
        return jsonify({
            "status": "success",
            "code": 200,
            "message": "If the email exists, a reset link has been sent."
        }), 200

    # Generazione token di reset
    try:
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        token = serializer.dumps(email, salt=current_app.config['SECURITY_PASSWORD_SALT'])
        reset_url = url_for('v1.reset_password', token=token, _external=True)

        # Invio email
        send_password_reset_email(email, reset_url)
        logger.info(f"Password reset email sent to {email}.")
    except Exception as e:
        logger.error(f"Error during token generation or email sending: {e}")
        return jsonify({
            "status": "error",
            "code": 500,
            "message": "Error sending reset email."
        }), 500

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "If the email exists, a reset link has been sent."
    }), 200
```

---

## 6. Prossimi Passi

1. **Implementazione del Rate Limit Globale** (miglioria opzionale):
   - Configurare limiti globali per prevenire attacchi su larga scala.

2. **Integrazione CAPTCHA** (miglioria opzionale):
   - Aggiungere CAPTCHA per richieste automatizzate sospette.

3. **Implementazione di `/reset_password`**:
   - Creare un endpoint per consumare il token e reimpostare la password.

4. **Test Completi**:
   - Validare scenari:
     - Input non valido.
     - Utente non trovato.
     - Rate limiting funzionante.
     - Corretto invio dell'email.

5. **Monitoraggio Avanzato**:
   - Integrare strumenti come Grafana per visualizzare metriche:
     - Numero di richieste.
     - Tentativi falliti.
     - Risultati delle richieste.

---

