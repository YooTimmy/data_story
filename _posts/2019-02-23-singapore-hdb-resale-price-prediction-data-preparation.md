---
layout: post
title:  "Singapore HDB Resale Price Prediction: Data Preparation"
date:   2019-02-23 12:26:57 +0800
categories:
---

In Singapore, for young working adults who are about to start a family, HDB(Singapore Public Housing) would be their first choice when they have a limited budget. However, it could become a real headache for those who are considering which HDB flat to choose: town areas, nearby facilities(e.g. schools, shopping malls), flat types, etc. To make people’s life easier when facing such significant choices, I decided to do a study for Singapore HDB resale price, explore what are the key factors affecting HDB resale price, and make some recommendations for people who are planning to purchase a HDB in a near future.

## Getting to know the dataset

The HDB resale data can be retrieved from [data.gov.sg](https://data.gov.sg/dataset/resale-flat-prices). Below is an example for the columns available from raw dataset:

![Sample Data from HDB Resale Price Dataset](https://cdn-images-1.medium.com/max/3896/1*nojo-B9YnibG-rcA8kvHtQ.png)*Sample Data from HDB Resale Price Dataset*

Below are some of the first impressions when I first look at these columns:

**Month** : Given in the format of year-month. We may retrieve the year data from this column, which may be useful when analysing the time trend for HDB resale price.

**Town** : Town location should be one of the key factors affecting HDB resale price — we are generally expecting an HDB flat in Orchard has much higher resale price than Yishun given same flat type.

**Flat Type**: There are total 7 different kinds of flat types: 1 Room, 2 Room, 3 Room, 4 Room, 5 Room, EC and Multi-generation. Among which the 4 Room HDB flats are the most popular ones in Singapore. We may consider using 4 Room data samples to construct the model.

**Storey Range** : This column is given as a string rather than numbers, we may need to do some data munging accordingly if we want to use it to build model.

**Flat Model** : Similarly, there are plenty of different flat models out there(35 different types). This factor would play an important role for the overall flat price. E.g., the DBSS (Design, Build and Sell Scheme) flats would have a higher resale price considering it allows buyer to design the HDB based on their own style.

**Remaining Lease** : Singapore HDB has a lease of 99 years. This column data has quite some NULL values, and it is calculated based on different years. We may need to adjust this column data accordingly when building the model.

## Feature Engineering

After getting some rough ideas about the dataset, we may start to do data wrangling. Below are some feature engineering work I have performed to pre-process corresponding data columns:

**Town : **To use this data column as an input for regression model, I have aggregate the dataset by Town, and take the median resale price for each town to subtract the overall median resale price. The result (‘town_premium’ column) can be interpreted as the amount money one need to pay additionally when they choose this location. For instance, one may need to pay an additional ~12.8k on average when buying HDB flats in Bukit Merah, while one can pay ~5.6K less if they choose a HDB flat in Bukit Panjiang area.

![Pre-Processing the “Town” Column](https://cdn-images-1.medium.com/max/4064/1*ipg9EOQ6jdVSJTwIGQgEuA.png)*Pre-Processing the “Town” Column*

**Flat_Model** : Similar pre-processing is done for “flat_model” column as well. As mentioned previously, we do see the DBSS HDB flats resale price is higher than average, one may need to pay an extra ~35K on average for this flat type.

![](https://cdn-images-1.medium.com/max/4044/1*WT4-vedKpaa4Ax-WuKG0yA.png)

**Storey_Range :** I just simply took the mean value from the two numbers in the given string. For example, for “04 TO 06” column, I will just assign the storey to be 5. It should be a good enough approximation, given the storey range spread is not really larger for each category (3 or 5 storey difference).

![](https://cdn-images-1.medium.com/max/4040/1*u1dqvWQu-4WWRo6HCdT7Ew.png)

**Remaining Lease** : The original remaining lease column contains NULL value, and they are referenced to different years depending the year of retrieval date. I have re-calculate the remaining lease value at resale date with the following formula:

**Formula**: remain_lease = lease_commence_date + 99 –resale_year

## What can we obtained from “Street Name”?

So now we have preprocessed the majority of columns in raw dataset. Looks all good… Wait, how about the “Street Name” column?

As mentioned in the first section, this column actually indicates the geographical location data for each HDB flats. It is actually a key factor for HDB resale price, as it would indicate if the HDB is near any MRT station, or if there are any shopping malls nearby. These information should be quite important based on my experience when renting HDB flats, and I believe it should apply to HDB resale price as well.

We may try to use GoogleMap or other available map API to extract the longitude and latitude for different street names, MRT stations as well as major shopping malls. Then we can obtain the relative distance from these street name to their nearest MRT and shopping mall as metrics for resale price prediction.

After doing some research, I noticed that [Onemap.sg](https://docs.onemap.sg/) offers a API which enable users conveniently query the longitude and latitude given a specific address. I write a simple python script to query location information using Onemap API.

    *###Sample Script to extract longitude/latitude information given street name*

    import json
    import requests
    import openpyxl
    import time
    
    # Open Excel sheet
    wb = openpyxl.load_workbook('Street_Name_List.xlsx')
    sheet = wb['Sheet1']
    
    count = 0
    for row in range(2, sheet.max_row + 1):
       if count < 250:  # retrive
          street_name = sheet['A' + str(row)].value
    
          query_string = 'https://developers.onemap.sg/commonapi/search?searchVal=' + str(
             street_name) + '&returnGeom=Y&getAddrDetails=N&pageNum=1'
          result = requests.get(query_string)
    
          # Convert JSON into Python Object
          data = json.loads(result.content)
          # Extract data from JSON
          sheet['B' + str(row)] = data['results'][0]['LONGITUDE']
          sheet['C' + str(row)] = data['results'][0]['LATITUDE']
          count = count + 1
       else:
          print("Pausing for 10 Seconds")
          time.sleep(10)
          count = 0
       wb.save('Street_Name_List.xlsx')
    
    print('Job Done.')

We can do the same thing to retrieve both Singapore MRT/Shopping Mall longitude/latitude information. The list of Singapore MRT stations could be found [here](https://en.wikipedia.org/wiki/List_of_Singapore_MRT_stations) , and the list of major shopping malls in Singapore can be extracted from [here](https://en.wikipedia.org/wiki/List_of_shopping_malls_in_Singapore).

One additional information we need to obtain is which Mall/MRT is the closet one given the street name. We may just calculate the relative distance for a street name with each Mall/MRT with a for loop and then obtain the smallest distance target accordingly. The sample code is attached for reference:

    *##Sample Script to obtain the closest shopping mall name given street name*

    import openpyxl
    import numpy as np
    
    def calculate_distance(x,y):
        return np.sqrt(((x[0]-y[0])*110.574)**2 + ((x[1]-y[1])*111.32)**2)
    
    wb = openpyxl.load_workbook('Street_Name_List.xlsx')
    wb2 = openpyxl.load_workbook('Shopping_Mall_List.xlsx')
    sheet = wb['Sheet1']
    sheet2 = wb2['Sheet1']
    for row in range (2, sheet.max_row +1):
       Distance = 200
       MRT = 0
       if sheet['B'+str(row)].value is not None:
          for k in range (2, sheet2.max_row + 1):
             x1 = sheet['B'+str(row)].value
             x2 = float(sheet2['B'+str(k)].value)
             y1 = sheet['C'+str(row)].value
             y2 = float(sheet2['C'+str(k)].value)
             #print (type(x1),type(x2),y1,y2)
             Dis_Temp = calculate_distance([x1,y1],[x2,y2])
             if Dis_Temp < Distance:
                Distance = Dis_Temp
                MRT = k
             else:
                continue
       else:
          continue
       print(MRT)
       print(sheet2['A'+str(MRT)].value)
       sheet['F'+str(row)].value = sheet2['A'+str(MRT)].value
    
    wb.save('Street_Name_List.xlsx')
    print('Job Done.')

If you follow along the way, now you should have already obtained the closest MRT/shopping mall name for each street name in dataset. With all the longitude/latitude data queried from API, we can easily calculate the relative distance from each street to their closet MRT/Shopping Mall. For easier visualisation, I have converted the relative distance(km) into the time needed by walking (mins) from corresponding street (1 mins walk is mapped to 80 meters distance on average).

Another interesting metrics I can think of is the travelling time by public transport from the nearest MRT station to the CBD (for example, Raffles Place MRT station). This could be an important metric to look at for young working adults who use MRT as the major means of transportation, since buying a car in Singapore is not really affordable for everyone.

Instead of calculating the time based on our own estimation, we can use the OneMap API routing service to query the travelling time by public transport. More details for the API usage can be found [here](https://docs.onemap.sg/#route).

With all the feature engineering works mentioned above, we have finally obtain the dataset with desired features below:

(Note: The Walk_Time_to_MRT, Walk_Time_to_Mall and Travel_Time_to Raffles are all in mins. town_premium & flat_model_premium are all in Singapore Dollars)

![Final Dataset to be used for resale price prediction model](https://cdn-images-1.medium.com/max/3404/1*YUoptjf0DqUbGQweqijWGw.png)*Final Dataset to be used for resale price prediction model*

Before we move on to modelling, we will explore more about the dataset first and find some interesting patterns from it. This will be covered in the next post.

Thanks for reading!
