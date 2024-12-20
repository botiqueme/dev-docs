
# Endpoint: `/update_password`

## 1. Dettagli
- **URL**: `/update_password`
- **Metodo**: `PUT`
- **Autenticazione**: Richiede un token JWT per identificare e autorizzare l’utente.
- **Scopo**: Consentire agli utenti di aggiornare la propria password in modo sicuro.

---

## 2. Implementazione del Rate Limiting
Il rate limiting è stato aggiunto per impedire attacchi brute force. Sono consentite:
- **3 tentativi per minuto per IP**.
- Configurabile per modificare il limite se necessario.

---

## 3. Codice Aggiornato con Rate Limiting

```
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

# Configurazione del rate limiting
limiter = Limiter(key_func=get_remote_address, default_limits=["3 per minute"])

@v1.route('/update_password', methods=['PUT'])
@limiter.limit("3 per minute")  # Rate limiting specifico per l'endpoint
def update_password():
    # Autenticazione tramite token JWT
    auth_header = request.headers.get('Authorization')
    if not auth_header:
        return jsonify_return_error("error", 401, "Authorization token required."), 401

    try:
        token = auth_header.split(" ")[1]
        payload = jwt.decode(token, current_app.config['SECRET_KEY'], algorithms=['HS256'])
        user_id = payload['user_id']
    except Exception:
        return jsonify_return_error("error", 401, "Invalid token."), 401

    # Recupero dell'utente
    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify_return_error("error", 404, "User not found."), 404

    # Recupero dati dal body della richiesta
    data = request.get_json()
    current_password = data.get('current_password')
    new_password = data.get('new_password')

    if not current_password or not new_password:
        return jsonify_return_error("error", 400, "Both current and new passwords are required."), 400

    # Verifica password attuale
    if not verify_password(user.password_hash, current_password):
        return jsonify_return_error("error", 400, "Current password is incorrect."), 400

    # Validazione della nuova password
    if current_password == new_password:
        return jsonify_return_error("error", 400, "New password must be different from the current password."), 400

    # Aggiornamento della password
    user.password_hash = hash_password(new_password)
    db.session.commit()

    return jsonify({
        "status": "success",
        "code": 200,
        "message": "Password updated successfully."
    }), 200
```

---

## 4. Considerazioni sul Rate Limiting

1. **Configurabilità**:
   - La limitazione è attualmente impostata a **3 tentativi per minuto per IP**.
   - Può essere modificata in base ai requisiti di sicurezza (es. `5 per 10 minuti` o `10 per giorno`).

2. **Gestione degli Errori di Rate Limiting**:
   - Se un utente supera il limite di richieste, Flask-Limiter restituirà automaticamente:
     - **HTTP Status**: `429 Too Many Requests`
     - **Body**:
```
{
  "status": "error",
  "code": 429,
  "message": "Too many requests. Please try again later."
}
```

3. **Prevenzione degli Abusi**:
   - Il rate limiting protegge da attacchi brute force senza bloccare l'intero sistema.

---

## 5. Prossimi Passi

1. **Testare il comportamento del rate limiting**:
   - Simulare più tentativi di aggiornamento password in breve tempo per verificare che il rate limiting funzioni correttamente.
2. **Integrare il rate limiting su altri endpoint sensibili**, come `/login` o `/change_email`.
3. **Documentare il comportamento del rate limiting** per i team di sviluppo e QA.

Se hai bisogno di ulteriori modifiche o di altri miglioramenti, fammi sapere! 😊
