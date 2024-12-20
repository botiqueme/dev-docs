# Endpoint: `/reset_password`

## Scopo
Consente agli utenti di reimpostare la propria password in modo sicuro utilizzando un token di reset generato in precedenza. La nuova versione implementa miglioramenti significativi in termini di sicurezza e scalabilità.

---

## 1. Dettagli Tecnici

### **Metodo**
`POST`

### **URL**
`/reset_password`

### **Autenticazione**
Nessuna (pubblico).

---

## 2. Parametri della Richiesta

| **Parametro**  | **Tipo**   | **Obbligatorio** | **Descrizione**                                      |
|----------------|------------|------------------|------------------------------------------------------|
| `token`        | Stringa    | Sì               | Token di reset ricevuto via email.                  |
| `new_password` | Stringa    | Sì               | Nuova password che l'utente desidera impostare.     |

Esempio di richiesta:
```
{
  "token": "example_reset_token",
  "new_password": "NewPassword123!"
}
```

---

## 3. Flusso Logico dell'Endpoint

### **1. Validazione dell'Input**
- **Token**:
  - Verifica che sia presente e nel formato corretto.
- **Password**:
  - Deve rispettare i criteri di sicurezza:
    - Lunghezza minima di 8 caratteri.
    - Deve includere:
      - Una lettera maiuscola.
      - Una lettera minuscola.
      - Un numero.
      - Un carattere speciale.

### **2. Decodifica e Validazione del Token**
- Utilizzare `URLSafeTimedSerializer` per:
  - Decodificare il token.
  - Verificare che non sia scaduto (validità di 15 minuti).
  - Estrarre l'email associata.
- **Gestione degli Errori**:
  - **Token scaduto**: Restituire un errore specifico.
  - **Token non valido**: Restituire un errore specifico.

### **3. Protezione contro Race Condition (miglioria opzionale)**
- Utilizzare un lock temporaneo per impedire l'uso simultaneo dello stesso token su richieste concorrenti.

### **4. Recupero dell'Utente**
- Cercare l'utente associato all'email estratta dal token.
- Restituire un errore se l'utente non esiste.

### **5. Aggiornamento della Password**
- Hashare la nuova password utilizzando `bcrypt`.
- Aggiornare il campo `password_hash` nel database.

### **6. Invalidazione del Token**
- Aggiungere un identificativo univoco (`jti`) al token.
- Memorizzare i token utilizzati in una blacklist per invalidarli dopo l'uso.

### **7. Logging Dettagliato**
- Registrare:
  - Email associate alla richiesta.
  - Risultati (successo/fallimento).
  - Timestamp.

### **8. Rate Limiting Globale (miglioria opzionale)**
- Limitare le richieste consecutive per IP o email per prevenire attacchi.

---

## 4. Risposte dell'Endpoint

| **Scenario**               | **HTTP Status**   | **Messaggio**                                |
|----------------------------|-------------------|---------------------------------------------|
| **Successo**               | `200 OK`         | "Password reset successfully."              |
| **Token Mancante/Non Valido** | `400 Bad Request` | "Invalid or expired token."                |
| **Password Non Valida**     | `400 Bad Request` | "Invalid password format."                  |
| **Utente Non Trovato**      | `404 Not Found`   | "User not found."                           |

Esempio di Risposta Successo:
```
{
  "status": "success",
  "code": 200,
  "message": "Password reset successfully."
}
```

---

## 5. Codice Aggiornato

```
from flask import request, jsonify, current_app
from itsdangerous import URLSafeTimedSerializer, SignatureExpired, BadSignature
from app.models import User
from app import db, blacklist
import bcrypt
import logging
from threading import Lock

# Configurazione logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Lock per protezione contro race condition
reset_lock = Lock()

@v1.route('/reset_password', methods=['POST'])
def reset_password():
    data = request.get_json()
    token = data.get('token')
    new_password = data.get('new_password')

    # Validazione input
    if not token or not new_password:
        logger.warning("Reset password failed: Missing token or new_password.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Missing token or new_password."
        }), 400

    # Validazione password
    if len(new_password) < 8 or not any(char.isdigit() for char in new_password) or not any(char.isalpha() for char in new_password):
        logger.warning("Reset password failed: Invalid password format.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Invalid password format."
        }), 400

    # Decodifica del token
    try:
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        email = serializer.loads(
            token,
            salt=current_app.config['SECURITY_PASSWORD_SALT'],
            max_age=900  # Token valido per 15 minuti
        )
    except SignatureExpired:
        logger.warning("Reset password failed: Token expired.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Invalid or expired token."
        }), 400
    except BadSignature:
        logger.warning("Reset password failed: Invalid token.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Invalid or expired token."
        }), 400

    # Protezione contro race condition
    with reset_lock:
        if token in blacklist:
            logger.warning("Reset password failed: Token already used.")
            return jsonify({
                "status": "error",
                "code": 400,
                "message": "Invalid or expired token."
            }), 400

        # Recupero utente
        user = User.query.filter_by(email=email).first()
        if not user:
            logger.warning(f"Reset password failed: User not found for email {email}.")
            return jsonify({
                "status": "error",
                "code": 404,
                "message": "User not found."
            }), 404

        # Aggiornamento password
        try:
            hashed_password = bcrypt.hashpw(new_password.encode('utf-8'), bcrypt.gensalt())
            user.password_hash = hashed_password
            db.session.commit()
            logger.info(f"Password reset successfully for user {email}.")
        except Exception as e:
            logger.error(f"Error updating password: {e}")
            return jsonify({
                "status": "error",
                "code": 500,
                "message": "Error resetting password."
            }), 500

        # Invalidazione del token
        blacklist.add(token)

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "Password reset successfully."
    }), 200
```

---

## 6. Sicurezza e Prossimi Passi

### **Sicurezza**
1. **Rotazione dei Token**:
   - Implementata con l'uso di una blacklist centralizzata.
2. **Protezione CSRF (miglioria opzionale)**:
   - Aggiungere un token CSRF per garantire che le richieste provengano da fonti legittime.

### **Prossimi Passi**
1. **Test Completi**:
   - Validare scenari complessi:
     - Token scaduti o non validi.
     - Password non conformi.
     - Utente inesistente.
2. **Integrazione Frontend**:
   - Creare una UI dedicata per l'inserimento del token e della nuova password.
3. **Monitoraggio e Analisi**:
   - Utilizzare strumenti di monitoraggio (es. Grafana) per raccogliere statistiche sull'uso dell'endpoint.

---

