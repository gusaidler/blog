---
layout: post
title: "Streaming Twitter data"
hidden: true
image: 
  path: /assets/images/posts/twitter-stream/twitter-post.png
  thumbnail: /assets/images/posts/twitter-stream/twitter-post.png
categories:
 - data analysis
tags:
 - social media
 - python
 - pandas
---

I would like to share how it is possible to stream tweets in real-time based on a set of defined keywords. In other words, how to capture tweets as they are generated worldwide.

For this, I'll use the [`tweepy`](http://www.tweepy.org/){:target="_blank"} Python library , along with [`pandas`](https://pandas.pydata.org/){:target="_blank"} in a [Jupyter](https://jupyter.org/){:target="_blank"} environment.

**Note:** If you want to follow along, you will need to [apply for a developer account](https://developer.twitter.com/en/apply/user){:target="_blank"} (free and relatively painless). 
Once applied, you have to [create an "app"](https://developer.twitter.com/en/apps){:target="_blank"}, which will generate the ***Consumer API keys*** and ***Access token & access token secret***, necessary to authenticate with Twitter's API
{: .notice--info}

So let's get started! First, I'll import the necessary packages and set the credentials (***Consumer API keys*** and ***Access token & access token secret***) to authenticate with the API:
```python
import tweepy
import pandas
import json # The API returns JSON formatted text

# Store OAuth authentication credentials - get at https://developer.twitter.com/en/apps
access_token = "-----(Access token)-----"
access_token_secret = "-----(Access token secret)-----"
consumer_key = "-----(API key)-----"
consumer_secret = "-----(API secret key)-----"

# Pass OAuth details to tweepy's OAuth handler
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
```

Then, I will set the keywords I want to track in the form of a list, and define the filename where the tweets will be written to. I also set a limit of 500 tweets here, but it can be increased as you wish (although you may face an error if this number is too high). 
```python
TRACKING_KEYWORDS = ['brazil', 'brasil', 'election', 'eleição']
OUTPUT_FILE = "tweets_brazil_election.txt"
TWEETS_TO_CAPTURE = 500
```
Now, the most important part of this demonstration: **the tweet listener class**.
This is a custom class inherited from the existing `tweepy.StreamListener`. The "customization" so to say is basically to support the output of tweets to a file and increase the count of tweets collected for control.
```python
class MyStreamListener(tweepy.StreamListener):
    """
    Twitter listener, collects streaming tweets and output to a file
    """
    def __init__(self, api=None):
        super(MyStreamListener, self).__init__()
        self.num_tweets = 0
        self.file = open(OUTPUT_FILE, "w")

    def on_status(self, status):
        tweet = status._json
        self.file.write( json.dumps(tweet) + '\n' )
        self.num_tweets += 1
        
        # Stops streaming when it reaches the limit
        if self.num_tweets < TWEETS_TO_CAPTURE:
            if self.num_tweets % 100 == 0: # just to see some progress...
                print('Numer of tweets captured so far: {}'.format(self.num_tweets))
            return True
        else:
            return False
        self.file.close()

    def on_error(self, status):
        print(status)
        
```
It's time to start streaming live tweets, so I will initialize the `MyStreamListener` class and pass it as an argument to `tweepy.Stream`, along with the authenticator set before.

To capture only tweets that fit the keywords I defined earlier, I need to use the `filter` method of the `Stream` class:
```python
%%time #let's see how long it takes

# Initialize Stream listener
l = MyStreamListener()

# Create you Stream object with authentication
stream = tweepy.Stream(auth, l)

# Filter Twitter Streams to capture data by the keywords:
stream.filter(track=TRACKING_KEYWORDS)
```
Great, at this point the tweets are being captured. The execution speed will depend on how "hot/trending" the keywords you defined currently are on Twitter. I defined keywords to collect data about elections in Brazil, which is currently a hot topic since it will take place next month (October/2018). 
You should see an output similar to this:
```
Numer of tweets captured so far: 100
Numer of tweets captured so far: 200
Numer of tweets captured so far: 300
Numer of tweets captured so far: 400
CPU times: user 502 ms, sys: 61.7 ms, total: 563 ms
Wall time: 1min 56s
```

All 500 tweets were successfully collected and stored on the file defined earlier ***in JSON format*** ("tweets_brazil_election.txt" in my case). Let's open the file and store its data in a list of dictionaries `tweets_data`:

```python
# Initialize empty list to store tweets
tweets_data = []

# Open connection to file
with open(OUTPUT_FILE, "r") as tweets_file:
    # Read in tweets and store in list
    for line in tweets_file:
        tweet = json.loads(line)
        tweets_data.append(tweet)
```
