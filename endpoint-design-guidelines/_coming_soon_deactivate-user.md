coming soon
/deactivate_user (opzionale, per amministratori o utenti)
	•	Scopo: Disattivare un account utente temporaneamente.
	•	Metodo: PATCH
	•	Autenticazione: Richiede token JWT.
	•	Considerazioni:
	•	Aggiornare lo stato dell’utente senza eliminarlo.
	•	Prevedere un endpoint per riattivare l’account (/reactivate_user).
