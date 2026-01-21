# WhatsApp Cloud API (Direct Meta) — Guida tecnica “salvaculo” (Chatbot Guest BnB)

> Ambito: integrazione **diretta** con **Meta WhatsApp Cloud API** (senza Twilio/Vonage).  
> Obiettivo: ridurre al minimo rischio di **ban/sospensione** applicando opt-in/out, regola 24h, template corretti, sicurezza webhook, rate control **e hardening degli asset Meta (Account/BM/App/Pagina)**.

> Nota: questa guida è “read & do”. Se un check non è rispettato, **non andare live**.

---

## 1) Fondamenta dell’Account (Il “Chi sei”)
Tag: meta-hardening, account-security, ops

### Perché è critico
Meta non banna solo “l’app”. Spesso limita **il profilo personale admin**.  
Se cade l’admin principale, puoi perdere accesso a **Business Manager, App e asset collegati**.

### Check + actionables
- [ ] **Verifica identità personale (admin principale)**  
  **Azione:** assicurati che il profilo personale dell’admin principale abbia completato la verifica identità (documento valido).  
  **Perché serve:** se Meta limita il profilo, “cade tutto il castello”.

- [ ] **2FA obbligatoria per tutti gli utenti con accesso al BM**  
  **Azione:** abilita 2FA su *tutti* i profili con ruoli nel Business Manager (admin inclusi).  
  **Perché serve:** Meta applica blocchi preventivi per “rischio sicurezza” quando manca 2FA.

- [ ] **Solo profili reali (niente account fake/tecnici)**  
  **Azione:** rimuovi profili creati ad hoc o sospetti; usa persone reali con attività normale.  
  **Perché serve:** Meta correla segnali (IP, attività, rete sociale) e se sembri un bot, banna tutto ciò che tocchi.

---

## 2) Business Manager (Il “contenitore”)
Tag: meta-hardening, business-manager, billing, recovery

### Perché è critico
Il Business Manager è l’entità “aziendale” che Meta decide se è affidabile.  
Un BM verificato ha tipicamente più tolleranza agli errori e meno blocchi automatici.

### Check + actionables
- [ ] **Business Verification completata**  
  **Azione:** avvia e completa la verifica aziendale (documenti fiscali / visura / equivalente).  
  **Perché serve:** aumenta affidabilità e riduce probabilità di limitazioni automatiche.

- [ ] **Backup admin (≥ 2-3 persone reali e fidate)**  
  **Azione:** aggiungi almeno 2-3 admin; evita “single point of failure”.  
  **Perché serve:** se un admin viene limitato, gli altri possono recuperare app/pagina.

- [ ] **Metodo di pagamento valido e stabile**  
  **Azione:** configura un metodo di pagamento business reale (no prepagate usa e getta).  
  **Perché serve:** anomalie pagamenti triggerano blocchi su account pubblicitario e asset collegati.

- [ ] **Dominio verificato**  
  **Azione:** verifica dominio del sito nel BM (Brand Safety → Domains).  
  **Perché serve:** prova proprietà legittima del dominio associato ad app/pagina.

---

## 3) Developer App (Il “codice”)
Tag: meta-hardening, meta-dev-config, app-review

### Perché è critico
Meta penalizza app “vuote” o incoerenti.  
Molti ban/reject arrivano da segnali di spam: campi incompleti, permessi non usati, app “test”.

### Check + actionables
- [ ] **Privacy Policy reale (URL pubblico e raggiungibile)**  
  **Azione:** inserisci URL di una pagina web (non PDF) con policy che menzioni esplicitamente uso dati Meta/WhatsApp.  
  **Cosa deve includere (minimo):**
  - chi è il titolare e contatti
  - quali dati tratti (wa_id, telefono, contenuti messaggi se applicabile)
  - finalità (customer care/servizio)
  - condivisione con Meta (WhatsApp Cloud API)
  - opt-out (STOP) e richiesta cancellazione  
  **Perché serve:** senza policy coerente, l’app viene vista come non affidabile.

- [ ] **App icon + dettagli completi**  
  **Azione:** carica icona 1024×1024 e compila tutti i campi (nome sensato, descrizione, contatti).  
  **Perché serve:** app “vuote”/generiche vengono flaggate come spam.

- [ ] **Niente permessi inutilizzati**  
  **Azione:** richiedi solo permessi che usi davvero nel codice e che puoi dimostrare.  
  **Perché serve:** permessi gratuiti = red flag + review più aggressiva.

- [ ] **App Review video (se richiedi permessi avanzati)**  
  **Azione:** prepara screencast che mostra:
  1) dove l’utente clicca (es. login/consenso)
  2) quali dati prendi
  3) dove li usi nell’app  
  **Perché serve:** se il video non è chiaro, scatta rifiuto o flag per “trasparenza”.

---

## 4) Gestione Pagina / Contenuti (se usi anche asset social Meta)
Tag: meta-hardening, account-quality, content-ops

### Perché è critico
Anche se il progetto è “solo WhatsApp”, account quality e segnali spam su altri asset Meta possono impattare la reputazione complessiva.

### Check + actionables
- [ ] **Evita pattern da spam**  
  **Azione:** niente posting ripetuto identico, niente raffiche di link uguali, niente azioni massicce in poco tempo.  
  **Perché serve:** trigger automatici anti-spam.

- [ ] **Warm-up se asset nuovi**  
  **Azione:** se pagina/BM sono nuovi, evita subito automazioni aggressive.  
  **Perché serve:** account nuovi con attività intensa sembrano bot.

- [ ] **Gestione feedback e segnalazioni**  
  **Azione:** monitora commenti/segnalazioni; se aumentano, riduci attività e indaga.  
  **Perché serve:** segnalazioni utenti fanno crollare Account Quality.

---

## 5) Strategia “Anti-disastro”
Tag: meta-hardening, incident-response, recovery

### Check + actionables
- [ ] **Monitoraggio Account Quality (settimanale)**  
  **Azione:** controlla la sezione Account Quality almeno 1 volta a settimana.  
  **Perché serve:** spesso ci sono warning piccoli prima del blocco totale.

- [ ] **Piano recupero se asset già bannati**  
  **Azione:** se ti hanno già cancellato una pagina/app:
  - considera IP/dominio “sospetti”
  - evita di ricreare tutto identico
  - usa connessione diversa se possibile
  - ricrea su BM nuovo e verificato (coerente con documenti)  
  **Perché serve:** Meta può mantenere reputazione negativa su segnali ricorrenti.

---

## 6) Modello dati + ciclo vita consenso (opt-in / opt-out / soggiorno)
Tag: backend, database, business-logic

### Dati minimi richiesti (per contatto WhatsApp)
Salva **una riga per wa_id** (guest e property manager seguono lo stesso schema).

- `wa_id` (string, unique) — identificativo WhatsApp utente (dal webhook).
- `phone_e164` (string, opzionale) — se ti serve per riconciliazione.
- `consent_status` (enum: `OPTED_IN | OPTED_OUT`)
- `consent_timestamp` (datetime, UTC)
- `consent_source` (enum/string: `booking_form | qr | deep_link | email_link | other`)
- `stay_id` (string, opzionale) — aggancio prenotazione/soggiorno.
- `checkout_at` (datetime, opzionale)
- `last_inbound_at` (datetime, UTC) — timestamp dell’ultimo messaggio ricevuto da quel wa_id.
- `conversation_state` (enum: `BOT | MANUAL | SUSPENDED`) — per controllo handover.
- `blocked_reason` (enum/string, opzionale: `STOP | CHECKOUT_EXPIRED | ABUSE | OTHER`)

### Regole consenso (obbligatorie)
- Non inviare **mai** messaggi proattivi se `consent_status != OPTED_IN`.
- **Stop definitivo** su parole chiave opt-out: `STOP`, `BASTA`, `FINE`, `UNSUBSCRIBE` (case-insensitive).
- **Scadenza post check-out**: dopo `checkout_at + 24h`, porta il contatto in `OPTED_OUT` (o `conversation_state=SUSPENDED`) per evitare invii fuori contesto.

### Handler opt-out (comportamento minimo richiesto)
Se l’inbound matcha opt-out:
1) Set `consent_status=OPTED_OUT` e `blocked_reason=STOP`.
2) Invia **un solo** messaggio di conferma disattivazione (se finestra 24h aperta → free-text; altrimenti → template).
3) Blocca a livello applicativo qualsiasi outbound futuro verso quel `wa_id` (anche se “forzato” da UI staff).

---

## 7) Finestra customer care 24 ore + guardrail UI (staff console)
Tag: backend, manager-ui, message-routing

### Regola
Se sono passate **> 24 ore** dall’ultimo messaggio ricevuto dall’utente, i messaggi “free-form” falliscono e devi usare un **template** (re-ingaggio).

### Gate obbligatorio (prima di ogni invio outbound)
- Se `now - last_inbound_at <= 24h`: consenti free-text (`type: "text"`)
- Se `now - last_inbound_at > 24h`: **blocca free-text** e consenti solo `type: "template"`

### Requisito UI (staff console)
Mostra stato:
- “Finestra APERTA” se `<=24h` → textarea abilitata
- “Finestra CHIUSA” se `>24h` → textarea disabilitata + CTA “Invia template per riaprire”

### Failure handling (must-have)
Se ricevi errore “window closed/out of window”:
- marca conversazione `WINDOW_CLOSED`
- forza template-only
- notifica lo staff in UI (“non puoi rispondere liberamente”)

---

## 8) Webhook: verifica (GET challenge) + validazione firma (POST)
Tag: backend, security, meta-dev-config

### 8.1 Verifica webhook (GET challenge)
L’endpoint deve gestire la verifica:
- query params: `hub.mode`, `hub.verify_token`, `hub.challenge`
- se `hub.verify_token` combacia col token configurato → rispondi con `hub.challenge` (plain text)

$$$javascript
// Express example
app.get("/webhook", (req, res) => {
  const mode = req.query["hub.mode"];
  const token = req.query["hub.verify_token"];
  const challenge = req.query["hub.challenge"];

  if (mode === "subscribe" && token === process.env.META_VERIFY_TOKEN) {
    return res.status(200).send(challenge);
  }
  return res.sendStatus(403);
});
$$$

### 8.2 Validazione firma (X-Hub-Signature-256) — OBBLIGATORIA
Valida che i POST arrivino davvero da Meta con HMAC-SHA256 su **raw body** e header `X-Hub-Signature-256`.

> Nota: la chiave HMAC è la **Meta App Secret** (non il verify token).

#### Express: preserva raw body
$$$javascript
import express from "express";
import crypto from "crypto";

const app = express();

// Serve il raw body per calcolare l'HMAC
app.use(express.json({
  verify: (req, res, buf) => {
    req.rawBody = buf; // Buffer
  }
}));

function verifyMetaSignature(req) {
  const header = req.get("x-hub-signature-256");
  if (!header) throw new Error("Missing X-Hub-Signature-256");

  const [algo, signature] = header.split("=");
  if (algo !== "sha256" || !signature) throw new Error("Invalid signature header");

  const expected = crypto
    .createHmac("sha256", process.env.META_APP_SECRET)
    .update(req.rawBody)
    .digest("hex");

  // confronto a tempo costante
  const ok = crypto.timingSafeEqual(Buffer.from(signature, "hex"), Buffer.from(expected, "hex"));
  if (!ok) throw new Error("Invalid signature");
}

app.post("/webhook", (req, res) => {
  try {
    verifyMetaSignature(req);
  } catch (e) {
    return res.sendStatus(401);
  }

  // TODO: processa webhook
  return res.sendStatus(200);
});
$$$

---

## 9) Rate limiting + backoff + anti-loop (prevenzione ban)
Tag: backend, devops, chatbot-logic

### Controlli obbligatori
- **Limiter per destinatario (wa_id)**: max `N` outbound per `wa_id` per finestra (es. `5 / 2 minuti`).
- **Limiter globale**: cap outbound al secondo (sotto i limiti del tier).
- **Exponential backoff** su errori di rate limit / throughput:
  - delay: 1s, 2s, 4s, 8s… (cap a 60s)
  - stop retry dopo pochi tentativi e genera alert

$$$javascript
async function sendWithBackoff(sendFn, maxRetries = 5) {
  let delayMs = 1000;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await sendFn();
    } catch (err) {
      const isRateLimit = err?.code === 130429 || err?.metaCode === 130429;
      if (!isRateLimit || attempt === maxRetries) throw err;
      await new Promise(r => setTimeout(r, delayMs));
      delayMs = Math.min(delayMs * 2, 60000);
    }
  }
}
$$$

### Anti-loop kill switch (must-have per chatbot)
Se il bot va in loop (prompt, retry, stato corrotto, replay):
- se `outbound_count(wa_id, 2min) > 5`:
  - set `conversation_state=MANUAL` (o `SUSPENDED`)
  - notifica staff
  - interrompi risposte automatiche per quel thread

---

## 10) Template obbligatori (UTILITY) per i tuoi flussi
Tag: meta-dev-config, backend, content

Servono template per:
- ogni messaggio business-initiated
- ogni re-ingaggio dopo finestra 24h chiusa

### 10.1 Template benvenuto guest (UTILITY)
Nome: `guest_welcome_utility`  
Categoria: `UTILITY`  
Lingua: `it` (+ copia `en`)

Body (no promo, include opt-out):
- `Ciao {{1}}! 👋 Sono l'assistente virtuale di {{2}}.`
- `Scrivimi per: check-in, Wi-Fi, regole casa, orari.`
- `Se non vuoi più ricevere messaggi, rispondi STOP.`

Quick replies consigliati:
- `Istruzioni check-in`
- `Info Wi-Fi`
- `Parla con lo staff`

### 10.2 Template alert staff / property manager (UTILITY)
Nome: `staff_alert_utility`  
Categoria: `UTILITY`

Body:
- `⚠️ Intervento richiesto`
- `Guest: {{1}}`
- `Struttura: {{2}}`
- `Ultimo messaggio: {{3}}`

---

## 11) Payload invio messaggi (text / template / media)
Tag: backend, api-integration

> Sostituisci `GRAPH_VERSION`, `PHONE_NUMBER_ID` e token.

### 11.1 Free-text (solo finestra aperta)
$$$json
{
  "messaging_product": "whatsapp",
  "to": "<WA_ID_OR_PHONE_E164>",
  "type": "text",
  "text": { "body": "Ciao! Come posso aiutarti?" }
}
$$$

### 11.2 Template (obbligatorio se finestra chiusa)
$$$json
{
  "messaging_product": "whatsapp",
  "to": "<WA_ID_OR_PHONE_E164>",
  "type": "template",
  "template": {
    "name": "guest_welcome_utility",
    "language": { "code": "it" },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Mario" },
          { "type": "text", "text": "B&B Bella Vista" }
        ]
      }
    ]
  }
}
$$$

### 11.3 Media: upload + invio PDF
Per inviare documenti (mappe, guide), carica prima il file e usa il `media_id`.

**Upload**
$$$bash
curl -X POST "https://graph.facebook.com/<GRAPH_VERSION>/<PHONE_NUMBER_ID>/media" \
  -H "Authorization: Bearer <META_ACCESS_TOKEN>" \
  -F "messaging_product=whatsapp" \
  -F "file=@./Guida_Checkin.pdf;type=application/pdf"
$$$

**Invio documento**
$$$json
{
  "messaging_product": "whatsapp",
  "to": "<WA_ID_OR_PHONE_E164>",
  "type": "document",
  "document": {
    "id": "<MEDIA_ID>",
    "filename": "Guida_Checkin.pdf"
  }
}
$$$

---

## 12) Vincoli prompt + guardrail backend (sicurezza e policy)
Tag: chatbot-prompt, backend-guardrails, manager-ui

### Vincoli obbligatori (prompt + code)
- Identificati sempre come assistente virtuale nel primo messaggio della sessione.
- Non chiedere mai:
  - dati carta / pagamento
  - foto documenti
  - “mandami passaporto/ID” via WhatsApp
- Se l’utente chiede cose sensibili → instrada a portale sicuro o staff.
- Escalation immediata a staff se:
  - chiede “operatore / umano”
  - linguaggio aggressivo/minacce
  - fallimenti ripetuti nella risposta

> Nota: non fidarti solo del prompt. Implementa regole backend che forzano `conversation_state=MANUAL` su trigger.

---

## 13) Checklist configurazione Meta (cose da impostare correttamente)
Tag: meta-dev-config, security, legal

### Richiesto
- Business verification completata (o accetti limiti/rischio).
- Display Name coerente con brand/sito.
- Privacy Policy URL configurata e pubblica.
- Webhook su HTTPS con certificato valido (no self-signed).
- Verify token impostato e custodito.
- 2FA abilitata sugli admin (hardening).
- Metodo di pagamento valido e stabile (billing hardening).
- Dominio verificato nel Business Manager.
- Monitoraggio Account Quality (settimanale).

---

## 14) Monitoring + “panic switch”
Tag: devops, backend, monitoring

### Da monitorare (minimo)
- error rate outbound (raggruppa per codice errore; evidenzia window/rate-limit)
- burst detection per utente (anti-loop)
- conteggio handover (quanto spesso vai in MANUAL)
- tasso opt-out (spike = rischio)
- warning Account Quality (early signals)

### Panic switch (obbligatorio)
Flag di config (es. `WHATSAPP_OUTBOUND_ENABLED=false`) che:
- blocca subito gli invii proattivi
- lascia attivi inbound + alert staff su canali alternativi

---

## 15) Definition of Done (accettazione)
Tag: backend, security, testing, meta-dev-config

- [ ] Hardening account: 2FA su tutti gli admin + profili reali (no fake).
- [ ] Business verification completata + dominio verificato + billing stabile.
- [ ] Privacy Policy pubblica e coerente (menziona dati Meta/WhatsApp).
- [ ] Webhook GET verify OK (ritorna `hub.challenge` se token ok).
- [ ] Webhook POST firma valida (rifiuta firme non valide).
- [ ] `last_inbound_at` salvato ed enforcement 24h attivo.
- [ ] Free-text bloccato fuori finestra; template-only enforced.
- [ ] Opt-out disabilita outbound per quel wa_id (hard block).
- [ ] Check-out expiry blocca outbound dopo `checkout+24h`.
- [ ] Limiter per wa_id + limiter globale attivi.
- [ ] Backoff esponenziale su rate limit / throughput.
- [ ] Anti-loop kill switch → MANUAL + alert staff.
- [ ] 2 template UTILITY creati e approvati (welcome + staff alert).
- [ ] Upload media + invio PDF testati.
- [ ] Prompt vincolato + escalation enforced in backend.
- [ ] Panic switch testato in staging.
