
# GUIDA COSTI WHATSAPP: MODELLO VONAGE (Aggiornato 2026)

## 1. Concetti Chiave: VONAGE

Vonage opera come Business Solution Provider (BSP) con un modello di costo ottimizzato per l'automazione.

* **Messaggi Ricevuti (Inbound) GRATUITI:** Vonage non addebita alcuna commissione sui messaggi che i guest inviano al tuo bot.
* **Messaggi Inviati (Outbound):** Paghi una tariffa per ogni messaggio inviato dal bot, che include già il markup di Vonage e la quota Meta per i messaggi fuori sessione.
* **Canone Numero:** Costo fisso mensile tra **$0.99 e $1.50** per il mantenimento del numero virtuale.
* **Vantaggio Scalabilità:** Poiché i messaggi in entrata sono gratis, il costo è molto più prevedibile rispetto a Twilio se gli utenti interagiscono molto con il bot.

---

## 2. Concetti Chiave: META

Il modello Meta 2026 si applica in modo identico anche su Vonage.

* **Service (Iniziate dall'Utente):**
* **Vantaggio:** Le prime **1.000 conversazioni** mensili sono **gratuite** lato Meta.
* Le risposte del bot entro la finestra di 24 ore sfruttano questa gratuità.


* **Utility & Marketing (Iniziate dal Business):**
* **Modello Pay-per-Message:** Ogni messaggio template (alert, check-in) viene pagato singolarmente.
* **Costo Utility Italia:** Circa **$0.00910** (tariffa finita Vonage + Meta).


* **Link Ufficiali:** [Vonage WhatsApp Pricing](https://www.vonage.com/communications-apis/messages/pricing/) | [Meta Developers Pricing](https://developers.facebook.com/docs/whatsapp/pricing).

---

## 3. Esempi Pratici (Costi per l'Italia)

### Caso A: Il Guest interagisce con il Chatbot

*Scenario: Guest scrive 2 messaggi, il bot risponde con 2 messaggi. Totale 4 messaggi entro 24 ore.*

* **Costo Meta:** **$0.00** (Sotto la soglia delle 1.000 conversazioni gratuite).
* **Costo Vonage (Inbound):** **$0.00** (Gratis).
* **Costo Vonage (Outbound):** 2 risposte × $0.00819 = **$0.01638**.
* **Totale:** Circa **1.6 centesimi di dollaro**.

### Caso B: Alert automatico al Property Manager

*Scenario: Invio di 1 alert (Template Utility) al Manager. Il Manager risponde con 1 messaggio.*

* **Costo Meta + Vonage (Utility):** **$0.00910** (Costo per singolo messaggio Utility).
* **Costo Vonage (Inbound):** **$0.00** (Gratis).
* **Totale:** Meno di **1 centesimo di dollaro**.

### Caso C: Chatbot "Logorroico" (Errore di loop)

*Scenario: Il bot invia per errore 100 messaggi di risposta a un guest in una sessione Service.*

* **Costo Meta:** **$0.00** (Rientra nella sessione gratuita).
* **Costo Vonage:** 100 messaggi × $0.00819 = **$0.819**.
* **Totale:** **82 centesimi di dollaro**.

---

## Sintesi per il budget mensile (Stima su 100 Guest)

*Dati: 100 Guest (media 10 messaggi/testa = 500 In / 500 Out) + 50 Alert Utility ai manager.*

1. **Canone Numero:** **$1.00**
2. **Meta (Guest - Service):** **$0.00** (Sotto soglia 1.000)
3. **Meta + Vonage (Manager - Utility):** 50 msg × $0.00910 = **$0.45**
4. **Vonage (Outbound Service):** 500 risposte × $0.00819 = **$4.09**
5. **Vonage (Inbound):** 500 messaggi = **$0.00**

* **TOTALE ESTIMATO: ~$5.54 / mese**

---

### Verdetto Finale: Twilio vs Vonage (100 Guest)

* **Twilio:** ~$7.42 / mese
* **Vonage:** ~$5.54 / mese
* **Risparmio:** **~25%** a favore di Vonage.

Scegliendo Vonage, abbatti il costo fisso dei messaggi in entrata, rendendo l'intera operazione molto più economica man mano che aumentano i guest e le interazioni.

