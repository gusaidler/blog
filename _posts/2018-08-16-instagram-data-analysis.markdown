---
layout: post
title: "Data analysis: Your Instagram data"
image: 
  path: /assets/images/instagram-download.png
  thumbnail: /assets/images/instagram-download.png
categories:
 - python
 - pandas
 

---

In this post I want to talk a bit about how to explore your own Instagram account and generate interesting insights.
I will be using [iPython - Jupyter Notebook](https://jupyter.org/) for this, with the following packages:
- pandas
- [LevPasha/Instagram-API-python](https://github.com/LevPasha/Instagram-API-python) (unofficial Instagram API)


First of all, please read the [README](https://github.com/LevPasha/Instagram-API-python/blob/master/README.md) in order to install the InstagramAPI package.

I assume you will have `pandas` installed already.

Then, let's import the necessary packages that will be used along this demonstration:

```python
from InstagramAPI import InstagramAPI
import pandas as pd
from pandas.io.json import json_normalize
```

Then, create a small function to login to Instagram with your account:

```python
def login_to_instagram(username, password):
    api = InstagramAPI(username, password)
    api.login()
    
    return api

api = login_to_instagram('instagram_username','instagram_password')
```
Once executed, you should receive this return message: `Login success!`

Cool, we are logged in to Instagram! 
Now we can start exploring it. I believe it's natural to start retrieving all your posts:
```python
def get_my_posts(api):
    '''Retrieve all posts from own profile'''
    my_posts = []
    has_more_posts = True
    max_id= ''

    while has_more_posts:
        api.getSelfUserFeed(maxid=max_id)
        if api.LastJson['more_available'] is not True:
            has_more_posts = False #stop condition

        max_id = api.LastJson.get('next_max_id','')
        my_posts.extend(api.LastJson['items']) #merge lists
        time.sleep(2) # slows down to avoid flooding

        if has_more_posts:
            print(str(len(my_posts)) + ' posts retrieved so far...')

    print('Total posts retrieved: ' + str(len(my_posts)))
    
    return my_posts

my_posts = get_my_posts(api)
```
The output should look like this:
```
18 posts retrieved so far...
36 posts retrieved so far...
54 posts retrieved so far...
72 posts retrieved so far...
Total posts retrieved: 90
```
`my_posts` will be a list of dictionaries, and each item represents a single post from your Instagram account. In my case, I have only 90 posts. 
I encourage you to explore the available fields of each post. Many interesting stuff there :)

Now that we have all the posts, let's retrieve all the "post likers" to see which users like your posts
```python
def get_posts_likers(api, my_posts):
    '''Retrieve all likers on all posts'''
    
    likers = []
    
    print('wait %.1f minutes' % (len(my_posts)*2/60.))
    for i in range(len(my_posts)):
        m_id = my_posts[i]['id']
        api.getMediaLikers(m_id)
        
        likers += [api.LastJson]
        
        # Include post_id in likers dict list
        likers[i]['post_id'] = m_id
        
        time.sleep(2)
    print('done')
    
    return likers


likers = get_posts_likers(api, my_posts)  
```
This should take some minutes, depending on the amount of posts you have. An approximate wait time will be displayed:
```
wait 3.0 minutes
``` 
You will receive `done` when it finished. 
`likers` will also be a list of dictionaries, and should have the same length as `my_posts`. Inside each dictionary, you will find the key `users`, which contain all the users that liked a specific post.

Ok, let's do a similar operation, but to return the post commenters this time:
```python
def get_posts_commenters(api, my_posts):
    '''Retrieve all commenters on all posts '''
    
    commenters = []
    
    print('wait %.1f minutes' % (len(my_posts)*2/60.))
    for i in range(len(my_posts)):
        m_id = my_posts[i]['id']
        api.getMediaComments(m_id)
        
        commenters += [api.LastJson]
        
        # Include post_id in commenters dict list
        commenters[i]['post_id'] = m_id
            
        time.sleep(2)
    print('done')
    
    return commenters

commenters = get_posts_commenters(api, my_posts)
```
You will have to wait the same time as you waited to retrieve the `likers`, and `done` will be printed out once it's finished.
`commenters` will also be a list of dictionaries, and should have the same length as `my_posts`. The actual comments will be under the key `comments` of each item of the `commenters` list. 

## Converting to pandas DataFrames

It's time to use the powerful `pandas` package and structure this data a little bit. I'll use [json_normalize](https://pandas.pydata.org/pandas-docs/version/0.22/generated/pandas.io.json.json_normalize.html) within pandas.io.json. The data transformation differs a bit betweeen `likers` and `commenters`:
```python
def posts_likers_to_df(likers):
    '''Transforms likers list of dicts into pandas DataFrame'''
    
    # Normalize likers by getting the 'users' list and the post_id of each like
    df_likers = json_normalize(likers, 'users', ['post_id'])
    
    # Add 'content_type' column to know the rows are likes
    df_likers['content_type'] = 'like'
    
    return df_likers

def posts_commenters_to_df(commenters):
    '''Transforms commenters list of dicts into pandas DataFrame'''
    
    # Include username and full_name of commenter in 'comments' list of dicts
    for i in range(len(commenters)):
        if len(commenters[i]['comments']) > 0: # checks if there is any comment on the post
            for j in range(len(commenters[i]['comments'])):
                # Puts username/full_name one level up
                commenters[i]['comments'][j]['username'] = commenters[i]['comments'][j]['user']['username']
                commenters[i]['comments'][j]['full_name'] = commenters[i]['comments'][j]['user']['full_name']
                
    # Create DataFrame
    # Normalize commenters to have 1 row per comment, and gets 'post_id' from parent 
    df_commenters = json_normalize(commenters, 'comments', 'post_id')
    
    # Get rid of 'user' column as we already handled it above
    del df_commenters['user']
    
    return df_commenters

df_likers = posts_likers_to_df(likers)
df_commenters = posts_commenters_to_df(commenters)
```
With this, we have 2 panda DataFrame: `df_likers` and `df_commenters`. Each row of `df_likers` represents a single like, and each row of `df_commenters` represents a single comment.
We can now get some interesting numbers. Let's start with some basic counts:
```python
print('Total posts: ' + str(len(my_posts)))
print('---------')
print('Total likes on profile: ' + str(df_likers.shape[0])) #shape[0] represents number of rows
print('Distinct users that liked your posts: ' +str(df_likers.username.nunique())) # nunique() will count distinct values of a col
print('---------')
print('Total comments on profile: ' + str(df_comment.shape[0]))
print('Distinct users that commented your posts: ' +str(df_comment.username.nunique()))
```
```
Total posts: 90
---------
Total likes on profile: 3254
Distinct users that liked your posts: 508
---------
Total comments on profile: 124
Distinct users that commented your posts: 51
```

### Top 10 likers of my Instagram account:

```python
# As each row represents a like, we can perform a value_counts on username and slice it to the first 10 items (pandas already order it for us)
df_likers.username.value_counts()[:10]
```
```
tetxhrcc           71
tucashxhrchinc     58
trlosxhriciuc      52
tepixhrninc        50
tuibaxhreric       44
tayonexhrllmanc    42
tateusxhrborlic    41
toanaxhratic       40
tprimxhrno1c       39
taraxhrdinc        38
Name: username, dtype: int64
```
Hmm, "tetxhrcc" is the person who liked my post the most: out of 90 posts, he liked 71 of them!
Let's plot the distribution of this Top 10:
##### Bar plot
```python
df_likers.username.value_counts()[:10].plot(kind='bar', title='Top 10 media likers', grid=True, figsize=(12,6))
```
![Bar plot]({{ '/assets/images/posts/instagram-data-analysis/likers_plot_bar.png'|absolute_url}}){: .align-center}
##### Pie plot
```python
df_likers.username.value_counts()[:10].plot(kind='pie', title='Top 10 media likers distribution', autopct='%1.1f%%', figsize=(12,6))
```
![Pie plot]({{ '/assets/images/posts/instagram-data-analysis/likers_plot_pie.png'|absolute_url}}){: .align-center}

We can do the same for the post commenters:
### Top 10 commenters of my Instagram account:
```python
df_commenters['username'].value_counts()[:10].plot(kind='bar', figsize=(12,6), title='Top 10 post commenters')
```
![Bar plot]({{ '/assets/images/posts/instagram-data-analysis/commenters_plot_bar.png'|absolute_url}}){: .align-center}

The interesting thing about comments is that the actual datetime is captured for each one of them (not available for likes, unfortunately). The fields are: `created_at` and `created_at_utc`. Therefore, we can play around with it a little bit. 

For example, I was curious to know on which day of the week I receive more comments. To accomplish this, first we have to convert the columns to actual datefime type:
```python
# Converts date from unix time to YYYY-MM-DD hh24:mm:ss
df_commenters.created_at = pd.to_datetime(df_commenters.created_at, unit='s')
df_commenters.created_at_utc = pd.to_datetime(df_commenters.created_at_utc, unit='s')
```
Now, we can make use of the [Datetime Properties](https://pandas.pydata.org/pandas-docs/stable/api.html#datetimelike-properties), which allow us to return many properties, like: `year, month, week, dayofweek...`
So, continuing with the example, I can now plot on which day of the week I receive more comments:
```python
df_commenters.created_at.dt.weekday.value_counts().sort_index().plot(kind='bar', figsize=(12,6), title='Comments per day of the week (0 - Sunday, 6 - Saturday)')
```
![Bar plot]({{ '/assets/images/posts/instagram-data-analysis/commenters_weekday_plot_bar.png'|absolute_url}}){: .align-center}

I can go a bit further and check at what time I usually receive comments. It is basically the same as above, I just need to change the `dt` property to `hour`:
```python
df_commenters.created_at.dt.hour.value_counts().sort_index().plot(kind='bar', figsize=(12,6))
```
![Bar plot]({{ '/assets/images/posts/instagram-data-analysis/commenters_hour_utc_plot_bar.png'|absolute_url}}){: .align-center}

The thing is, I am actually Brazilian and I know for a fact that most part of my "Instagram audience" is also Brazilian, but the `create_at` field is in UTC time. So, at least for me, the results of the chart above might not be what I am looking for.
Not a problem, as I can easily create a new column in the `df_commenters` DataFrame and store the conversion from UTC to Brazilian time in it:
```python
 # Create a column to show when a a comment was created in Brazilian time
df_commenters['created_at_br'] = df_commenters.created_at_utc.dt.tz_localize('UTC').dt.tz_convert('America/Sao_Paulo')
```
Let's plot again and see the difference:
```python
df_commenters.created_at_br.dt.hour.value_counts().sort_index().plot(kind='bar', figsize=(12,6))
```
![Bar plot]({{ '/assets/images/posts/instagram-data-analysis/commenters_hour_brt_plot_bar.png'|absolute_url}}){: .align-center}

