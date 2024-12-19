Ecco la versione aggiornata dell’endpoint /register, con tutte le modifiche richieste:

# Endpoint: `/register`

## Dettagli
- **URL**: `/register`
- **Metodo**: `POST`
- **Scopo**: Registrare un nuovo utente con tutti i dati richiesti.
- **Autenticazione**: Nessuna (endpoint pubblico).

---

## Parametri Accettati
Il corpo della richiesta deve essere in formato JSON e includere i seguenti parametri:

| **Parametro**   | **Tipo**    | **Obbligatorio** | **Descrizione**                     |
|------------------|-------------|------------------|-------------------------------------|
| `email`         | Stringa     | Sì               | L'email dell'utente.               |
| `password`      | Stringa     | Sì               | La password da hashare.            |
| `name`          | Stringa     | Sì               | Il nome dell'utente.               |
| `surname`       | Stringa     | Sì               | Il cognome dell'utente.            |
| `phone_number`  | Stringa     | No               | Numero di telefono (validato).     |
| `company`       | Stringa     | No               | Nome dell'azienda.                 |
| `vat_number`    | Stringa     | No               | Partita IVA (validata).            |

Esempio di richiesta:
```
{
  "email": "user@example.com",
  "password": "securepassword",
  "name": "John",
  "surname": "Doe",
  "phone_number": "+1234567890",
  "company": "Example Corp",
  "vat_number": "IT12345678901"
}
```

---

## Logica dell'Endpoint

1. **Validazione dei Parametri**:
   - Controllare che i campi obbligatori siano presenti.
   - Validare il formato dei campi `email`, `phone_number` e `vat_number`.

2. **Hash della Password**:
   - Utilizzare `bcrypt` per hashare la password prima di salvarla.

3. **Generazione dell'User ID**:
   - Generare un **User ID** univoco per ogni utente (es. con UUID).

4. **Salvataggio nel Database**:
   - Inserire tutti i dati forniti, compresi i nuovi campi, nella tabella `users`.

5. **Risposta**:
   - Restituire una risposta JSON con il messaggio di successo e l'`user_id`.

---

## Risposte dell'Endpoint

### Successo
- **HTTP Status**: `201 Created`
- **Body**:
```
{
  "status": "success",
  "code": 201,
  "message": "User registered successfully",
  "data": {
    "user_id": "123e4567-e89b-12d3-a456-426614174000"
  }
}
```

### Errori
1. **Parametro Mancante**:
   - **HTTP Status**: `400 Bad Request`
   - **Body**:
```
{
  "status": "error",
  "code": 400,
  "message": "Missing required parameters: email, password, name, surname."
}
```
2. **Email Non Valida o Duplicata**:
   - **HTTP Status**: `409 Conflict`
   - **Body**:
```
{
  "status": "error",
  "code": 409,
  "message": "Email already registered or invalid."
}
```
3. **Errore Generico**:
   - **HTTP Status**: `500 Internal Server Error`
   - **Body**:
```
{
  "status": "error",
  "code": 500,
  "message": "An error occurred while creating the user."
}
```

---

## Codice Aggiornato

```
from flask import request, jsonify
import uuid
from app import db
from app.models import User
from sqlalchemy.exc import IntegrityError
import bcrypt

@v1.route('/register', methods=['POST'])
def register():
    data = request.get_json()

    # Validazione dei parametri
    required_fields = ['email', 'password', 'name', 'surname']
    missing_fields = [field for field in required_fields if field not in data]
    if missing_fields:
        return jsonify({
            "status": "error",
            "code": 400,
            "message": f"Missing required parameters: {', '.join(missing_fields)}."
        }), 400

    email = data['email']
    password = data['password']
    name = data['name']
    surname = data['surname']
    phone_number = data.get('phone_number')
    company = data.get('company')
    vat_number = data.get('vat_number')

    # Hash della password
    password_hash = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())

    # Generazione User ID
    user_id = str(uuid.uuid4())

    # Creazione utente
    new_user = User(
        user_id=user_id,
        email=email,
        password_hash=password_hash,
        name=name,
        surname=surname,
        phone_number=phone_number,
        company=company,
        vat_number=vat_number,
        is_verified=False
    )

    try:
        db.session.add(new_user)
        db.session.commit()
    except IntegrityError:
        db.session.rollback()
        return jsonify({
            "status": "error",
            "code": 409,
            "message": "Email already registered or invalid."
        }), 409
    except Exception as e:
        print(e)
        db.session.rollback()
        return jsonify({
            "status": "error",
            "code": 500,
            "message": "An error occurred while creating the user."
        }), 500

    return jsonify({
        "status": "success",
        "code": 201,
        "message": "User registered successfully",
        "data": {
            "user_id": user_id
        }
    }), 201
```

---

## Test

1. **Registrazione Valida**:
   - Inviare una richiesta completa e verificare che tutti i dati vengano salvati correttamente.
2. **Campi Mancanti**:
   - Simulare richieste con campi obbligatori mancanti e verificare la risposta con codice `400`.
3. **Email Duplicata**:
   - Provare a registrare un utente con un'email già presente nel database.
4. **Errore Generico**:
   - Simulare un errore del database per verificare la gestione degli errori.

