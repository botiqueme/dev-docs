# Implementazione del Refresh Token**

## **1. Che Cos'è un Refresh Token?**
Un **refresh token** è un token a lunga durata utilizzato per ottenere un nuovo **access token** quando l'access token corrente è scaduto.

### **Differenza tra Access Token e Refresh Token**
| **Proprietà**        | **Access Token**               | **Refresh Token**              |
|-----------------------|--------------------------------|---------------------------------|
| **Scopo**            | Autenticazione delle richieste API. | Ottenere un nuovo access token. |
| **Durata**           | Breve (es. 15 minuti - 1 ora). | Lunga (es. giorni o settimane). |
| **Conservazione**    | Frontend (cookie HTTP-only o memoria sicura). | Backend o cookie sicuro.       |

---

## **2. Perché Serve un Refresh Token?**
### **Problemi con Solo Access Token**
- Quando un **access token** scade, l'utente deve effettuare nuovamente il login, interrompendo la sessione e creando una cattiva esperienza utente.

### **Benefici del Refresh Token**
1. **Esperienza Utente Migliore**:
   - L'utente rimane autenticato senza dover reinserire le credenziali.
2. **Maggiore Sicurezza**:
   - Riduce l'esposizione dell'access token a lungo termine.
3. **Controllo del Sessione**:
   - Il backend può invalidare il refresh token per revocare l'accesso in caso di necessità (es. logout o compromissione).

---

## **3. Quando Serve il Refresh Token?**
- Il refresh token è necessario quando l’access token è **scaduto**, ma l'utente deve rimanere autenticato senza effettuare un nuovo login.

### **Flusso Tipico del Refresh Token**
1. **Login**:
   - Dopo un login valido, il backend restituisce:
     - Un **access token** (breve durata).
     - Un **refresh token** (lunga durata).
2. **Richiesta API**:
   - Il frontend utilizza l'access token per autenticare le richieste API.
3. **Scadenza dell'Access Token**:
   - Se l'access token è scaduto, il frontend invia il refresh token al backend per ottenere un nuovo access token.
4. **Generazione di un Nuovo Token**:
   - Il backend valida il refresh token e restituisce un nuovo access token.

---

## **4. Implementazione del Refresh Token**

### **Endpoint per il Refresh Token**
| **Proprietà**        | **Valore**                     |
|-----------------------|--------------------------------|
| **Endpoint**         | `/refresh_token`               |
| **Metodo**           | `POST`                         |
| **Autenticazione**   | Necessario il refresh token.   |
| **Body della Richiesta** | Nessuno.                    |

### **Logica**
1. **Ricezione del Refresh Token**:
   - Recuperare il refresh token dall'intestazione `Authorization` o da un cookie HTTP-only.
2. **Validazione del Refresh Token**:
   - Decodificare il refresh token utilizzando la chiave segreta del server.
   - Verificare:
     - Validità del token (non scaduto).
     - Tipo di token (`refresh`).
     - Utente associato esistente.
   - Restituire errore se il token non è valido.
3. **Generazione di un Nuovo Access Token**:
   - Creare un nuovo access token con scadenza breve.
4. **Risposta al Frontend**:
   - Restituire il nuovo access token.

---

### **Risposte dell'Endpoint**
#### **Successo**
- **HTTP Status**: `200 OK`
- **Body**:
```
{
  "status": "success",
  "code": 200,
  "data": {
    "access_token": "nuovo_access_token"
  }
}
```

#### **Errori**
1. **Refresh Token Mancante**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Refresh token missing."
}
```

2. **Refresh Token Non Valido**:
   - **HTTP Status**: `401 Unauthorized`
   - **Body**:
```
{
  "status": "error",
  "code": 401,
  "message": "Invalid refresh token."
}
```

---

### **Codice di Esempio**
```
@v1.route('/refresh_token', methods=['POST'])
def refresh_token():
    # Recupero del refresh token dall'intestazione
    refresh_token = request.headers.get('Authorization')
    if not refresh_token:
        return jsonify({"error": "Refresh token missing"}), 401

    try:
        # Decodifica del token
        payload = jwt.decode(refresh_token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        if payload['type'] != 'refresh':
            return jsonify({"error": "Invalid token type"}), 401

        # Recupero utente
        user = User.query.filter_by(id=payload['user_id']).first()
        if not user:
            return jsonify({"error": "User not found"}), 404

        # Generazione di un nuovo access token
        new_access_token = jwt.encode({
            "user_id": user.id,
            "email": user.email,
            "exp": datetime.utcnow() + timedelta(minutes=15),
            "type": "access"
        }, current_app.config['SECRET_KEY'], algorithm='HS256')

        return jsonify({"access_token": new_access_token}), 200

    except jwt.ExpiredSignatureError:
        return jsonify({"error": "Refresh token expired"}), 401
    except Exception:
        return jsonify({"error": "Invalid token"}), 401
```

---

### **Considerazioni di Sicurezza**
1. **Salvataggio Sicuro del Refresh Token**:
   - Utilizzare cookie HTTP-only per impedire l'accesso tramite JavaScript.
2. **Rotazione del Refresh Token**:
   - Generare un nuovo refresh token ogni volta che viene utilizzato, invalidando il precedente.
3. **Blacklist**:
   - Salvare i refresh token revocati (es. durante il logout) in una blacklist per impedire usi futuri.
4. **Durata Limitata**:
   - Configurare una scadenza del refresh token adeguata (es. 7 giorni).
5. **Protezione CSRF**:
   - Implementare un token CSRF per le richieste che utilizzano cookie.

---

### **Prossimi Passi**
1. **Implementare l'endpoint `/refresh_token` seguendo le specifiche.**
2. **Aggiornare il frontend per gestire la scadenza dell'access token.**
3. **Testare**:
   - Richieste con un access token scaduto.
   - Richieste con refresh token non valido.
   - Rotazione dei token.
4. **Documentare il processo per il team di sviluppo.**
