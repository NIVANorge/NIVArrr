## Cleaning latin names with errors    

This is function for cleaning species names. It utilizes the search service of Artsdatabanken (Norwegian Biodiversity Information Centre)
and returns the correct name, as well as the Artsdatabanken ID.   

![code example](pics/adbsearch_species.JPG)

### Usage    

#### 1. Load the function below (copy into the R console and push enter). 

```
adbsearch_species <- function(txt){
  txt2 <- gsub(" ", "%20", txt, fixed = TRUE)
  urlstring <- paste0("http://webtjenester.artsdatabanken.no/Artsnavnebase.asmx/Artssok?Search=", txt2)
  conn <- url(urlstring)
  page <- readLines(conn,  encoding = "UTF-8", warn = FALSE)
  close(conn)
  parsed <- xmlTreeParse(page)
  doc <- xmlRoot(parsed)
  if (length(doc) >= 1){
    cat <- xmlAttrs(doc[[1]])[["Kategori"]]
    status <- xmlAttrs(doc[[1]])[["Hovedstatus"]]
    if (cat == "Art" & status == "Gyldig"){
      df <- data.frame(OriginaltNavn = txt, 
                       LatinskNavnID = xmlAttrs(doc[[1]])[["LatinskNavnID"]],
                       VitenskapligNavn = xmlValue(doc[[1]][["VitenskapligNavn"]]), stringsAsFactors = FALSE)
    } else if (cat == "Art" & status == "Synonym"){
      obj <- doc[[1]][["Takson"]][["LatinskNavn"]]
      df <- data.frame(OriginaltNavn = txt, 
                       LatinskNavnID = xmlAttrs(obj)[["LatinskNavnID"]],
                       VitenskapligNavn = xmlValue(obj[["VitenskapligNavn"]]), stringsAsFactors = FALSE)
    } else {
      df <- data.frame(OriginaltNavn = txt, 
                       LatinskNavnID = xmlAttrs(doc[[1]])[["LatinskNavnID"]],
                       VitenskapligNavn = xmlValue(doc[[1]][[cat]]), stringsAsFactors = FALSE)
    }
  } else {
    df <- data.frame(OriginaltNavn = txt, LatinskNavnID = NA, VitenskapligNavn = NA, stringsAsFactors = FALSE)
  }
  df
  }
```
  
#### 2. Run for a single name  

```
adbsearch_species("Hydropsyche nevae")
```
This returns the following small data frame (one row):
```
      OriginaltNavn LatinskNavnID  VitenskapligNavn
1 Hydropsyche nevae        132242 Hydropsyche newae
```

#### 3. Run for a lot of names (a column in a data frame)  
We use 'map_dfr' from purrr package to run it for several names   


```
# We make a data set
test <- data.frame(Species = c("Hydropsyche nevae", "Cheumatopsyche nevae", "Oniscoidea", "Galba truncatula"))

# Necessary packages
library(dplyr)
library(purrr)

# Run 'adbsearch_species' for all and combine to a data frame:  
result <- test$Species %>% map_dfr(adbsearch_species) 
```

The result is:
```
> result
         OriginaltNavn LatinskNavnID  VitenskapligNavn
1    Hydropsyche nevae        132242 Hydropsyche newae
2 Cheumatopsyche nevae          <NA>              <NA>
3           Oniscoidea        138354        Oniscoidea
4     Galba truncatula        121438  Galba truncatula
```

#### 4. Starting with data (each species occurs several times)   

If we use method (3) above, this may take a long time as the function is performed several times for every species name. Instead we
extract the unique species names from the data, use the method in (3), and then add the corrected species names back to the data.  

```
# We make a data set
test_data <- data.frame(
  Species = c("Hydropsyche nevae", "Hydropsyche nevae", "Hydropsyche nevae", "Galba truncatula", "Galba truncatula"),
  Number = c(4,7,6,10,4)
  )
  
# We need only the unique names for each species
unique_species <- unique(test_data$Species)

# Necessary packages
library(dplyr)
library(purrr)

# Run 'adbsearch_species' for all and combine to a data frame:  
corrected_species <- unique_species %>% map_dfr(adbsearch_species) 

# Add the new names back to the original data
# (because the original names are called 'Species' in test_data and 'OriginaltNavn' in corrected_species, we need to specify this
#  with  'by = ')
test_data <- test_data %>%
  left_join(corrected_species, by = c("Species" = "OriginaltNavn"))

```
The `test_data` now contains the additonal names `LatinskNavnID` and `VitenskapligNavn`    

  
  
