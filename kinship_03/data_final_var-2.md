Create Final Dataset for Analysis - 03 - 2
================

``` r
library(data.table)
library(dplyr)

# import parent ID + gender data
cleanUKIreland <- fread(file = params$path_data, na.strings='', encoding="UTF-8")
```

## Plan of attack:

1)  DETERMINE FINAL VARIABLES & CLEAN
2)  MERGE \# OF SIBLINGS
3)  MERGE BIRTH ORDER
4)  CREATE “FAMILY ID” USING EITHER MOTHER OR FATHER’S ID
5)  DESCRIPTIVE STATISTICS OF SAMPLE
6)  ESTIMATE REGRESSIONS

# STEP 1 - DETERMINE FINAL VARIABLES

## outcome: age at death

## independent variable: migrant status

## covariates: gender, death country (FE), birth cohort, \# of siblings, family ID (RE)

## potentially, death cohort? to get at period effects of mortality. or is that over controlling?

## also, birth order dummy (or first born son?)

## mother’s age at birth or average birth interval?? SES proxy

## try to proxy when one migrated by looking at where first child was born

``` r
finalvar <- cleanUKIreland %>% 
  rename(ego = profileid, deathage = age, birth_country = birth_location_country,
         death_country = death_location_country) %>% 
  mutate( 
    # 5 year birth cohort
    birth5 = cut(as.numeric(birth_year),
                 breaks = seq(1735, 1896, by = 5),
                 labels = seq(1735, 1890, by = 5), include.lowest = T, right = T), 
    # 10 year birth cohort
    birth10 = cut(as.numeric(birth_year),
                  breaks = seq(1735, 1896, by = 10),
                  labels = seq(1735, 1885, by = 10), include.lowest = T, right = T),
    # 5 year birth cohort
    death5 = cut(as.numeric(death_year),
                 breaks = seq(1735, 2000, by = 5),
                 labels = seq(1735, 1995, by = 5), right = F),
    # 10 year birth cohort
    death10 = cut(as.numeric(birth_year), 
                  breaks = seq(1735, 2000, by = 10),
                  labels = seq(1735, 1990, by = 10), right = F),
    migrant = if_else(birth_country != death_country, T, F),
    migrant = if_else(birth_country == "United Kingdom" & death_country == "Ireland",
                      0, migrant), #nonmigrants for our purposes
    migrant = if_else(birth_country == "Ireland" & death_country == "United Kingdom",
                      0, migrant))
rm(cleanUKIreland)
```

## STEP 2 - NUMBER OF SIBLINGS

``` r
sib.num <- fread(params$sib.num)

finalvar <- finalvar %>% left_join(sib.num, by = "ego")
rm(sib.num)
```

## STEP 3 - BIRTH ORDER

``` r
sib.or <- fread(params$sib.or)

sib.order <- sib.or %>% 
  select(-sib.ct) %>% 
  mutate(firstborn = if_else(birth_order == 1, 1, 0))

rm(sib.or)

finalvar <- finalvar %>% 
  left_join(sib.order, by = "ego") %>% 
  mutate(
    firstmale = if_else((firstborn == 1 & gender == "male"), 1, 0),
    firstmale = if_else(is.na(firstborn), NA, firstmale)
    )
rm(sib.order)
```

## STEP 4 - FAMILY ID

``` r
mp.df <- fread(params$mp.df)
mp.df <- mp.df %>% rename(ego = ch.id) %>% select(ego, mom, pop)

finalvars <- finalvar %>% left_join(mp.df, by = "ego") %>% 
  rename(famid = mom) %>% 
  mutate(famid = ifelse(is.na(famid), pop, famid)) %>% 
  select(-pop)

rm(mp.df, finalvar)

sourceCountries <- c("Ireland", "United Kingdom")

finalvars <- finalvars %>% 
  filter(gender != "",
         deathage >= 15 & deathage <= 110) %>% 
  mutate(birth_country = case_when(birth_country %in% sourceCountries ~ "UK/Ireland",
                                   T ~ birth_country),
         death_country = case_when(death_country %in% sourceCountries ~ "UK/Ireland",
                                   T ~ death_country),
         death_country = factor(death_country, 
                                levels = c("UK/Ireland", "Canada", "South Africa",
                                           "Australia", "New Zealand",
                                           "United States of America"))) 

# recode sib count and birth order variables
# missing different than 0 recorded siblings
final.fixed <- finalvars %>% 
  mutate(sib.ct = if_else(is.na(famid), -1, sib.ct)) %>% 
  mutate(sib_size_cat = case_when(sib.ct == 0 ~ '0',
                                  sib.ct == 1 ~ '1',
                                  sib.ct == 2 ~ '2',
                                  sib.ct >= 3 & sib.ct <= 5 ~ '3-5',
                                  sib.ct >= 6 ~ '6+',
                                  TRUE ~ 'missing')) %>% 
  mutate(sib_size_cat = factor(sib_size_cat,
                               levels = c('missing', '0', '1', '2', '3-5', '6+')))

final.fixed1 <- final.fixed %>% 
  mutate(birth_order = if_else(sib.ct == -1, -1, birth_order),
         firstborn = case_when(birth_order >= 0 & birth_order <= 1 ~ '1',
                               birth_order >= 2 ~ '0',
                               TRUE ~ 'missing'),
         firstborn = factor(firstborn,
                            levels = c('missing', '0', '1')))


final.fixed1 <- final.fixed1 %>% 
  select(-birth5, -death5, -firstmale)
rm(final.fixed)

fvs.nona <- finalvars %>% filter(!is.na(famid))

# Filter years - NOTE: still some NAs due to egos being only parent and not child
finalvars <- final.fixed1 %>% filter(birth_year > 1734 & birth_year < 1896)
```

## Write final data to CSV for analysis

``` r
fwrite(finalvars, params$save_path)
```
