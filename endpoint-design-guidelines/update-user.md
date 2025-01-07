# Endpoint: /update_user

## Dettagli
- **URL**: `/update_user`
- **Metodo**: `PUT` o `PATCH` (preferibile per aggiornamenti parziali).
- **Autenticazione**: Richiedere un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di aggiornare le proprie informazioni personali.

## Comportamento

### 1. Autenticazione
- Recuperare il token JWT dall’intestazione `Authorization`.
- Decodificare il token per ottenere l’`user_id`.
- Verificare che l’utente esista nel database.

### 2. Parametri Accettati

Il corpo della richiesta deve essere in formato JSON. Campi aggiornabili:
- **name** (stringa)
- **surname** (stringa)
- **phone_number** (stringa, formato validato)
- **company** (stringa)
- **vat_number** (stringa, formato validato)

Nota: È importante validare ogni campo prima di aggiornarlo nel database (verifica fatta in frontend)

### 3. Aggiornamento
- Aggiornare solo i campi presenti nella richiesta.
- Se un campo non è incluso, mantenerne il valore attuale nel database.

### 4. Risposta
- Restituire uno stato di successo con i dati aggiornati o un messaggio di errore in caso di fallimento.

## 3. Esempio di Richiesta

Richiesta `PATCH /update_user`

```
{
  "name": "John",
  "surname": "Doe",
  "phone_number": "+1234567890",
  "company": "Example Corp",
  "vat_number": "IT12345678901"
}
```

## 4. Risposte dell’Endpoint

### Successo

| HTTP Status | Messaggio     |
|-------------|---------------|
| 200 OK      | "Successo"    |

**Body:**
```
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

### Errore: Utente Non Autenticato

| HTTP Status | Messaggio                  |
|-------------|----------------------------|
| 401 Unauthorized | "Authorization token required." |

**Body:**
```
{
  "status": "error",
  "code": 401,
  "message": "Authorization token required."
}
```

### Errore: Utente Non Trovato

| HTTP Status | Messaggio                  |
|-------------|----------------------------|
| 404 Not Found | "User not found."         |

**Body:**
```
{
  "status": "error",
  "code": 404,
  "message": "User not found."
}
```
### Errore: No Data Provided

| HTTP Status | Messaggio                  |
|-------------|----------------------------|
| 400 Error | "No data provided."         |

**Body:**
```
{
  "status": "error",
  "code": 400,
  "message": "No Data Provided."
}
```

## 5. Codice Aggiornato

```
@v1.route('/update_user', methods=['PATCH'])
@jwt_required()
def update_user():
    """
    Endpoint per aggiornare le informazioni dell'utente.
    Richiede autenticazione tramite JWT.
    """
    try:
        # Ottieni l'ID dell'utente dal token JWT
        user_id = get_jwt_identity()

        # Recupera l'utente dal database
        user = User.query.get(user_id)
        if not user:
            return utils.jsonify_return_error("error", 404, "User not found"), 404
        
        # Recupera i dati dal corpo della richiesta
        data = request.get_json()
        if not data:
            return utils.jsonify_return_error("error", 400, "No data provided"), 400
        
        # Aggiorna i campi forniti nella richiesta
        if 'name' in data:
            user.name = data['name']
        if 'surname' in data:
            user.surname = data['surname']
        if 'phone_number' in data:
            user.phone_number = data['phone_number']
        if 'company' in data:
            user.company = data['company']
        if 'vat_number' in data:
            user.vat_number = data['vat_number']
        
        # Salva le modifiche nel database
        try:
            db.session.commit()
        except Exception:
            db.session.rollback()
            return utils.jsonify_return_error("error", 500, "An error occurred while updating the user"), 500

        # Prepara i dati aggiornati per la risposta
        updated_data = {
            "name": user.name,
            "surname": user.surname,
            "phone_number": user.phone_number,
            "company": user.company,
            "vat_number": user.vat_number
        }

        # Restituisce una risposta di successo con i dati aggiornati
        return utils.jsonify_return_success("success", 200, updated_data), 200
    except Exception as e:
        # Gestione di errori imprevisti
        return utils.jsonify_return_error("error", 500, "An unexpected error occurred"), 500

```

## 6. Validazioni da Implementare
1. **Formato Numero di Telefono**:
   - Verificare che `phone_number` sia in un formato valido (es. con prefisso internazionale, come +1234567890).
2. **Partita IVA**:
   - Verificare che `vat_number` sia valido per il paese (es. per l’Italia, formato IT12345678901).
3. **Dati Mancanti**:
   - Ignorare i campi non inclusi nella richiesta senza sovrascrivere i valori esistenti.

## 7. Considerazioni Aggiuntive (Migliorie Opzionali)

1. **Gestione degli Errori Dettagliata**:
   - Fornire errori specifici per ogni tipo di fallimento, come errore nel formato dei dati o problemi con il salvataggio nel database.
   - **Miglioria opzionale**: Includere il tipo di errore (es. errore di convalida, errore di database) nella risposta per facilitare la risoluzione dei problemi.
2. **Rate Limiting**:
   - Limitare il numero di richieste per IP o utente per prevenire abusi (e.g. 5 richieste per minuto).
   - **Miglioria opzionale**: Aggiungere il rate limiting per prevenire tentativi di abuso dell’endpoint (ad esempio attacchi brute force).
3. **Audit Log**:
   - Creare un log di audit per tracciare quando e da chi sono stati effettuati aggiornamenti ai dati dell’utente.
   - **Miglioria opzionale**: Integrare un sistema di logging per monitorare le modifiche e mantenere traccia delle operazioni per scopi di sicurezza e debugging.

## 8. Prossimi Passi
1. Implementare l’endpoint seguendo le specifiche.
2. Testare i seguenti scenari:
   - Aggiornamento parziale (es. modifica solo del `name`).
   - Aggiornamento completo con tutti i campi.
   - Invio di dati non validi (es. numero di telefono errato).
3. Aggiornare la documentazione API per includere dettagli su questo endpoint.
4. Integrare il rate limiting per prevenire abusi.
5. Valutare l’implementazione di un sistema di audit log per monitorare gli aggiornamenti.
