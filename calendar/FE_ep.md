# Specifica API Calendario — per il backender

## 1. `GET /get_properties`

**Quando:** la pagina calendario si apre. Popola il dropdown dove l'host sceglie quale proprietà visualizzare.  
**Body:** nessuno.

**Risposta attesa:**

```json
{
  "data": {
    "properties": [
      { "property_id": "uuid", "name": "Nome proprietà" }
    ]
  }
}
```
2. GET /get_calendar_view?propertyId=<uuid>

Quando: l’host seleziona una proprietà dal dropdown. Richiamato anche automaticamente dopo ogni save/delete ICS.
Body: nessuno. propertyId è un query param nell’URL.

Risposta attesa:
```
[
  {
    "id": "uid-evento",
    "title": "Nome prenotazione",
    "start": "2026-04-07T15:00:00",
    "end": "2026-04-10T11:00:00",
    "allDay": false,
    "description": "...",
    "location": "...",
    "additional_info": {
      "firstName": "Mario",
      "lastName": "Rossi",
      "phone": "+39...",
      "email": "mario@example.com",
      "contractSigned": true,
      "identityVerified": false,
      "checkedIn": false,
      "accessCodeSent": true,
      "notes": "Ospite con cane. Check-in posticipato alle 16."
    }
  }
]
```
additional_info è null se l’host non ha ancora salvato dati per quell’evento.

3. POST /save_ics

Quando: l’host incolla un link ICS nel campo in alto a destra e clicca ✓.

Body:
```
{
  "propertyId": "uuid-proprietà-selezionata",
  "source": "Personalizzato",
  "icsUrl": "https://esempio.com/calendario.ics"
}
```
4. DELETE /delete_ics

Quando: l’host clicca 🗑️ accanto al campo ICS. Rimuove il calendario sincronizzato, ma la proprietà non viene eliminata.

Body:
```
{
  "propertyId": "uuid-proprietà-selezionata",
  "source": "Personalizzato"
}
```
5. POST /save_additional_info

Quando: l’host clicca un evento nel calendario, si apre un pannello laterale, compila dati ospite, checkbox di stato e/o note libere, poi clicca Salva.

Body:
```
{
  "propertyId": "uuid-proprietà-selezionata",
  "uidEvent": "uid-evento-cliccato",
  "additionalInfo": {
    "firstName": "Mario",
    "lastName": "Rossi",
    "phone": "+39...",
    "email": "mario@example.com",
    "contractSigned": true,
    "identityVerified": false,
    "checkedIn": false,
    "accessCodeSent": true,
    "notes": "Ospite con cane. Check-in posticipato alle 16."
  }
}
```
I campi di additionalInfo sono tutti opzionali. Il FE invia solo quelli compilati dall’host.
Il backender deve salvare e ritornare notes insieme agli altri campi dentro additional_info nella risposta di get_calendar_view.
