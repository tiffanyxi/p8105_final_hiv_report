income\_data\_handling
================

``` r
library(tidyverse)
```

    ## -- Attaching packages --------------------------------------------------------------------------------- tidyverse 1.2.1 --

    ## √ ggplot2 3.0.0     √ purrr   0.2.5
    ## √ tibble  1.4.2     √ dplyr   0.7.7
    ## √ tidyr   0.8.1     √ stringr 1.3.1
    ## √ readr   1.1.1     √ forcats 0.3.0

    ## -- Conflicts ------------------------------------------------------------------------------------ tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
id1 = "1qGkhF7qXf_Qa8YTePLgqvEYfVLWX1tFs"
pums_raw = read_csv(sprintf("https://docs.google.com/uc?id=%s&export=download", id1))  
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
dataURL <- "http://faculty.baruch.cuny.edu/geoportal/resources/nyc_geog/nyc_zcta10_to_puma10.xls"
download.file(dataURL, destfile=temp, mode='wb')

zcta_to_puma = readxl::read_xls(temp, sheet = 2)%>% 
  select(zcta = zcta10, puma = puma10) %>% 
  mutate(puma = as.numeric(puma))

dataURL <- "http://faculty.baruch.cuny.edu/geoportal/resources/nyc_geog/zip_to_zcta10_nyc_revised.xls"
download.file(dataURL, destfile=temp, mode='wb')
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
  summarise(mid_income = median(income, na.rm = TRUE))  # calculate yearly median income for each area
```

``` r
puma_to_zipcode = right_join(zip_to_zcta, zcta_to_puma, by = "zcta") %>%   # generaate a puma to zipcode file
  select(puma, zipcode)

income_zipcode = right_join(pums_data, puma_to_zipcode, by = "puma") %>%  # matching zipcode with median income data
  select(year, zipcode, mid_income)
```

Data Source description
-----------------------

Here, we use the data from 2011-2015 American Community Survey (ACS) Public Use Microdata Sample (PUMS). The dataset was selected based on our interested variables - location and totaly income. The location data from 2011 ACS is based on Public use microdata area code (PUMA) 2000, while the definition for PUMA 2000 is nowhere to be found. This is why we exclude the data from 2011.

Column names meaning in PUMS
----------------------------

This data is selected based on the variables that we may need

PUMA00 --
Public use microdata area code 2000, which is the area code used before 2012.
-9 means this classifications is N/A for data collected after 2012.

PUMA10 --
Public use microdata area code 2010, which is the area code used after 2012.
-9 means this classifications is N/A for data collected prior to 2012.

ADJINC --
Adjustment factor for income and earnings dollar amounts
1073094 -- 2011 factor
1042852 -- 2012 factor
1025215 -- 2013 factor
1009585 -- 2014 factor
1001264 -- 2015 factor

PINCP --
Total persons's income

RAC3P05 --
Recoded detailed race code for data collected prior to 2012

RAC3P12 --
Recoded detailed race code for data collected in 2012 or later

For more info, you can check the data dictionarty "<https://www2.census.gov/programs-surveys/acs/tech_docs/pums/data_dict/PUMS_Data_Dictionary_2011-2015.pdf?#>"

Data from The ACS Public Use Microdata Sample files (PUMS), u can find it here "<https://factfinder.census.gov/faces/tableservices/jsf/pages/productview.xhtml?pid=ACS_pums_csv_2011_2015&prodType=document>"

This is the ZCTA10 to PUMA10 file "<http://faculty.baruch.cuny.edu/geoportal/resources/nyc_geog/nyc_zcta10_to_puma10.xls>"

This thi the ZIP code to ZCTA10 file "<http://faculty.baruch.cuny.edu/geoportal/resources/nyc_geog/zip_to_zcta10_nyc_revised.xls>"
