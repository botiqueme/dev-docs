# Endpoint: Aggiunta o Modifica dell'Avatar

## Dettagli
- **Endpoint**: `/upload_avatar`
- **Metodo**: `POST` o `PUT`
- **Autenticazione**: Richiedere un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di caricare un nuovo avatar o sostituire quello esistente.
- **Parametri**:
  - `avatar` (file): Il file immagine selezionato dall'utente.

---

## Comportamento

### 1. Autenticazione
- Recuperare il token JWT dall’intestazione `Authorization`.
- Verificare e decodificare il token per identificare l’utente.

### 2. Validazioni
- Verificare la presenza del file `avatar` nella richiesta.
- Consentire solo file con estensioni `png`, `jpg`, `jpeg`, `gif`.
- Controllare la dimensione massima del file (es. 5 MB).
- Verificare il tipo MIME del file (es. `image/png`, `image/jpeg`).

### 3. Salvataggio
- Generare un nome univoco per il file (es. `{user_id}_avatar.png`).
- **Storage locale**:
  - Salvare il file in una directory specifica sul server.
- **Storage cloud**:
  - Caricare il file su un servizio come AWS S3 o Google Cloud Storage e ottenere l’URL del file.

### 4. Aggiornamento del Database
- Salvare il percorso o l’URL dell’avatar nella tabella `User`.

### 5. Risposta
- Restituire uno stato di successo e l’URL del nuovo avatar.

---

## Esempio di Risposte

### Richiesta
- File immagine inviato nel campo `avatar`.

### Risposta in caso di successo
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "avatar_url": "https://storage.example.com/avatars/123_avatar.png"
  }
}
```

### Risposta in caso di errore
```json
{
  "status": "error",
  "code": 400,
  "message": "Invalid file type"
}
````

## Esempio di codice

```py
@v1.route('/upload_avatar', methods=['POST'])
def upload_avatar():
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

    # Recupero del file
    avatar_file = request.files.get('avatar')
    if not avatar_file:
        return jsonify_return_error("error", 400, "No file provided"), 400

    # Validazioni del file
    allowed_extensions = {'png', 'jpg', 'jpeg', 'gif'}
    if '.' not in avatar_file.filename or avatar_file.filename.split('.')[-1].lower() not in allowed_extensions:
        return jsonify_return_error("error", 400, "Invalid file type"), 400

    if avatar_file.content_length > 5 * 1024 * 1024:  # 5 MB
        return jsonify_return_error("error", 400, "File too large"), 400

    # Salvataggio del file
    filename = f"{user.id}_{avatar_file.filename}"
    upload_path = f"/path/to/upload/{filename}"  # Cambia con la tua directory
    avatar_file.save(upload_path)

    # Simulazione URL (se usi storage cloud, cambia il percorso)
    avatar_url = f"https://storage.example.com/avatars/{filename}"

    # Aggiornamento del database
    user.avatar = avatar_url
    db.session.commit()

    return jsonify_return_success("success", 200, {"avatar_url": avatar_url})
```

## Considerazioni Importanti

1. **Validazioni Avanzate**:
   - Verificare che il file caricato sia effettivamente un’immagine.
   - Controllare eventuali limitazioni sulla risoluzione dell’immagine (opzionale).

2. **Storage Cloud**:
   - Se utilizzi uno storage cloud, integra le API del provider per caricare il file e ottenere l’URL.

3. **Sovrascrittura**:
   - L'endpoint dovrebbe sempre sovrascrivere l’avatar esistente con quello nuovo.

4. **Gestione degli Errori**:
   - Restituire messaggi chiari per problemi relativi all'autenticazione, validazione del file, o errori di salvataggio.

---

## Prossimi Passi

1. **Implementare l'endpoint** seguendo le specifiche.
2. **Configurare lo storage**:
   - Storage locale (directory sul server) o cloud (es. AWS S3).
3. **Testare** i seguenti scenari:
   - Caricamento iniziale dell’avatar.
   - Sostituzione di un avatar esistente.
   - Invio di un file non valido (es. dimensioni o formato errato).
4. **Aggiornare la documentazione API** per includere i dettagli di questo endpoint.

