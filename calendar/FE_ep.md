⏺ Specifica API Calendario — per il backender                                                                                     
                                                                                                                                  
  ---                                                                                                                                    
  1. GET /get_properties                                                                                                                 
                                                                                                                                         
  Quando: la pagina calendario si apre — serve a popolare il dropdown in alto a sinistra dove l'host sceglie quale proprietà             
  visualizzare.                                                                                                                          
  Nessun body.
  Risposta attesa:                                                                                                                       
  {                                                            
    "data": {                                                                                                                            
      "properties": [                                          
        { "property_id": "uuid", "name": "Nome proprietà" }                                                                              
      ]                                                    
    }                                                                                                                                    
  }                                                            
                                                                                                                                         
  ---
  2. GET /get_calendar_view?propertyId=<uuid>                                                                                            
                                                               
  Quando: l'host seleziona una proprietà dal dropdown. Anche richiamato automaticamente dopo ogni salvataggio o cancellazione ICS.
  Nessun body — propertyId è un query param nell'URL.                                                                                    
  Risposta attesa:                                                                                                                       
  [                                                                                                                                      
    {                                                                                                                                    
      "id": "uid-evento",                                      
      "title": "Nome prenotazione",                                                                                                      
      "start": "2026-04-07T15:00:00",                          
      "end": "2026-04-10T11:00:00",                                                                                                      
      "allDay": false,             
      "description": "...",                                                                                                              
      "location": "...",                                       
      "additional_info": null                                                                                                            
    }                                                          
  ]                                                                                                                                      
   
  ---                                                                                                                                    
  3. POST /save_ics                                            
                   
  Quando: l'host incolla un link ICS nel campo in alto a destra e clicca ✓.
  Body:                                                                                                                                  
  {      
    "propertyId": "uuid-proprietà-selezionata",                                                                                          
    "source": "Personalizzato",                                                                                                          
    "icsUrl": "https://esempio.com/calendario.ics"
  }                                                                                                                                      
                                                               
  ---                                                                                                                                    
  4. DELETE /delete_ics                                        
                                                                                                                                         
  Quando: l'host clicca 🗑️  accanto al campo ICS. Rimuove il calendario sincronizzato dalla proprietà — la proprietà NON viene eliminata.
  Body:                                                                                                                                  
  {                                                            
    "propertyId": "uuid-proprietà-selezionata",                                                                                          
    "source": "Personalizzato"                                 
  }

  ---
  5. POST /save_additional_info
                                                                                                                                         
  Quando: l'host clicca su un evento nel calendario → si apre un pannello laterale → compila i dati dell'ospite e le checkbox → clicca
  Salva.                                                                                                                                 
  Body:                                                        
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
      "accessCodeSent": true                                                                                                             
    }                                                          
  }                                                                                                                                      
       
