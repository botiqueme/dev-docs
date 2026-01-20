Ecco la guida aggiornata con il modello di fatturazione **Meta 2026** (passaggio al costo "per messaggio" per i template) e l'integrazione corretta dei costi **Twilio**.

---

# GUIDA COSTI WHATSAPP: MODELLO TWILIO (Aggiornato 2026)

## 1. Concetti Chiave: TWILIO

Twilio agisce come intermediario tecnico. La sua struttura di costo è **lineare** e si basa sul volume puro dei dati transitati.

* **Tariffa "Per Messaggio" ($0.005):** Twilio addebita una commissione fissa su **ogni singolo messaggio** (Inbound e Outbound) che attraversa la sua piattaforma. Non esistono messaggi gratuiti lato Twilio.
* **Canone Numero (Monthly Fee):** Paghi un costo fisso mensile (circa **$1.00 - $2.00**) per mantenere attivo il numero di telefono.
* **Indipendenza dalle Sessioni:** Twilio non applica le logiche di "sessione" di Meta; ogni "nuvoletta" di testo scambiata costa $0.005 di commissione Twilio.

---

## 2. Concetti Chiave: META

Nel 2026, Meta differenzia i costi in base alla categoria del messaggio.

* **Service (Iniziate dall'Utente):**
* **Vantaggio:** Le prime **1.000 conversazioni** (sessioni di 24h) ogni mese sono **gratuite**. Oltre la soglia, si paga una tariffa a sessione.


* **Utility & Marketing (Iniziate dal Business):**
* **Modello Pay-per-Message:** Si paga per **ogni singolo messaggio Template** inviato. Non esiste più la finestra gratuita di 24h per queste categorie.
* **Costo Utility Italia:** Circa **$0.0034** per messaggio consegnato.


* **Link Ufficiali:** [Meta Pricing Docs](https://developers.facebook.com/docs/whatsapp/pricing) | [Twilio WhatsApp Pricing](https://www.twilio.com/en-us/whatsapp/pricing)

---

## 3. Esempi Pratici (Costi per l'Italia)

### Caso A: Il Guest interagisce con il Chatbot

*Scenario: Guest scrive "Password Wi-Fi?". Il bot risponde. Totale 4 messaggi (2 inviati, 2 ricevuti) entro 24 ore.*

* **Costo Meta:** **$0.00** (Rientra nel Free Tier delle 1.000 sessioni Service).
* **Costo Twilio:** $0.005 × 4 messaggi = **$0.02**.
* **Totale:** **2 centesimi di dollaro**.

### Caso B: Alert automatico al Property Manager

*Scenario: Il bot invia 1 alert Utility al Manager. Il Manager risponde con 1 messaggio.*

* **Costo Meta:** **$0.0034** (Tariffa per singolo messaggio Utility).
* **Costo Twilio:** $0.005 × 2 messaggi = **$0.01**.
* **Totale:** Circa **1.3 centesimi di dollaro**.

### Caso C: Chatbot "Logorroico" (Errore di loop)

*Scenario: Il bot invia per errore 100 messaggi di testo libero in 5 minuti a un guest.*

* **Costo Meta:** **$0.00** (È una sessione Service già aperta).
* **Costo Twilio:** $0.005 × 100 messaggi = **$0.50**.
* **Totale:** **50 centesimi di dollaro**.

---

## Sintesi per il budget mensile (Stima su 100 Guest)

*Dati: 100 Guest (media 10 messaggi/testa = 1.000 msg) + 50 Alert Utility ai manager.*

1. **Canone Numero:** **$2.00**
2. **Meta (Guest - Service):** **$0.00** (Sotto soglia 1.000)
3. **Meta (Manager - Utility):** 50 msg × $0.0034 = **$0.17**
4. **Twilio (Messaggi totali):** 1.050 messaggi × $0.005 = **$5.25**

* **TOTALE ESTIMATO: ~$7.42 / mese**

> **Nota Tecnica:** Recentemente, il costo Meta Utility è sceso (si paga il singolo messaggio e non l'intera sessione da 3 centesimi), ma il "peso" maggiore su Twilio rimane la commissione sui messaggi ricevuti.

**Desideri che faccia la stessa revisione millimetrica per la pagina di Vonage?**
