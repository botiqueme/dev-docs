Perfetto! Ecco il recap in italiano da inoltrare direttamente al tuo sviluppatore. Puoi incollarlo in un’email, in Slack o su qualsiasi documento interno.

⸻

✅ Email da attivare su Stripe

Queste sono le email che Stripe può inviare automaticamente agli utenti finali, senza doverle gestire manualmente lato backend. Sono sufficienti per coprire i casi principali (pagamento, rinnovo, errore).

📌 Email da attivare in Stripe

Vai su: https://dashboard.stripe.com/settings/billing/automatic

Verifica che siano abilitate le seguenti opzioni:
	•	✅ Invia email per i pagamenti riusciti
Stripe invia una conferma automatica dopo ogni addebito andato a buon fine
	•	✅ Invia email per i pagamenti non riusciti
L’utente viene avvisato che la carta è stata rifiutata e riceve un link per aggiornarla
	•	✅ Invia email prima del rinnovo automatico
Utile soprattutto per i piani annuali
	•	✅ Invia email per abbonamenti cancellati
Quando una subscription termina (es. per mancato pagamento), Stripe avvisa l’utente

⸻

🧾 Invio automatico della fattura (invoice)

Per fare in modo che Stripe invii automaticamente la fattura al cliente via email, bisogna fare quanto segue:

1. Aggiungere l’email del cliente

Quando create il customer via API, assicuratevi che il campo email sia popolato:

{
  "email": "cliente@email.com"
}


⸻

2. Abilitare l’invio automatico delle fatture

Andare su: https://dashboard.stripe.com/settings/billing/invoices

Attivare:
	•	✅ Invia automaticamente le fatture finalizzate ai clienti
	•	✅ (opzionale) Allega la fattura in PDF all’email

⸻

3. Finalizzare la fattura (solo per invoice manuali)
	•	Se usate subscriptions, Stripe finalizza l’invoice in automatico.
	•	Se invece create invoice manualmente via API, dovete finalizzarle:

POST /v1/invoices/{INVOICE_ID}/finalize


⸻

4. (Opzionale) Impostare la lingua dell’email

Potete definire una lingua preferita per ogni customer:

{
  "preferred_locales": ["it", "en"]
}


⸻

✅ Con queste impostazioni, Stripe si occuperà in autonomia dell’invio delle email di pagamento, rinnovo e fatturazione, evitando di doverle ricreare lato backend.

Fammi sapere se vuoi anche una versione .txt o .pdf pronta da inviare.