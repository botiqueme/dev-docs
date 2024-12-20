# Documentazione Endpoint: `/verify_email`

## Scopo
Validare l'email di un utente tramite un token univoco generato al momento della registrazione.

---

## Specifiche Tecniche

### **Metodo**
`GET`

### **URL**
`/verify_email`

### **Autenticazione**
Nessuna (pubblico).

### **Parametri**
| **Parametro** | **Tipo**  | **Obbligatorio** | **Descrizione**                             |
|---------------|-----------|------------------|---------------------------------------------|
| `token`       | Stringa   | Sì               | Token univoco per verificare l'indirizzo email. |

---

## Logica dell'Endpoint

1. **Validazione del Token**:
   - Controlla se il token è presente nella richiesta.
   - Verifica che il token non sia nella blacklist.
   - Decodifica il token e controlla la sua validità (non scaduto o corrotto).

2. **Recupero dell'Utente**:
   - Utilizza l'email decodificata dal token per cercare l'utente nel database.
   - Restituisce un errore se l'utente non è trovato.

3. **Verifica dello Stato dell'Utente**:
   - Controlla se l'utente è già verificato.
   - Se sì, restituisce un errore indicando che l'utente è già verificato.

4. **Aggiornamento dello Stato**:
   - Aggiorna il campo `is_verified` dell'utente a `True`.

5. **Logging**:
   - Registra i tentativi di verifica completati o falliti per monitoraggio.

---

## Risposte

### **Successo**
| **HTTP Status** | **Messaggio**                          |
|-----------------|---------------------------------------|
| `200 OK`        | "Email verified successfully."        |

### **Errori**
| **Causa**                  | **HTTP Status**   | **Messaggio**                                |
|----------------------------|-------------------|---------------------------------------------|
| Token mancante             | `400 Bad Request` | "Missing token."                           |
| Token scaduto o non valido | `401 Unauthorized`| "Invalid or expired token."               |
| Utente non trovato         | `404 Not Found`   | "User not found."                          |
| Utente già verificato      | `409 Conflict`    | "User is already verified."                |
| Rate limit superato        | `429 Too Many Requests` | "Too many attempts. Try again later."    |

---

## Codice di Implementazione

```python
from flask import request, jsonify, current_app
from itsdangerous import URLSafeTimedSerializer, SignatureExpired, BadSignature
from app.models import User
from app import db, blacklist
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
import logging

# Configurazione logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Configurazione rate limiter
limiter = Limiter(key_func=get_remote_address)

@v1.route('/verify_email', methods=['GET'])
@limiter.limit("5 per minute", override_defaults=False, deduct_when=lambda resp: resp.status_code == 200)
@limiter.limit("3 per minute", override_defaults=False, deduct_when=lambda resp: resp.status_code != 200)
def verify_email():
    token = request.args.get('token')

    # Validazione token mancante
    if not token:
        logger.warning("Verification failed: Missing token.")
        return jsonify({
            "status": "error",
            "code": 400,
            "message": "Missing token."
        }), 400

    # Verifica se il token è nella blacklist
    if token in blacklist:
        logger.warning(f"Verification failed: Token {token} blacklisted.")
        return jsonify({
            "status": "error",
            "code": 401,
            "message": "Token invalidated."
        }), 401

    try:
        # Decodifica del token
        serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
        email = serializer.loads(
            token,
            salt=current_app.config['SECURITY_PASSWORD_SALT'],
            max_age=86400  # Token valido per 24 ore
        )
    except SignatureExpired:
        logger.warning("Verification failed: Token expired.")
        return jsonify({
            "status": "error",
            "code": 401,
            "message": "Invalid or expired token. Please request a new one."
        }), 401
    except BadSignature:
        logger.warning("Verification failed: Invalid token signature.")
        return jsonify({
            "status": "error",
            "code": 401,
            "message": "Invalid or expired token."
        }), 401

    # Recupero utente dal database
    user = User.query.filter_by(email=email).first()

    if not user:
        logger.warning(f"Verification failed: User not found for email {email}.")
        return jsonify({
            "status": "error",
            "code": 404,
            "message": "User not found."
        }), 404

    # Controllo se l'utente è già verificato
    if user.is_verified:
        logger.info(f"Verification skipped: User {email} already verified.")
        return jsonify({
            "status": "error",
            "code": 409,
            "message": "User is already verified."
        }), 409

    # Aggiornamento stato utente
    user.is_verified = True
    db.session.commit()

    logger.info(f"Verification successful: User {email} verified.")
    return jsonify({
        "status": "success",
        "code": 200,
        "message": "Email verified successfully."
    }), 200
```

# Prossimi Passi per l'Endpoint `/verify_email`

## 1. **Monitoraggio Completo**
### Obiettivo
Integrare strumenti di monitoraggio come **Prometheus** o **Grafana** per raccogliere e analizzare metriche dettagliate sulle verifiche email.

### Azioni Necessarie
- Configurare **Prometheus Flask Exporter**:
  - Raccogliere metriche come:
    - Numero di verifiche completate con successo.
    - Tentativi falliti (es. token non valido, token scaduto).
    - Frequenza di token invalidati.
  - Esempio di codice per esportare metriche:
    ```python
    from prometheus_flask_exporter import PrometheusMetrics
    metrics = PrometheusMetrics(app)
    metrics.info('app_info', 'Application Info', version='1.0.0')
    ```

- Integrare **Grafana**:
  - Creare dashboard per visualizzare le metriche raccolte da Prometheus.
  - Aggiungere grafici per monitorare trend di verifiche email.

---

## 2. **Pagina Frontend**
### Obiettivo
Migliorare l'esperienza utente fornendo una pagina dedicata per il feedback visivo in base allo stato della verifica.

### Azioni Necessarie
- Creare una pagina HTML con i seguenti stati:
  - **Successo**:
    - Messaggio: "La tua email è stata verificata con successo."
  - **Token Scaduto**:
    - Messaggio: "Il link è scaduto. Richiedi un nuovo token."
    - Pulsante: "Richiedi Nuovo Token" (collegato a `/resend_verification`).
  - **Token Non Valido**:
    - Messaggio: "Errore durante la verifica. Contatta l'assistenza."
  
- Aggiornare l'endpoint `/verify_email` per restituire una pagina HTML per richieste da browser:
  ```python
  if request.accept_mimetypes['text/html']:
      return render_template('success.html')
  else:
      return jsonify({"status": "success", "message": "Email verified successfully."}), 200
  ```

  ## 3. **Test Automatici**
### Obiettivo
Garantire la qualità e il corretto funzionamento dell'endpoint in tutti i casi d'uso.

### Azioni Necessarie
- Creare test unitari e di integrazione per coprire:
  - **Verifiche con Token Validi**:
    - Testare che un token valido verifichi correttamente l'utente.
  - **Verifiche con Token Invalidi**:
    - Testare token scaduti o con firma non valida.
  - **Blacklist**:
    - Testare che i token nella blacklist vengano respinti.
  - **Rate Limiting**:
    - Verificare che i limiti siano applicati correttamente:
      - 5 richieste al minuto per token validi.
      - 3 richieste al minuto per token non validi.

- Usare framework come **pytest** o **unittest**:
  - Esempio di test:
    ```python
    def test_verify_email_success(client):
        response = client.get('/verify_email?token=valid_token')
        assert response.status_code == 200
        assert response.json['message'] == "Email verified successfully."

    def test_verify_email_blacklisted(client):
        response = client.get('/verify_email?token=blacklisted_token')
        assert response.status_code == 401
        assert response.json['message'] == "Token invalidated."
    ```

---

## 4. **Ottimizzazione per Scalabilità**
### Obiettivo
Assicurare che il sistema rimanga performante in ambienti con molte richieste simultanee.

### Azioni Necessarie
- **Distribuzione della Blacklist**:
  - Usare un sistema distribuito come **Redis** per sincronizzare la blacklist tra più istanze backend.
  
- **Verifica dei Limiti di Carico**:
  - Eseguire test di carico con strumenti come **Apache JMeter** o **k6**.
  - Monitorare l'uso delle risorse durante test intensivi:
    - Percentuale di richieste completate entro un certo tempo.
    - Tassi di errore durante carichi elevati.

---

## Priorità
1. **Pagina Frontend**: Migliora immediatamente l'esperienza utente.
2. **Test Automatici**: Garantisce qualità e stabilità del sistema.
3. **Monitoraggio Completo**: Permette un'analisi continua delle metriche.
4. **Ottimizzazione per Scalabilità**: Essenziale per ambienti con traffico elevato.
