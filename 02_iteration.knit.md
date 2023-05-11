---
output:
  pdf_document: default
  html_document: default
eval: FALSE
---

# Iteration {-}

Up until this point, we have been able to download and map the range of one plant species at a time. However, what if we want to download  the data for and create a range map of several plant species? This is where iteration comes into play!

Essentially, iteration is a way of running the code you wrote in your function multiple times. So if you ever find yourself copy-pasting, then you could likely benefit from using iteration. 

There are three main ways to do iterations: 

1. For Loops
2. Apply functions
3. Map functions

## Objective

The goal of this mini-workshop is to map multiple Fraxinus species in one plot. In order to do this, I will need to download the data for each Fraxinus species, combine each respective dataset into one dataframe, and then can use this dataframe to plot the range of multiple species. 

## Iteration Examples

First, I will need to write a function to download the data.


```r
# Let's create one datafame to start 
download_data <- function(x){
  GBIFdata = occ_search(scientificName = x, limit = 300, hasCoordinate = TRUE, basisOfRecord = "PRESERVED_SPECIMEN")
dat <-  GBIFdata$data
return(dat)
}

# test
df_FP <- download_data("Fraxinus pallisiae")
```


*For Loops*

Now, let's use this function to download data for other species of Fraxinus. The first method I'll will use is For Loops, which essentially repeats the code you've written across different input values (i). For example, 


```r
for(i in 1:10) {
  print(i)
}
```

We will need to create an empty dataframe to save the results of the loop


```r
results <- rep(NA, 10)

for(i in 1:10) {
  results[i] <- i*10
}
```

Now let's try to translate this to the GBIF example.


```r
# Now let's try bringing in multiple species
## create list of all Fraxinus species
species_names <- read.csv("fraxinus_species.csv") %>% #I scraped this list off the GBIF website
  dplyr::select(Name)%>%
  as.list() 

species_names

## there's a lot so for time sake let's just select a few species
species <- c("Fraxinus albicans Buckley", "Fraxinus americana", "Fraxinus angustifolia", "Fraxinus anomala", "Fraxinus apertisquamifera")

## we need to create an empty dataframe for the loop to fill!
comb_species <- data.frame()

## for loop
for(i in species) {
  df <- download_data(i)
  comb_species <- bind_rows(comb_species, df)
}
## now we get this error that numbers of columns of arguments do not match

### Let's just select columns of interest across datasets: 
colnames(df_FP)
### Columns we want: "scientificName", "decimalLatitude","decimalLongitude"  

## Update function
download_data_updated <- function(x){
  GBIFdata <-  occ_search(scientificName = x, limit = 300, hasCoordinate = TRUE, basisOfRecord = "PRESERVED_SPECIMEN")
  dat <-  GBIFdata$data 

dat <- tryCatch( # this tells R to skip over any dataframes that did not have the columns of interest
  {dat %>%
  filter(scientificName != "NA") %>% #get rid of species name with "NA"
  dplyr::select(scientificName, decimalLatitude, decimalLongitude)}, 
  error=function(error_message) {
            message("Missing column.")
            message("And below is the error message from R:")
            message(error_message)
            return(NA)})
return(dat)
}

## Run for loop again and see if it works
comb_species <- data.frame()
for(i in species) {
  df <- download_data_updated(i)
  comb_species <- bind_rows(comb_species, df)
} 
```

*Apply functions*
Another option for iteration are apply functions in base R. Check out this tutorial for more information: https://ademos.people.uic.edu/Chapter4.html. They often run faster than for loops & can be a lot simpler to set up!



```r
?apply
# just the apply function requires a matrix or dataframe to loop over, but this isn't the format of our input value which is a list...let's look at other options!
?lapply
?sapply

comb_species_apply <- lapply(species, FUN = download_data) #spits out a list!

# how to go from list to dataframe?
# do.call calls the function you specify and runs it on the next argument
comb_species_apply_df <- do.call("bind_rows", comb_species_apply) #this will turn the list into a dataframe
# this also helps with iterating through mulitple files
```

*Map functions*

Apply functions are very handy, but for those dedicated to tidyverse you might be interested in the map family of functions from the purrr package (which is in the tidyverse suite of packages). Check out this tutorial for more: https://jennybc.github.io/purrr-tutorial/


```r
?purrr::map

comb_species_map <- purrr::map(species, .f = download_data)
comb_species_map_df <- do.call("bind_rows", comb_species_apply)
```

For fun, let's see how the timing compares between the for loop and the apply functions. 


```r
## For Loop
system.time({ 
  rm(comb_species)
comb_species <- data.frame()
for(i in species) {
  df <- download_data(i)
  comb_species <- bind_rows(comb_species, df)
}})
 

## Apply
system.time(comb_species_apply <- lapply(species, FUN = download_data))
```

## Final Plot

Now let's make a nice plot!

