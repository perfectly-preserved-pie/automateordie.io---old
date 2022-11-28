# WhereToLive.LA

## You can view [my source code](https://github.com/perfectly-preserved-pie/larentals) on GitHub. The website is located at [https://wheretolive.la](https://wheretolive.la).

### I recommend viewing the website on a device with a large screen (tablet, laptop, etc.). Mobile devices won't have a good UI experience (for the reason why see the [Challenges section at the end)](https://automateordie.io/wheretolivedotla/#challenges).

---

I've been looking to move into a new place for a while now and have been amassing resources and websites that I can peruse through to find the right place.
One of the moderators on [the /r/LArentals subreddit](https://www.reddit.com/r/LARentals/) posts [a spreadsheet](https://docs.google.com/spreadsheets/d/1gBLt73zziGg41IUS3FdqU4ddULWV5sFaSRawLyO5YyY/edit#gid=80625330) of new rental properties every week.

As you can see, [it's quite detailed and organized.](https://user-images.githubusercontent.com/28774550/188333676-11846918-2c1d-4ebd-aa92-897fc2f18dfa.png)

You can filter by any column to narrow down the results list. I absolutely loved that; Zillow, RedFin, etc. don't have as quite a granular filter as this spreadsheet does.

But I was a little miffed that I kept having to open new tabs and paste the street address into Google to find out where in LA the property resided. At 350+ rows, that's a LOT of browser tabs and LA County is such a massive sprawl that I simply can't visualize where every property is.

## The Idea
So I thought, "why not just visually map every street address on Google Maps?" And then that idea became "the map should be filterable in all the same ways as the spreadsheet."
So I wanted a filterable map. And that's how I fell down the rabbit hole...

## The Tech Stack
* BeautifulSoup 4
* Dash Leaflet
* Dash Bootstrap Components
* GeoPy
* ImageKit
* Pandas


### Data Handling
Because the spreadsheet was already in CSV form, Pandas was an obvious choice here. I could simply just read it into a dataframe and add columns and manipulate the data in whatever way I needed to.

### Geocoding
I first needed some kind of API that I could feed street addresses from the spreadsheet and it would spit out the assoicated coordinates. I intially spun up an instance of Nominatim because it was free and easy and tried that. Unfortunately, quite a few addresses just simply wouldn't resolve in Nominatim but resolved just fine with the Google Maps API. So I switched over to Google Maps, which provides [a generous free tier](https://mapsplatform.google.com/pricing/) ($200/month).
I haven't had any issues since then; it handled every address I threw at it and returned me accurate coordinates. 

I created a quick function to return coordinates based on a provided street address:

```python
# Create a function to get coordinates from the full street address
g = GoogleV3(api_key=os.getenv('GOOGLE_API_KEY')) # https://github.com/geopy/geopy/issues/171
def return_coordinates(address):
    try:
        geocode_info = g.geocode(address)
        lat = float(geocode_info.latitude)
        lon = float(geocode_info.longitude)
        coords = f"{lat}, {lon}"
    except Exception:
        lat = NaN
        lon = NaN
        coords = NaN
    return lat, lon, coords
```

And then I iterated over every row with that function:

```python
# Iterate through the dataframe and fetch coordinates for rows that don't have them
# If the Coordinates column is already present, iterate through the null cells
# Similiar to above, we can use the presence of the Coordinates column as a proxy for Longitude and Latitude; all 3 should exist together or none at all
# This assumption will reduce the number of API calls to Google Maps
if 'Coordinates' in df.columns:
    for row in df['Coordinates'].isnull().itertuples():
        print(f"Grabbing coordinates for row #{row.Index}...")
        coordinates = return_coordinates(df.at[row.Index, 'Full Street Address'])
        df.at[row.Index, 'Latitude'] = coordinates[0]
        df.at[row.Index, 'Longitude'] = coordinates[1]
        df.at[row.Index, 'Coordinates'] = coordinates[2]
# If the Coordinates column doesn't exist (i.e this is a first run), create it using df.at
elif 'Coordinates' not in df.columns:
    for row in df.itertuples():
        print(f"Grabbing coordinates for row #{row.Index}...")
        coordinates = return_coordinates(df.at[row.Index, 'Full Street Address'])
        df.at[row.Index, 'Latitude'] = coordinates[0]
        df.at[row.Index, 'Longitude'] = coordinates[1]
        df.at[row.Index, 'Coordinates'] = coordinates[2]
```


### Mapping
Now that I had a list of coordinates, I needed a way to actually _display_ the points on the map. This led me to [Folium](http://python-visualization.github.io/folium/), however I wasn't too happy with the look and soon moved on to [Dash by Plotly](https://github.com/plotly/dash). Even then I still wasn't satisified with any of the map types. Heatmaps, chloropeths, etc. all were too complex for what I wanted: a simple marker with a table of the property's characteristics (rent price, garage spaces, address, etc.). My search led me to [Dash-Leaflet](https://dash-leaflet.herokuapp.com/) which was perfect. Not only did it look good but nearby points could all be part of a cluster group that would expand and shrink as the user zoomed the map in or out:

![image](https://user-images.githubusercontent.com/28774550/188334554-2006aed2-747b-42db-8505-691bb261d46b.png)

I wanted each marker to show the property details, so I created a pretty massive function to return HTML code for the marker's popup:
```python
# Define HTML code for the popup so it looks pretty and nice
def popup_html(row):
    i = row.Index
    street_address=df['Full Street Address'].at[i] 
    mls_number=df['Listing ID'].at[i]
    mls_number_hyperlink=df['bhhs_url'].at[i]
    mls_photo = df['MLS Photo'].at[i]
    lc_price = df['List Price'].at[i] 
    price_per_sqft=df['Price Per Square Foot'].at[i]                  
    brba = df['Br/Ba'].at[i]
    square_ft = df['Sqft'].at[i]
    year = df['YrBuilt'].at[i]
    garage = df['Garage Spaces'].at[i]
    pets = df['PetsAllowed'].at[i]
    phone = df['List Office Phone'].at[i]
    terms = df['Terms'].at[i]
    sub_type = df['Sub Type'].at[i]
    listed_date = pd.to_datetime(df['Listed Date'].at[i]).date() # Convert the full datetime into date only. See https://stackoverflow.com/a/47388569
    furnished = df['Furnished'].at[i]
    key_deposit = df['DepositKey'].at[i]
    other_deposit = df['DepositOther'].at[i]
    pet_deposit = df['DepositPets'].at[i]
    security_deposit = df['DepositSecurity'].at[i]
...
```

That function uses the Pandas dataframe row to populate different fields like rental price, garage spaces, terms, etc. and formats them into an HTML table for that specific marker:

![image](https://user-images.githubusercontent.com/28774550/188334715-20842be0-b171-4631-8c1b-cf908ee1715a.png)

So now, whenever a user clicks on a marker, a popup appears with that property's details. Very handy to see what it's like at a glance.

### Filters
So I had the map and markers ready, but how could I filter the markers depending on different variables? Luckily, that's what Dash does via "[callbacks](https://dash.plotly.com/basic-callbacks)". A box could be checked or a slider could be dragged, and that would cause an action on the backend to fire off. In my case, a checked box would need to change the dataframe; the dataframe is how the markers on the map are being populated. So if I wanted to see ONLY condos, I would need to query the dataframe for ONLY condos.

```python
def update_map(subtypes_chosen, pets_chosen, terms_chosen, garage_spaces, rental_price, bedrooms_chosen, bathrooms_chosen, sqft_chosen, years_chosen, sqft_missing_radio_choice, yrbuilt_missing_radio_choice, garage_missing_radio_choice, ppsqft_chosen, ppsqft_missing_radio_choice, furnished_choice, security_deposit_chosen, security_deposit_radio_choice, pet_deposit_chosen, pet_deposit_radio_choice, key_deposit_chosen, key_deposit_radio_choice, other_deposit_chosen, other_deposit_radio_choice, listed_date_datepicker_start, listed_date_datepicker_end, listed_date_radio):
  df_filtered = df[
    (df['Sub Type'].isin(subtypes_chosen)) &
    pets_radio_button(pets_chosen) &
    (df['Terms'].isin(terms_chosen)) &
    # For the slider, we need to filter the dataframe by an integer range this time and not a string like the ones aboves
    # To do this, we can use the Pandas .between function
    # See https://stackoverflow.com/a/40442778
    (((df['Garage Spaces'].between(garage_spaces[0], garage_spaces[1])) | garage_radio_button(garage_missing_radio_choice, garage_spaces[0], garage_spaces[1]))) & # for this one, combine a dataframe of both the slider inputs and the radio button input
    # Repeat but for rental price
    (df['List Price'].between(rental_price[0], rental_price[1])) &
    (df['Bedrooms'].between(bedrooms_chosen[0], bedrooms_chosen[1])) &
    (df['Total Bathrooms'].between(bathrooms_chosen[0], bathrooms_chosen[1])) &
    (((df['Sqft'].between(sqft_chosen[0], sqft_chosen[1])) | sqft_radio_button(sqft_missing_radio_choice, sqft_chosen[0], sqft_chosen[1]))) &
    (((df['YrBuilt'].between(years_chosen[0], years_chosen[1])) | yrbuilt_radio_button(yrbuilt_missing_radio_choice, years_chosen[0], years_chosen[1]))) &
    (((df['Price Per Square Foot'].between(ppsqft_chosen[0], ppsqft_chosen[1])) | ppsqft_radio_button(ppsqft_missing_radio_choice, ppsqft_chosen[0], ppsqft_chosen[1]))) &
    furnished_checklist_function(furnished_choice) &
    security_deposit_function(security_deposit_radio_choice, security_deposit_chosen[0], security_deposit_chosen[1]) &
    pet_deposit_function(pet_deposit_radio_choice, pet_deposit_chosen[0], pet_deposit_chosen[1]) &
    key_deposit_function(key_deposit_radio_choice, key_deposit_chosen[0], key_deposit_chosen[1]) &
    other_deposit_function(other_deposit_radio_choice, other_deposit_chosen[0], other_deposit_chosen[1]) &
    listed_date_function(listed_date_radio, listed_date_datepicker_start, listed_date_datepicker_end)
  ]

  # Create markers & associated popups from dataframe
  markers = [dl.Marker(children=dl.Popup(popup_html(row)), position=[row.Latitude, row.Longitude]) for row in df_filtered.itertuples()]
  ```
  
### Web Scraping
  Because knowing _when_ a listing was posted is important (a listing from 9 months ago probably isn't going to be available) I wanted to get the "listed" date. That also led me to finding an MLS photo associcated with the property, so I figured I'd scrape that too and insert that photo into the HTML popup for the property.

```python
## Webscraping Time
# Create a function to scrape the listing's Berkshire Hathaway Home Services (BHHS) page using BeautifulSoup 4 and extract some info
def webscrape_bhhs(url, row_index):
    try:
        response = requests.get(url)
        soup = bs4(response.text, 'html.parser')
        # First find the URL to the actual listing instead of just the search result page
        try:
          link = 'https://www.bhhscalifornia.com' + soup.find('a', attrs={'class' : 'btn cab waves-effect waves-light btn-details show-listing-details'})['href']
          logging.info(f"Successfully fetched listing URL for {row_index}.")
        except AttributeError as e:
          link = None
          logging.warning(f"Couldn't fetch listing URL for {row_index}. Passing on...")
          pass
        # If the URL is available, fetch the MLS photo and listed date
        if link is not None:
          # Now find the MLS photo URL
          # https://stackoverflow.com/a/44293555
          try:
            photo = soup.find('a', attrs={'class' : 'show-listing-details'}).contents[1]['src']
            logging.info(f"Successfully fetched MLS photo for {row_index}.")
          except AttributeError as e:
            photo = None
            logging.warning(f"Couldn't fetch MLS photo for {row_index}. Passing on...")
            pass
          # For the list date, split the p class into strings and get the last element in the list
          # https://stackoverflow.com/a/64976919
          try:
            listed_date = soup.find('p', attrs={'class' : 'summary-mlsnumber'}).text.split()[-1]
            logging.info(f"Successfully fetched listed date for {row_index}.")
          except AttributeError as e:
            listed_date = pd.NaT
            logging.warning(f"Couldn't fetch listed date for {row_index}. Passing on...")
            pass
        elif link is None:
          pass
    except Exception as e:
      listed_date = pd.NaT
      photo = NaN
      link = NaN
      logging.warning(f"Couldn't scrape BHHS page for {row_index} because of {e}. Passing on...")
      pass
    return listed_date, photo, link 
```

## Challenges
I've been eternally frustrated by these first two challenges and saddened by the third one:
1. Slow rendering of markers due to poor Pandas dataframe query performance
2. The map looks like shit on mobile
3. ~~And now, [the announcement](https://www.reddit.com/r/LARentals/comments/z48xdj/no_rentals_list_going_forward_11252022/) from the person who makes the spreadsheets that the weekly spreadsheets are no more~~

I'll address these 3 challenges below:

1: probably caused by [the 16 separate Pandas operations that take place every time you change a user option](https://github.com/perfectly-preserved-pie/larentals/blob/master/app.py#L938-L960) like rent price, pet policy, etc. I haven't been able to figure out a way to optimize that.

2: caused by me not being a web developer and not understanding how to make the Dash-Leaflet HTML popup resize itself based on device size like the cards do with Dash Bootstrap Components. [As a result the marker popup is rendered _way too big_ for mobile devices](https://user-images.githubusercontent.com/28774550/204154932-d7d41930-6d47-49d1-90c7-5ca619b6c03a.jpeg). The popup gets cut off and takes up most of the available space on the map, making it difficult to navigate away. The website is only functionally usable on big screens like tablets, laptops, and monitors. If you have any ideas on how to resolve this, please submit a PR or issue! 😩

3: ~~The tl;dr of it is that the guy/gal who posts these CSV spreadsheets weekly has [announced that they can't do it any longer](https://www.reddit.com/r/LARentals/comments/z48xdj/no_rentals_list_going_forward_11252022/). I had a feeling this might happen but was hoping to finish this project in time before it happened so people got some use out of it. Alas, I was too late/slow. Without any new incoming data, this project is dead in the water and will soon be totally irrelevant as the current rental listings change or drop off the market.~~

~~I knew that would be a risk; any project that relies on a single dependency is a risky endeavor:~~

![one guy](https://imgs.xkcd.com/comics/dependency.png)

### **EDIT!** #3 has been resolved; after getting overwhelming support the person has decided to resume posting the weekly spreadsheets! Fuck yeah!!! This project is back on track baby 🎉

I hope you get some use out of this website; it was a labor of love.
