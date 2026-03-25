Ecco l'audit completo degli endpoint usati dalla feature calendario.
                                                                                          
  ---                                                                                     
  Audit endpoint — Feature Calendario          
                                                                                          
  Base URL                                                                                
                                                                                          
  VITE_BACKEND_CALENDAR_URL (default: http://localhost:5000/)                             
  Autenticazione: Bearer token iniettato automaticamente dall'interceptor Axios.          
                                                                                          
  ---             
  1. GET /get_calendar_view                                                               
                                                                                          
  Quando: al cambio di proprietà selezionata (attualmente mockat — non ancora chiamato)
                                                                                          
  ┌─────────────────┬────────────────────┐                                                
  │                 │                    │                                                
  ├─────────────────┼────────────────────┤                                                
  │ Query param     │ propertyId: string │
  ├─────────────────┼────────────────────┤
  │ Response attesa │ CalendarEvent[]    │
  └─────────────────┴────────────────────┘                                                
   
  CalendarEvent {                                                                         
    id: string                                                                            
    title: string
    start: string          // "2026-03-25" oppure "2026-03-25T14:00:00Z"                  
    end: string                                                                           
    allDay?: boolean
    description?: string                                                                  
    location?: string
    additional_info?: {                                                                   
      firstName?: string
      lastName?: string
      phone?: string
      email?: string
      contractSigned?: boolean                                                            
      identityVerified?: boolean
      checkedIn?: boolean                                                                 
      accessCodeSent?: boolean                                                            
    }
  }                                                                                       
                  
  ▎ ⚠️  Mock attivo — il frontend usa MOCK_EVENTS_BY_PROPERTY hardcoded. Da sostituire con 
  la chiamata reale.
                                                                                          
  ---             
  2. POST /save_ics
                                                                                          
  Quando: click sul pulsante salva (✓) nella riga sync ICS
                                                                                          
  ┌─────────────────┬────────────────────────────────────────────────────────┐
  │                 │                                                        │            
  ├─────────────────┼────────────────────────────────────────────────────────┤
  │ Body            │ { propertyId: string, source: string, icsUrl: string } │
  ├─────────────────┼────────────────────────────────────────────────────────┤
  │ Response attesa │ qualsiasi truthy (il FE non usa il body)               │            
  └─────────────────┴────────────────────────────────────────────────────────┘            
                                                                                          
  ▎ ⚠️  source è hardcoded a "Personalizzato" — non c'è campo per specificarlo. Da valutare
   se il backend si aspetta un nome piattaforma specifico.
                                                                                          
  ---             
  3. DELETE /delete_ics
                       
  Quando: click sul pulsante elimina (🗑) nella riga sync ICS
                                                                                          
  ┌─────────────────┬────────────────────────────────────────┐
  │                 │                                        │                            
  ├─────────────────┼────────────────────────────────────────┤
  │ Body            │ { propertyId: string, source: string } │
  ├─────────────────┼────────────────────────────────────────┤
  │ Response attesa │ qualsiasi truthy                       │
  └─────────────────┴────────────────────────────────────────┘

  ▎ ⚠️  Il FE elimina per source (non per URL). Il campo icsUrl digitato non viene passato 
  — si usa solo per validare che il campo non sia vuoto prima di chiamare. Se il backend 
  deve ricevere l'URL da eliminare, c'è un disallineamento.                               
                  
  ---
  4. POST /save_additional_info
                               
  Quando: submit del form nel drawer dettaglio evento
                                                                                          
  ┌────────────────┬──────────────────────────────────────────────────────────────────┐
  │                │                                                                  │   
  ├────────────────┼──────────────────────────────────────────────────────────────────┤
  │ Body           │ { propertyId: string, uidEvent: string, additionalInfo: { ... }  │
  │                │ }                                                                │
  ├────────────────┼──────────────────────────────────────────────────────────────────┤   
  │ Response       │ qualsiasi non-undefined (il FE mostra toast success)             │   
  │ attesa         │                                                                  │   
  └────────────────┴──────────────────────────────────────────────────────────────────┘   
                  
  additionalInfo: {
    firstName: string
    lastName: string
    phone: string
    email: string
    contractSigned?: boolean
    identityVerified?: boolean                                                            
    checkedIn?: boolean
    accessCodeSent?: boolean                                                              
  }               

  ▎ ⚠️  Il campo si chiama uidEvent nel body ma il FE passa event.id — verificare che il   
  backend usi lo stesso identificativo (UID ICS vs ID interno).
                                                                                          
  ---             
  5. GET /get_properties (non ancora integrato)
                                                                                          
  Quando: caricamento pagina per popolare il dropdown proprietà
                                                                                          
  ┌────────────────┬──────────────────────────────────────────────────────────────────┐
  │                │                                                                  │   
  ├────────────────┼──────────────────────────────────────────────────────────────────┤
  │ Response       │ Array<{ property_id: string, name: string }> (struttura assunta  │
  │ attesa         │ dal mock)                                                        │
  └────────────────┴──────────────────────────────────────────────────────────────────┘   
   
  ▎ ⚠️  Mock attivo — MOCK_PROPERTIES hardcoded. Da collegare a                            
  PropertiesService.getProperties() già esistente nel progetto.
                                                                                          
  ---             
  Riepilogo criticità
                                                                                          
  ┌─────┬────────────────────────────────────┬────────────────────────────────────────┐
  │  #  │               Issue                │                Impatto                 │   
  ├─────┼────────────────────────────────────┼────────────────────────────────────────┤
  │ 1   │ get_calendar_view è mockat         │ Il calendario non carica dati reali    │
  ├─────┼────────────────────────────────────┼────────────────────────────────────────┤
  │ 2   │ get_properties è mockat            │ Le proprietà non vengono dal backend   │   
  ├─────┼────────────────────────────────────┼────────────────────────────────────────┤   
  │     │ source hardcoded a                 │ Il backend potrebbe non trovare la     │   
  │ 3   │ "Personalizzato" in save/delete    │ sorgente corretta                      │   
  │     │ ICS                                │                                        │
  ├─────┼────────────────────────────────────┼────────────────────────────────────────┤   
  │ 4   │ delete_ics non passa l'URL, solo   │ Se ci sono più URL per la stessa       │
  │     │ la source                          │ source, non si sa quale eliminare      │
  ├─────┼────────────────────────────────────┼────────────────────────────────────────┤   
  │ 5   │ uidEvent = event.id — verificare   │ Potrebbe salvare l'enrichment su un ID │   
  │     │ coerenza con UID ICS               │  sbagliato                             │   
  └─────┴────────────────────────────────────┴────────────────────────────────────────┘   
                  
──
Le chiamate HTTP sono già cablate — i metodi del service esistono e vengono invocati.   
  Cosa funziona e cosa no:                                                                
                                               
  ---                                                                                     
  ✅ Già funzionante (appena il backend è up)                                             
                                                                                          
  Salva ICS (handleSaveIcs)                                                               
  - Chiama realmente POST /save_ics                                                       
  - Passa propertyId (mock per ora), "Personalizzato", e l'URL digitato
  - Toast di errore in caso di fallimento                                                 
                                                                                          
  Elimina ICS (handleDeleteIcs)
  - Chiama realmente DELETE /delete_ics                                                   
  - Passa propertyId e "Personalizzato" come source
                                                                                          
  Salva dati ospite (onSubmit nel drawer)                                                 
  - Chiama realmente POST /save_additional_info
  - Passa propertyId, event.id, e tutti i campi del form (checkboxes + dati ospite)       
  - Toast di successo/errore gestiti
                                                                                          
  ---
  ❌ Bloccanti prima di andare in produzione                                              
                                                                                          
  1. propertyId è mock — finché non si integra get_properties, i propertyId passati al
  backend non saranno quelli reali                                                        
  2. event.id è mock ("evt_1", "evt_2"...) — dovrebbe essere l'UID ICS reale che arriva da
   get_calendar_view                                                                      
  3. source hardcoded — salva e cancella sempre con source "Personalizzato", non c'è modo
  di gestire Airbnb vs Booking come sorgenti separate                                     
                  
  ---                                                                                     
  In pratica      
            
  Se colleghi il backend ora e sostituisci i due mock (proprietà + eventi), tutti e tre i 
  pulsanti funzionano senza toccare altro codice. Il wiring è completo.                   
   
─────
