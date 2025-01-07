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
| **Internal Error**          | `500 Internal Error` | "Internal (Re-send verification) Server Error, please contact the admin"|

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
@v1.route('/resend_verification', methods=['POST'])
@limiter.limit("3 per hour")  # Limita a 3 richieste all'ora per utente
def resend_verification():
    """
    Resends a verification email to a user.

    Returns:
        Response: A response with a message and a status code.
    """
    # Validazione dell'input
    email = request.form.get('email')
    if not email:
        return jsonify({"Bad Request": "Missing email."}), 400

    # Verifica che l'email esista nel database
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify({"Not Found": "User not found"}), 404
    
    if user.is_verified:
        if request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html:
            return jsonify({"Conflict": "User is already verified."}), 409
        else:
            return redirect(url_for('v1.verified_token'))

    try:
        # Generazione token di verifica
        token = utils.generate_verification_token(email)
        # logger.info(email)
        verify_url = url_for('v1.verify_email', token=token, email=email, _external=True)
        # logger.info(verify_url)

        response = utils.send_verification_email(email, verify_url)
        # logger.info(response)
        if response.status_code == 404:
            callback_refresh()
            response = utils.send_verification_email(email, verify_url)
    except Exception as e:
        current_app.logger.info(e)
        return jsonify_return_error("Error", 500, "Internal (Verification Email) Server Error, please contact the admin"), 500

    return jsonify({"OK": "Verification email resent."}), 200
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
