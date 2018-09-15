---
layout: post
title: "Streaming and visualizing Twitter data"
image: 
  path: /assets/images/posts/twitter-stream/twitter-post.png
  thumbnail: /assets/images/posts/twitter-stream/twitter-post.png
categories:
 - data analysis
tags:
 - social media
 - python
 - visualization
 - pandas
---

I would like to share how it is possible to stream tweets in real-time based on a set of defined keywords. In other words, how to capture public tweets as they are generated worldwide.

For this, I'll use the [`tweepy`](http://www.tweepy.org/){:target="_blank"} Python package , along with [`pandas`](https://pandas.pydata.org/){:target="_blank"} in a [Jupyter](https://jupyter.org/){:target="_blank"} environment.

**Note:** If you want to follow along, you will need to [apply for a developer account](https://developer.twitter.com/en/apply/user){:target="_blank"} (free and relatively painless). 
Once applied, you have to [create an "app"](https://developer.twitter.com/en/apps){:target="_blank"}, which will generate the ***Consumer API keys*** and ***Access token & access token secret***, necessary to authenticate with Twitter's API
{: .notice--info}

# Streaming

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

Then, I will set the keywords I want to track in the form of a list, and define the filename where the tweets will be written to. I also set a limit of 5000 tweets here, but it can be increased as you wish (although you may face an error if this number is too high). 
```python
TRACKING_KEYWORDS = ['donald trump']
OUTPUT_FILE = "trump_tweets.txt"
TWEETS_TO_CAPTURE = 5000
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
        if self.num_tweets <= TWEETS_TO_CAPTURE:
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
Great, at this point the tweets are being captured. The execution speed will depend on how "hot/trending" the keywords you defined currently are on Twitter. I defined keywords to collect data about Donald Trump, which is always a hot topic...  
You should see an output similar to this when it finishes:
```
Numer of tweets captured so far: 100
Numer of tweets captured so far: 200
Numer of tweets captured so far: 300
...
Numer of tweets captured so far: 4900
Numer of tweets captured so far: 5000
CPU times: user 5.53 s, sys: 809 ms, total: 6.33 s
Wall time: 1h 31min 45s
```

## Reading and converting the data to pandas DataFrame

All 5000 tweets were successfully collected and stored on the file defined earlier, ***in JSON format*** ("trump_tweets.txt" in my case). Let's open the file and store its data in a list of dictionaries `tweets_data`:

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

It is now possible to convert `tweets_data` to a `pandas` DataFrame with the `tweets_data` list. I will do it, selecting only a subset of columns for this demonstration:
```python
df = pd.DataFrame(tweets_data, columns=['created_at','lang', 'text', 'source'])
df.head()
```
![DF Head]({{ '/assets/images/posts/twitter-stream/df_head.png'|absolute_url}}){: .align-center}

As you can see, the DataFrame was successfully created, but I would like to work on 2 columns before continuing: 
- `created_at`: currently a string, even though it is clearly a date field, and I want to explicitly convert it to date 
- `source`: represents the Twitter client (Android, iPhone...) used for that tweet, but it is wrapped in a HTML code, and I want to get rid of it, keeping only the client name:

```python
# Just convert to datetime
df['created_at'] = pd.to_datetime(df.created_at)
# Regular expression to get only what's between HTML tags: > <
df['source'] = df['source'].str.extract('>(.+?)<', expand=False).str.strip() 

# Check DataFrame head again
df.head()
```
![DF Head]({{ '/assets/images/posts/twitter-stream/df_head_clean.png'|absolute_url}}){: .align-center}

This is a very simple DataFrame, but we can still check some interesting stuff. Here I am just checking the tweets count for Language and Source:
```python
df.lang.value_counts()

en     4293
fr      253
und     159
es      153
pt       40
tr       30
in       14
de       14
nl       11
no        6
...
el        1
cs        1
Name: lang, dtype: int64
----------
df.source.value_counts()

Twitter for iPhone                1672
Twitter for Android               1405
Twitter Web Client                 910
Twitter Lite                       333
Twitter for iPad                   301
IFTTT                               44
dlvr.it                             34
Facebook                            28
TweetDeck                           26
...
LinkedIn                             1
Gab.ai                               1
groupchat app                        1
Integromat                           1
Name: source, Length: 123, dtype: int64
```
# Visualizing
Naturally, English is the most popular language for this small dataset, and iPhone the most popular source people used to tweet.
So let's plot these numbers together (only the most popular ones not to become too messy).
```python
# create filter for most popular languages
lang_mask = (df.lang == 'en') | (df.lang == 'ca') | (df.lang == 'fr') | (df.lang == 'es')

# create a filter for most popular sources
source_mask = (df.source == 'Twitter for iPhone') | (df.source == 'Twitter for Android')\
    | (df.source == 'Twitter Web Client') | (df.source == 'Twitter for iPad') \
    | (df.source == 'Twitter Lite') | (df.source == 'Tweet Old Post')


(df[lang_mask & source_mask].groupby(['source','lang']) # apply filter/groupby
 .size() # get count of tweets per source/lang
 .unstack() # unstack to create new DF 
 .fillna(0) # fill NaN with 0
 .plot(kind='bar', figsize=(14,7), title='Tweets by source and language') # plot
)
```
![Bar Plot]({{ '/assets/images/posts/twitter-stream/tweet-source-lang.png'|absolute_url}}){: .align-center}

## Creating a Word Cloud

There is a really cool think we can do with the `text` column of our DataFrame: we can check the frequency with which a word appears in the texts, and then display them in a nice **Word Cloud**.  
Fortunately, a good soul developed a Python package for this purpose, called (be shocked): [wordcloud](https://amueller.github.io/word_cloud/index.html){:target="_blank"}

Alright, prior to creating a word cloud we should clean the `text` field of our DataFrame a little bit. It currently contains many mentions (@), special characters and a mix of uppercase/lowercase letters:

```python
# remove URLs, and twitter handles
for i in range(len(df['text'])):
    df['text'][i] = " ".join([word for word in df['text'][i].split()
                              if 'http' not in word and '@' not in word and '<' not in word])

# remove special characters and convert to lowercase
df['text'] = df['text'].apply(lambda x: re.sub('[!@#$:).;,?&-]', '', x.lower()))
df['text'] = df['text'].apply(lambda x: re.sub('  ', ' ', x))
```
Cool, it should be a bit cleaner. Now, let's convert the `text` column to a single string variable containing all text, from all rows of our DataFrame. This one-liner does the trick:
```python
text = ' '.join(txt for txt in df.text)
print ('There are {} words in the combination of text rows.'.format(len(text)))
```
`There are 482984 words in the combination of text rows.`

**It is time to create our Word Cloud!**  
The `wordcloud` package works together with `matplotlib`, so let's import them:
```python
import matplotlib.pyplot as plt
from wordcloud import WordCloud, STOPWORDS
```
I won't go into much detail about how the `wordcloud` package works, but it is quite easy to comprehend, just check out the [documentation](https://amueller.github.io/word_cloud/index.html){:target="_blank"} if you need help (also with a bunch of examples)
```python
stopwords = set(STOPWORDS) # pre-defined words to ignore
# adding extra words to ignore: 
# many tweets contain RT in the text, and we know the tweets are about Donald Trump
stopwords.update(['rt', 'donald', 'trump']) 

wordcloud = (WordCloud(background_color="white", # easier to read
                      max_words=50, # let's no polute it too much
                      stopwords=stopwords) # define words to ignore
                      .generate(text)) # generate the wordcloud with text

plt.figure(figsize=(15,10)) # make the plot bigger
# Show the plot (interpolation='bilinear' makes it better looking)
plt.imshow(wordcloud, interpolation='bilinear') 
plt.axis("off") 
```
![Word Cloud]({{ '/assets/images/posts/twitter-stream/twitter_wordcloud.png'|absolute_url}}){: .align-center}

Awesome, I believe it is quite easy to catch what was being spoken about Donald Trump during this "Twitter Streaming session", which took **1h31m, starting at 2018-09-15 11:00 CEST**. Here some things I could deduce from the image:
- **Paul Manafort**, most probably due to his [ongoing criminal trials](https://en.wikipedia.org/wiki/Trials_of_Paul_Manafort){:target="_blank"} in connection with the 2016 USA elections and the Russian interference;
- **Chinese / tariffs / bln**: we all know about the recent commercial war between USA and China (started by Trump, of course);
- **Crise** (crisis): well... the 2 topics above can explain this one.