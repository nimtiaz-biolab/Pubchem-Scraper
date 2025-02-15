# Pubchem-Scraper

### Load required libraries
```
library(httr)
library(jsonlite)
```

### Sample ligand list (replace this with your actual list of ligands)
```
ligands <- c("glucose", "adenine", "caffeine")  # Replace with your 500 ligands
```

### Initialize an empty data frame to store results
```
smiles_data <- data.frame(Ligand = character(), CID = character(), SMILES = character(), stringsAsFactors = FALSE)
```
### Base URL for PubChem API
```
base_url <- "https://pubchem.ncbi.nlm.nih.gov/rest/pug"
```

### Loop through each ligand
```
for (i in seq_along(ligands)) {
  ligand <- ligands[i]
  message(paste("Processing ligand:", ligand, "(", i, "of", length(ligands), ")"))
  
  # Step 1: Get PubChem CID
  cid_url <- paste0(base_url, "/compound/name/", URLencode(ligand), "/cids/JSON")
  
  cid_response <- tryCatch({
    GET(cid_url)
  }, error = function(e) {
    message("Error fetching CID for ligand:", ligand)
    return(NULL)
  })
  
  # Check if the request was successful and parse the response
  if (!is.null(cid_response) && cid_response$status_code == 200) {
    cid_data <- fromJSON(content(cid_response, "text"), flatten = TRUE)
    
    if (!is.null(cid_data$IdentifierList$CID)) {
      cid <- cid_data$IdentifierList$CID[1]  # Take the first CID if multiple exist
      
      # Step 2: Get SMILES using CID
      smiles_url <- paste0(base_url, "/compound/cid/", cid, "/property/CanonicalSMILES/JSON")
      
      smiles_response <- tryCatch({
        GET(smiles_url)
      }, error = function(e) {
        message("Error fetching SMILES for CID:", cid, " Ligand:", ligand)
        return(NULL)
      })
      
      # Check if the SMILES request was successful and parse the response
      if (!is.null(smiles_response) && smiles_response$status_code == 200) {
        smiles_data_json <- fromJSON(content(smiles_response, "text"), flatten = TRUE)
        smiles <- smiles_data_json$PropertyTable$Properties$CanonicalSMILES
        
        # Store the result
        smiles_data <- rbind(smiles_data, data.frame(Ligand = ligand, CID = cid, SMILES = smiles))
      } else {
        # If SMILES could not be retrieved, store NA
        smiles_data <- rbind(smiles_data, data.frame(Ligand = ligand, CID = cid, SMILES = NA))
      }
      
    } else {
      # If CID could not be found, store NA for CID and SMILES
      smiles_data <- rbind(smiles_data, data.frame(Ligand = ligand, CID = NA, SMILES = NA))
    }
    
  } else {
    # If CID request failed, store NA for CID and SMILES
    smiles_data <- rbind(smiles_data, data.frame(Ligand = ligand, CID = NA, SMILES = NA))
  }
  
  # Rate limiting - add a delay after each request to avoid overwhelming the server
  Sys.sleep(0.2)
  
  # Optional: Longer pause every 100 requests
  if (i %% 100 == 0) {
    message("Pausing for 5 seconds to avoid rate limits...")
    Sys.sleep(5)
  }
}
```
### Display the results
```
view(smiles_data)
```

 
