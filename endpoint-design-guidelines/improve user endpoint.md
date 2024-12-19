# Richiesta di Revisione Endpoint `/register`

## Contesto
L'interfaccia utente prevede una pagina con campi da riempire e un pulsante "Salva". Quando l'utente clicca su "Salva", si aspetta che tutti i dati inseriti (compresi quelli del profilo e la password) vengano registrati in un unico passaggio.

## Problema Attuale
L'endpoint `/register` attualmente gestisce solo i parametri:
- `email`
- `password`

I seguenti campi aggiuntivi richiesti in interfaccia non vengono registrati:
- `name`
- `surname`
- `phone_number`
- `company`
- `vat_number`

## Richiesta
Espandere l'endpoint `/register` per accettare e registrare anche i campi aggiuntivi richiesti.

---

## Specifiche di Implementazione

### 1. Modifica dell'Endpoint
- Aggiungere i seguenti parametri al corpo della richiesta (`POST`):
  - `name`
  - `surname`
  - `phone_number`
  - `company`
  - `vat_number`
- Validare ciascun parametro per garantire:
  - Formati corretti (es. numeri di telefono e partita IVA).
  - Obbligatorietà solo per i campi realmente necessari.

### 2. Hash della Password
- Continuare a utilizzare `bcrypt` per hashare la password prima di salvarla nel database.

### 3. Modifica del Modello `User`
- Assicurarsi che il modello utente supporti i nuovi campi:
  - Aggiornare lo schema del database se necessario.

### 4. Risposta
- Restituire una risposta JSON coerente che confermi il successo della registrazione o segnali eventuali errori.

---

## Test

1. Verificare che l'endpoint accetti tutti i parametri richiesti.
2. Assicurarsi che i dati vengano salvati correttamente nel database.
3. Simulare il flusso completo dall'interfaccia, cliccando su "Salva" e confermando che tutti i campi vengano registrati.

---

## Obiettivo Finale
L'utente deve poter inserire tutti i dati richiesti in una sola schermata e cliccare su "Salva", garantendo che tutte le informazioni vengano salvate correttamente in un unico passaggio. 
