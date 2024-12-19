Ecco i dettagli per l’endpoint di modifica o aggiunta dell’avatar:

Endpoint: Modifica o Aggiunta dell’Avatar

Dettagli
	•	Endpoint: /upload_avatar
	•	Metodo: POST o PUT
	•	Autenticazione: Richiedere un token JWT per identificare e autorizzare l’utente.
	•	Parametri:
	•	avatar (file): Il file immagine selezionato dall’utente.

Comportamento
	1.	Autenticazione
	•	Recuperare il token JWT dall’intestazione Authorization.
	•	Verificare e decodificare il token per identificare l’utente.
	2.	Validazioni
	•	Verificare la presenza del file avatar nella richiesta.
	•	Consentire solo file con estensioni png, jpg, jpeg, gif.
	•	Controllare la dimensione massima del file (es. 5 MB).
	•	Verificare il tipo MIME del file (es. image/png, image/jpeg).
	3.	Salvataggio
	•	Generare un nome univoco per il file (es. {user_id}_avatar.png).
	•	Storage locale:
	•	Salvare l’immagine in una directory specifica sul server.
	•	Storage cloud:
	•	Caricare l’immagine su un servizio come AWS S3 o Google Cloud Storage e ottenere l’URL del file.
	4.	Aggiornamento del Database
	•	Salvare il percorso o l’URL dell’avatar nella tabella User.
	5.	Risposta
	•	Restituire uno stato di successo e l’URL del nuovo avatar.

Esempio di Risposta

Richiesta
	•	File immagine inviato nel campo avatar.

Risposta in caso di successo

{
  "status": "success",
  "code": 200,
  "data": {
    "avatar_url": "https://storage.example.com/avatars/123_avatar.png"
  }
}

Risposta in caso di errore (es. file non valido)

{
  "status": "error",
  "code": 400,
  "message": "Invalid file type"
}

Esempio di Codice

@v1.route('/upload_avatar', methods=['POST'])
def upload_avatar():
    # Autenticazione tramite token JWT
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify_return_error("error", 401, "Authorization token required"), 401
```py
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
