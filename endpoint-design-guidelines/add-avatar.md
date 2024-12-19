Ecco le istruzioni strutturate per il backendista in modo che possa implementare l’upload dell’avatar basandosi sul lavoro del frontendista:

Implementazione Endpoint per l’Upload dell’Avatar

1. Contesto

L’utente, nella schermata di modifica del profilo, può cliccare su “Aggiungi” per selezionare un’immagine dal proprio file system e caricarla come avatar. L’operazione deve essere gestita dal backend tramite un endpoint dedicato.

2. Endpoint da Implementare

Descrizione
	•	Endpoint: /upload_avatar
	•	Metodo: POST
	•	Autenticazione: L’utente deve essere autenticato (tramite token JWT).
	•	Parametri:
	•	avatar (file): Il file immagine selezionato dall’utente.
	•	Risultato:
	•	Salva l’immagine e restituisce l’URL del file caricato.

3. Specifiche di Implementazione

a. Validazione dei File
	•	Consentire solo immagini con estensioni png, jpg, jpeg, gif.
	•	Limitare la dimensione del file (es. massimo 5 MB).
	•	Verificare che il file caricato sia effettivamente un’immagine (controllo tipo MIME).

b. Salvataggio del File
	•	Opzione 1: Storage Locale:
	•	Salvare l’immagine in una directory specifica sul server.
	•	Esempio: /path/to/upload/{user_id}_avatar.png.
	•	Opzione 2: Storage Cloud:
	•	Caricare l’immagine su un servizio come AWS S3 o Google Cloud Storage.
	•	Restituire l’URL pubblico o autenticato del file.

c. Aggiornamento del Database
	•	Aggiornare la tabella User per salvare il percorso (locale o URL) dell’avatar.

d. Risposta
	•	Restituire un JSON con lo stato dell’operazione e l’URL dell’avatar caricato.

4. Codice di Esempio

Endpoint /upload_avatar
```
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

    # Recupero del file dalla richiesta
    avatar_file = request.files.get('avatar')
    if not avatar_file:
        return jsonify_return_error("error", 400, "No file provided"), 400

    # Validazione del file
    allowed_extensions = {'png', 'jpg', 'jpeg', 'gif'}
    if '.' not in avatar_file.filename or avatar_file.filename.split('.')[-1].lower() not in allowed_extensions:
        return jsonify_return_error("error", 400, "Invalid file type"), 400

    if avatar_file.content_length > 5 * 1024 * 1024:
        return jsonify_return_error("error", 400, "File too large"), 400

    # Salvataggio del file (storage locale o cloud)
    filename = f"{user.id}_{avatar_file.filename}"
    upload_path = f"/path/to/upload/{filename}"  # Cambiare con la directory effettiva
    avatar_file.save(upload_path)

    # URL simulato (da cambiare se si usa storage cloud)
    avatar_url = f"https://storage.example.com/avatars/{filename}"

    # Aggiornamento del database
    user.avatar = avatar_url
    db.session.commit()

    return jsonify_return_success("success", 200, {"avatar_url": avatar_url})
```
5. Validazioni da Implementare
	•	Verificare:
	•	Tipo MIME del file (image/png, image/jpeg, ecc.).
	•	Dimensione massima accettabile.
	•	Che il file non contenga codice malevolo (opzionale).

6. Prossimi Passi
	1.	Implementare l’endpoint e testarlo localmente.
	2.	Configurare lo storage:
	•	Locale (es. directory sul server).
	•	Cloud (es. AWS S3 o Google Cloud Storage).
	3.	Testare l’integrazione con il frontend:
	•	Assicurarsi che il file inviato dal frontend venga processato correttamente e che il backend restituisca l’URL dell’avatar.
	4.	Aggiornare la documentazione API per il nuovo endpoint.

