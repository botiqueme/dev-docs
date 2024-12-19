# Endpoint: Aggiornamento delle Informazioni Utente

## Dettagli
- **Endpoint**: `/update_user`
- **Metodo**: `PUT` o `PATCH` (preferibile per aggiornamenti parziali).
- **Autenticazione**: Richiedere un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di aggiornare le proprie informazioni personali.

---

## Comportamento

### 1. Autenticazione
- Recuperare il token JWT dall’intestazione `Authorization`.
- Decodificare il token per ottenere l’identità dell’utente.
- Verificare che l’utente esista nel database.

### 2. Parametri Accettati
- Il body della richiesta deve essere in formato JSON.
- Campi aggiornabili:
  - `name` (stringa)
  - `surname` (stringa)
  - `phone_number` (stringa, formato validato)
  - `company` (stringa)
  - `vat_number` (stringa, formato validato)

> **Nota**: È importante validare ogni campo prima di aggiornarlo nel database.

### 3. Aggiornamento
- Aggiornare solo i campi presenti nella richiesta.
- Se un campo non è incluso, mantenerne il valore attuale nel database.

### 4. Risposta
- Restituire uno stato di successo con i dati aggiornati o un messaggio di errore in caso di fallimento.

---

## Esempio di Richiesta

### Richiesta `PUT /update_user`
```json
{
  "name": "John",
  "surname": "Doe",
  "phone_number": "+1234567890",
  "company": "Example Corp",
  "vat_number": "IT12345678901"
}
```

## Esempio di Risposra
### Risposta in caso di successo
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "name": "John",
    "surname": "Doe",
    "phone_number": "+1234567890",
    "company": "Example Corp",
    "vat_number": "IT12345678901"
  }
}
```

### Risposta in caso di errore

#### Utente non autenticato

```json
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required"
}
```
#### Utente non trovato

```json
{
  "status": "error",
  "code": 404,
  "message": "User not found"
}
````

## Esempio di codice 

```py
@v1.route('/update_user', methods=['PUT'])
def update_user():
    # Autenticazione tramite token JWT
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify_return_error("error", 401, "Authorization token required"), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        email = payload['email']
    except Exception:
        return jsonify_return_error("error", 401, "Invalid token"), 401

    # Recupero dell'utente
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify_return_error("error", 404, "User not found"), 404

    # Recupero dati dal body della richiesta
    data = request.get_json()
    user.name = data.get('name', user.name)
    user.surname = data.get('surname', user.surname)
    user.phone_number = data.get('phone_number', user.phone_number)
    user.company = data.get('company', user.company)
    user.vat_number = data.get('vat_number', user.vat_number)

    # Salvataggio delle modifiche
    db.session.commit()

    # Risposta con i dati aggiornati
    return jsonify_return_success("success", 200, {
        "name": user.name,
        "surname": user.surname,
        "phone_number": user.phone_number,
        "company": user.company,
        "vat_number": user.vat_number
    })
```

## Validazioni da Implementare

1. **Formato Numero di Telefono**:
   - Verificare che `phone_number` sia in un formato valido (es. con prefisso internazionale, come `+1234567890`).

2. **Partita IVA**:
   - Verificare che `vat_number` sia valido per il paese (es. per l’Italia, formato `IT12345678901`).

3. **Dati Mancanti**:
   - Ignorare i campi non inclusi nella richiesta senza sovrascrivere i valori esistenti.

---

## Prossimi Passi

1. **Implementare l'endpoint** seguendo le specifiche.
2. **Testare** i seguenti scenari:
   - Aggiornamento parziale (es. modifica solo del `name`).
   - Aggiornamento completo con tutti i campi.
   - Invio di dati non validi (es. numero di telefono errato).
3. **Aggiornare la documentazione API** per includere dettagli su questo endpoint.
