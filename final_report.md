combined\_dataset
================

``` r
UHF_zipcode = 
  read_csv("./data/appb.csv") %>% 
  slice(-43) %>% 
  select(-Borough) %>% 
  rename("UHF" = "UHF Neighborhood") %>% 
  janitor::clean_names()
```

    ## Parsed with column specification:
    ## cols(
    ##   Borough = col_character(),
    ##   `UHF Neighborhood` = col_character(),
    ##   `Zip Code` = col_character()
    ## )

``` r
raw_hiv = 
  GET("https://data.cityofnewyork.us/api/views/fju2-rdad/rows.csv") %>% 
  content("parsed") %>% 
  janitor::clean_names()
```

    ## Parsed with column specification:
    ## cols(
    ##   Year = col_integer(),
    ##   Borough = col_character(),
    ##   UHF = col_character(),
    ##   Gender = col_character(),
    ##   Age = col_character(),
    ##   Race = col_character(),
    ##   `HIV diagnoses` = col_integer(),
    ##   `HIV diagnosis rate` = col_double(),
    ##   `Concurrent diagnoses` = col_integer(),
    ##   `% linked to care within 3 months` = col_integer(),
    ##   `AIDS diagnoses` = col_integer(),
    ##   `AIDS diagnosis rate` = col_double(),
    ##   `PLWDHI prevalence` = col_double(),
    ##   `% viral suppression` = col_integer(),
    ##   Deaths = col_integer(),
    ##   `Death rate` = col_double(),
    ##   `HIV-related death rate` = col_double(),
    ##   `Non-HIV-related death rate` = col_double()
    ## )

``` r
combine_hiv = 
  right_join(UHF_zipcode, raw_hiv, by = "uhf") %>%
  janitor::clean_names() %>% 
  separate(zip_code, into = c("zipcode1", "zipcode2", "zipcode3", 
                              "zipcode4", "zipcode5", "zipcode6", "zipcode7", "zipcode8",
                              "zipcode9"), sep = ", ") %>% 
  gather(key = zip_code, value = zipcode_value, zipcode1:zipcode9) %>% 
  filter(!is.na(zipcode_value)) %>% 
  rename("zipcode" = "zipcode_value") %>% 
  select(zipcode, everything(), -zip_code)
```

``` r
r = GET('http://data.beta.nyc//dataset/3bf5fb73-edb5-4b05-bb29-7c95f4a727fc/resource/6df127b1-6d04-4bb7-b983-07402a2c3f90/download/f4129d9aa6dd4281bc98d0f701629b76nyczipcodetabulationareas.geojson')
nyc_zipcode = readOGR(content(r,'text'), 'OGRGeoJSON', verbose = F)
```

    ## No encoding supplied: defaulting to UTF-8.

``` r
zipcode_lat_lng = nyc_zipcode@data %>% 
  select(zipcode = postalCode, longitude, latitude) %>% 
  mutate(zipcode = as.character(zipcode))
  
combine_hiv1 = 
  full_join(zipcode_lat_lng, combine_hiv, by = "zipcode") %>% 
  mutate(longitude = as.numeric(longitude), latitude = as.numeric(latitude)) %>% 
  group_by(uhf) %>% 
  summarise(lng = mean(longitude),
            lat = mean(latitude)) %>% 
  filter(!(uhf == "Pelhem - Throgs Neck"))
```

``` r
pums_raw = read_csv("./data/selected_pums.csv")
```

    ## Warning: Missing column names filled in: 'X1' [1]

    ## Parsed with column specification:
    ## cols(
    ##   X1 = col_integer(),
    ##   PUMA00 = col_integer(),
    ##   PUMA10 = col_integer(),
    ##   ADJINC = col_integer(),
    ##   PINCP = col_integer(),
    ##   RAC3P05 = col_integer(),
    ##   RAC3P12 = col_integer()
    ## )

``` r
temp = tempfile(fileext = ".xls")

dataURL = "http://faculty.baruch.cuny.edu/geoportal/resources/nyc_geog/nyc_zcta10_to_puma10.xls"

download.file(dataURL, destfile = temp, mode = 'wb')

zcta_to_puma = readxl::read_xls(temp, sheet = 2) %>% 
  select(zcta = zcta10, puma = puma10) %>% 
  mutate(puma = as.numeric(puma))

dataURL =
  "http://faculty.baruch.cuny.edu/geoportal/resources/nyc_geog/zip_to_zcta10_nyc_revised.xls"

download.file(dataURL, destfile = temp, mode = 'wb')
zip_to_zcta = readxl::read_xls(temp, sheet = 2) %>% 
  select(zipcode, zcta = zcta5) 
```

``` r
pums_data = 
  pums_raw %>% 
  select(puma = PUMA10, income = PINCP, year = ADJINC) %>% 
  filter(puma != -9)  # remove data from 2011 due to lack of area information

pums_data$year = recode(pums_data$year, 
                        "1042852" = "2012",
                        "1025215" = "2013",  
                        "1009585" = "2014", 
                        "1001264" = "2015")   

pums_data = 
  pums_data %>% 
  group_by(year, puma) %>% 
  summarise(mid_income = median(income, na.rm = TRUE)) %>% 
  ungroup()         # calculate yearly median income for each area
```

``` r
puma_to_zipcode = right_join(zip_to_zcta, zcta_to_puma, by = "zcta") %>%   # generaate a puma to zipcode file
  select(puma, zipcode)

income_zipcode = right_join(pums_data, puma_to_zipcode, by = "puma") %>%  # matching zipcode with median income data
  select(year, zipcode, mid_income) %>% 
  mutate(year = as.numeric(year))

combine_hiv_income = 
  left_join(combine_hiv, income_zipcode, by = c("year", "zipcode"))
```