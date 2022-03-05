# Rejected Plates
A few weeks ago I saw [a post on the /r/cars subreddit](https://www.reddit.com/r/cars/comments/siv6ik/in_florida_over_500_personalized_license_plate/) titled "_In Florida, Over 500 Personalized License Plate Requests Were Denied in 2021 - Here's the List_". It was a list of all the vanity plate requests that the Florida DMV had denied in 2021. A local Florida news station had gotten the data directly from the DMV, which I thought was very interesting. 

I wanted to get a similiar list for other states and post it - but where? My mind instincively went to the [@everyword Twitter bot](https://twitter.com/everyword?lang=en). Because there was very little additional context available regarding the license plates (just like the English words that @everyword tweeted) I figured I should post the rejected vanity plates in a similiar fashion: a single string being Tweeted once an hour or so. I could [write a Python script](https://github.com/perfectly-preserved-pie/rejectedplates) that would ingest the data via Pandas dataframe and iterate over it, Tweeting a single license plate at some set interval. Seemed easy enough...

## Fun With FOIA
But how could I get this data for other states and different years? I read the original news article and found a mention that the news station had submitted a FOIA request. "The Freedom of Information Act (FOIA) is a federal law that gives the public the right to make requests for federal agency records." There! I had a method to get data for other states now. After some more research on FOIA, I found a service that actually makes the requests for you, including following up, negotiating payments and data delivery methods, etc. It's called [MuckRock](https://www.muckrock.com/). Any FOIA request made on that website is then made publicly available for free, so I searched for rejected vanity plate records and [found about 11 of them](https://www.muckrock.com/foi/list/?csrfmiddlewaretoken=LK03Lo11SnN2japNrxaSeW31ymquwbe1YHlxvpQ4Z8aOTRrH10vMyHX6UfX73P2F&q=license+plate+vanity&status=done&has_embargo=&has_crowdfund=&minimum_pages=&date_range_min=&date_range_max=&file_types=). Some of them even had the data readily available in .CSV form which I could easily ingest via Panda's read_csv function.

Once I exhaust [my current supply of data](https://github.com/perfectly-preserved-pie/rejectedplates/tree/main/States), I intend on using MuckRock to file more FOIA requests.

### "The good thing about standards is that there are so many to choose from."
![image](https://user-images.githubusercontent.com/28774550/156232090-b3d30300-4afb-43ee-9ad8-e1ba6cc03396.png)

Unfortunately, every state has their own method of delivering this data, so each completed FOIA request wasn't exactly the same. Some files had different headers (or no headers at all), some had additional details like rejection reason while others didn't, some were delivered in PDF form or .xls (old school Excel). It wasn't hard but I did have to manually clean up the spreadsheets or export from PDF to CSV. You can see my results on [the GitHub page for the project](https://github.com/perfectly-preserved-pie/rejectedplates/tree/main/States).

### Side Note: Why Didn't I Post The Rejection Reason Along With The Plate?
I elected to only post the plate itself simply because the rejection reason wasn't available consistently. While I could ask for it, going through the Muckrock documents I realized that a state may not have this data readily available and could possibly charge (more) money to complete the request. Some explanations given by state representatives:
 * The reason is never logged/entered by policy
 * The data is logged, but is in a different system and therefore requires manual labor to associate with each plate configuration
 * The data is all pen-and-paper and nothing has been digitized (sounds about right for the DMV...). Therefore, manual labor and the associated costs are required.

So it made more sense to just post the plate configuration only, as that's the only thing that would be consistently available. The reason, the date, etc. might not be.

## New Coding Concepts
### Using the Twitter API via Tweepy
The script itself is pretty simple. I'm again using Pandas to iterate over a dataframe. One problem I didn't forsee was how to keep track of what's been posted and what hasn't. I originally had thought of just marking a dataframe column entry as "done" whenever a Tweet was posted; however the dataframe lives in memory and these tracking changes would get wiped if the script or VM ever crashed. So I needed to either:
1. Write to a file as a sort of log
2. Use Twitter's API to search for what's already been posted

I elected to use the latter method [using the Tweepy Python module](https://docs.tweepy.org/en/stable/client.html#tweepy.Client.get_users_tweets):

``` 
for plate in df.itertuples():
	try:
		# Get the most recent 10 tweets
		tweets = client.get_users_tweets(id=twitter_id,user_auth=True)
    ...
  # Create an empty list 
	tweets_list = []
	# Iterate over the tweets and add the tweet text to the empty list we just created
	for tweet in tweets.data:
		tweets_list.append(tweet.text)
	# Iterate over the new list. If the license plate we're about to post doesn't already exist, post it to Twitter
	if plate[1] not in tweets_list:
		try:
			client.create_tweet(text=plate[1],place_id=place_id)
			logging.info(f"{plate[1]} has been tweeted.")
			time.sleep(1800) 
```
Admittedly, this isn't very efficient or fault-tolerant.

#### The Looming Threat
Say that down the line, once the account has 2,000+ tweets for example, the script crashes and I restart it. It would iterate through the dataframe from the beginning and for plate #1, send an API call for the last 10 tweets. But since plate #1 has already been tweeted days/weeks/months ago and there's already 2000+ tweets, plate #1 wouldn't be found in the last 10 tweets and therefore technically not exist. The script would then re-post plate #1 which isn't what I want (no duplicates!)

According to Twitter's API docs, only the most 3200 recent tweets can be grabbed per one request via pagination. The first dataset alone (Maryland 2013) has approx. 4000 entries so I would need to use pagination and check that list for the plate entry instead of just looking through the last 10 tweets. Unfortunately that's something I didn't code for but I'm definitely looking into using pagination for subsequent datasets. For now, I'll take the risk that the Maryland 2013 dataset will complete without any crashing or hiccups.

### Logging & Alerting
This time around I wanted to make sure all progress/warnings/errors would be logged to a file. In addition, I also wanted to be alerted about any warnings or errors via Telegram.

#### Logging

Logging to a file was simple enough thanks to [Python's native logging module](https://docs.python.org/3/howto/logging.html#logging-to-a-file): 
```
# Log to a file
logging.basicConfig(
	filename='rejectedplates.log',
	encoding='utf-8',
	format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
	datefmt='%Y-%m-%d %I:%M:%S %p',
	level=logging.INFO)
```

Doing a `tail -f rejectedplates.log` gets me a nice autoscrolling output about what's going on:
```
2022-03-02 06:43:05 AM - root - INFO - 11MFA0 has been tweeted.
2022-03-02 07:13:06 AM - root - INFO - 11MFAO has been tweeted.
2022-03-02 07:43:06 AM - root - INFO - 11UV1R6 has been tweeted.
2022-03-02 08:13:07 AM - root - INFO - 11UVIR6 has been tweeted.
```

#### Alerting
But parsing through a log file isn't enough. I wanted to be notified immediately if there was any warning or error. I don't really need to be alerted any time there's a success, so no INFO or DEBUG-type messages would be needed. I opted to use a Telegram bot to notify me as that's what I've done for a few other homelab projects.

There are multiple Python Telegram modules available but since I didn't need anything super complicated or powerful I opted to use Telethon as that seemed the easiest to set up and get running. I [created a Telegram app](https://docs.telethon.dev/en/stable/basic/signing-in.html#signing-in) and added some code to sign in as a bot:
```
# Set up Telegram API stuff
# https://my.telegram.org, under API Development.
# https://docs.telethon.dev/en/stable/basic/signing-in.html#signing-in-as-a-bot-account
telegram_username = os.getenv('telegram_username')
api_id = os.getenv('telegram_api_id')
api_hash = os.getenv('telegram_api_hash')
bot_token = os.getenv('telegram_bot_token')
bot = TelegramClient('bot', api_id, api_hash).start(bot_token=bot_token)
```

And then added a few try/except blocks to send any exceptions to the Telegram bot which would alert me. For example, if there was an error retrieving the last tweets from the Twitter API:
```
except tweepy.TweepError as e:
		timeline_error_msg = f"Couldn't get the last 10 tweets because {e.reason}"
		logging.error(timeline_error_msg)
		bot.send_message(telegram_username, timeline_error_msg) # send a Telegram message
		continue # Skip this iteration of the for loop and continue to the next one
 ```

I both log it to a file and send it via Telegram.

Or if a plate configuration had already been tweeted:
```
elif plate[1] in tweets_list:
		post_warning_msg = f"{plate[1]} was already tweeted, skipping..."
		logging.warning(post_warning_msg)
		bot.send_message(telegram_username, post_warning_msg)
```

## What's Next For @rejectedplates?
I definitely need to work on the [Tweet search/pagination issue I mentioned above](https://github.com/perfectly-preserved-pie/perfectly-preserved-pie.github.io/blob/master/_posts/2022-03-01-rejectedplates.md#the-looming-threat). There's also no doubt in my mind that I'll need to rewrite some of the CSV ingestion lines to handle the differing formats of each CSV dataset. I plan on continuiing this bot for as long as I can get fresh data for it. Theoretically, I have 50 states per year for 8 years to get through so I should be busy for quite a while...
