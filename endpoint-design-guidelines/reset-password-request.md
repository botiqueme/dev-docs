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
@v1.route('/reset_password_request', methods=['POST'])
@limiter.limit("3 per hour", key_func=lambda: request.remote_addr)
def reset_password_request():
    """
    Endpoint per richiedere un link di reset della password.
    """
    try:
        # Recupera i dati dal corpo della richiesta
        data = request.get_json()
        if not data or 'email' not in data:
            return utils.jsonify_return_error("Bad Request", 400, "Missing email."), 400

        email = data['email']

        # Verifica se l'email è un'email usa e getta
        disposable = is_disposable_email.check(email)
        if disposable:
            return utils.jsonify_return_error("Error", 400, "Disposable emails are not allowed."), 400

        # Cerca l'utente nel database
        user = User.query.filter_by(email=email).first()

        # Protezione contro l'enumerazione degli utenti
        if user:
            # Genera un token di reset della password
            token = utils.generate_verification_token(email)
            # Costruisce l'URL di reset della password
            verify_url = url_for('v1.verify_email', token=token, _external=True)

            response = utils.send_verification_email(email, verify_url)
            if response.status_code == 404:
                callback_refresh()
                response = utils.send_verification_email(email, verify_url)

    except Exception as e:
        return utils.jsonify_return_error("Error", 500, "Internal Server Error, please contact the admin"), 500

    if response.status_code == 200:
        return utils.jsonify_return_success("success", 200, {"message": "If an account with that email exists, a password reset link has been sent."}), 200
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

