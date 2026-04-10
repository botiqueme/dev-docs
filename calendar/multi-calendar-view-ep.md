
# Backend Specs — Multi-Calendar Feature

## Contesto

Il frontend ora supporta più sorgenti ICS per ogni proprietà (Booking, Airbnb, Expedia, VRBO, o nome custom).
Ogni sorgente è identificata dalla coppia `(propertyId, source)`.
Queste modifiche richiedono una migration del DB e aggiornamenti su endpoint esistenti + un nuovo endpoint.

---

## 1. Migration DB

### Tabella sorgenti ICS

La chiave primaria cambia da `(propertyId)` a `(propertyId, source)`.

```sql
-- Prima
PRIMARY KEY (propertyId)

-- Dopo
PRIMARY KEY (propertyId, source)
```

Migrare i dati esistenti assegnando `source = "Personalizzato"` alle righe già presenti.

### Tabella enrichment guest

La chiave primaria cambia da `(propertyId, uidEvent)` a `(propertyId, source, uidEvent)`.

```sql
-- Prima
PRIMARY KEY (propertyId, uidEvent)

-- Dopo
PRIMARY KEY (propertyId, source, uidEvent)
```

Migrare i dati esistenti assegnando `source = "Personalizzato"`.

---

## 2. `POST /save_ics` — modifica logica

### Request (invariata)
```json
{
  "propertyId": "uuid",
  "source": "Booking",
  "icsUrl": "https://admin.booking.com/ical/xxx.ics"
}
```

### Comportamento
- Upsert su `(propertyId, source)`: inserisce se non esiste, aggiorna `icsUrl` se esiste già
- **Non tocca** le altre sorgenti della stessa proprietà

### Response (invariata)
```json
{ "status": "ok" }
```

---

## 3. `DELETE /delete_ics` — modifica logica

### Request (invariata)
```json
{
  "propertyId": "uuid",
  "source": "Booking"
}
```

### Comportamento
- Elimina **solo** la riga con `(propertyId, source)` corrispondente
- **Non tocca** le altre sorgenti della stessa proprietà

### Response (invariata)
```json
{ "status": "ok" }
```

---

## 4. `GET /get_calendar_view` — modifica response

### Request (invariata)
```
GET /get_calendar_view?propertyId=uuid
```

### Comportamento (invariato)
- Scarica live tutti i feed ICS della proprietà
- Mergia e parsa gli eventi
- Arricchisce ogni evento con i dati enrichment dal DB (se presenti)

### Response — aggiunta campo `source`

Ogni evento deve includere il campo `source` con il nome della sorgente da cui proviene.

```json
[
  {
    "id": "uid-dall-ics",
    "title": "Prenotazione Rossi",
    "start": "2026-04-10T15:00:00",
    "end": "2026-04-13T10:00:00",
    "allDay": false,
    "source": "Booking",
    "description": "3 notti",
    "location": "Milano, Via Verdi 12",
    "additional_info": {
      "firstName": "Marco",
      "lastName": "Rossi",
      "phone": "+39 333 1234567",
      "email": "marco.rossi@example.com",
      "contractSigned": true,
      "identityVerified": true,
      "checkedIn": false,
      "accessCodeSent": true,
      "notes": "Arriva tardi."
    }
  }
]
```

- `source` deve corrispondere esattamente al valore salvato via `POST /save_ics`
- `additional_info` è `null` o assente se l'host non ha ancora inserito dati guest per quell'evento
- Se nessuna sorgente è configurata per la proprietà: `{ "status": "no_ics_found" }`

---

## 5. `GET /get_ics_sources` — nuovo endpoint

### Request
```
GET /get_ics_sources?propertyId=uuid
```

### Response 200 — sorgenti trovate
```json
[
  { "source": "Booking",  "icsUrl": "https://admin.booking.com/ical/xxx.ics" },
  { "source": "Airbnb",   "icsUrl": "https://www.airbnb.it/calendar/ical/xxx.ics" },
  { "source": "MioNome",  "icsUrl": "https://custom.example.com/calendar.ics" }
]
```

### Response 200 — nessuna sorgente configurata
```json
[]
```

### Response 404 — propertyId non trovato
```json
{ "message": "Property not found" }
```

---

## 6. `POST /save_additional_info` — modifica body

### Request — aggiunta campo `source`

```json
{
  "propertyId": "uuid",
  "uidEvent": "uid-dall-ics",
  "source": "Booking",
  "additionalInfo": {
    "firstName": "Marco",
    "lastName": "Rossi",
    "phone": "+39 333 1234567",
    "email": "marco.rossi@example.com",
    "contractSigned": true,
    "identityVerified": true,
    "checkedIn": false,
    "accessCodeSent": true,
    "notes": "Arriva tardi."
  }
}
```

### Comportamento
- Upsert su `(propertyId, source, uidEvent)`
- Il campo `source` è obbligatorio

### Response (invariata)
```json
{ "status": "ok" }
```

---

## 7. Scheduler refresh feed

Se esiste uno scheduler che rifa il fetch periodico dei feed ICS:
- Deve iterare su **tutte** le sorgenti di ogni proprietà, non solo una
- La query di fetch deve essere `SELECT * FROM ics_sources WHERE propertyId = ?` (restituisce N righe)

---

## Riepilogo

| # | Cosa | Tipo |
|---|---|---|
| 1 | Migration tabella sorgenti ICS — chiave `(propertyId, source)` | Migration DB |
| 2 | Migration tabella enrichment — chiave `(propertyId, source, uidEvent)` | Migration DB |
| 3 | `POST /save_ics` — upsert su `(propertyId, source)` | Modifica logica |
| 4 | `DELETE /delete_ics` — elimina solo la sorgente specificata | Modifica logica |
| 5 | `GET /get_calendar_view` — aggiungere `source` su ogni evento | Modifica response |
| 6 | `GET /get_ics_sources` — nuovo endpoint | Nuovo |
| 7 | `POST /save_additional_info` — aggiungere `source` nel body | Modifica contratto |
| 8 | Scheduler — iterare su tutte le sorgenti | Modifica logica |
