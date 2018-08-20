---
layout: post
title: "Data analysis: Your Instagram data"

---

In this post I want to talk a bit about how exploring your own Instagram account and generate interesting insights.
I will be using [iPython - Jupyter Notebook](https://jupyter.org/) for this, with the following packages:
- pandas
- [LevPasha/Instagram-API-python](https://github.com/LevPasha/Instagram-API-python) (unofficial Instagram API)


First of all, let's import the necessary packages that will be used along this demonstration:

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

Now that we have all the posts, let's retrieve all the "post likers" and see who are the users who like your posts the most:
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
This should take some minutes, depending on the amount of posts you have. An approximate wait time will be displayed
```
wait 3.0 minutes
``` 
You will receive `done` when it finished. 
`likers` will also be a list of dictionaries, and should have the same length as `my_posts`. Inside each dictionary, you will find the key `users`, which contain all the users that liked a specific post.

It's time to use the powerful `pandas` package and structure this data a little bit. I'll use [json_normalize](https://pandas.pydata.org/pandas-docs/version/0.22/generated/pandas.io.json.json_normalize.html) within pandas.io.json:
```python
# Normalize likers by getting the 'users' list and the post_id of each like
df_likers = json_normalize(likers, 'users', ['post_id'])	
```
With this, we have a pandas DataFrame `df_likers`, where each row represents a single like. 
We can now get some interesting numbers. For example: 

##Top 10 likers of my Instagram account:

```python
df_likers.username.value_counts()[:10]
```
```
leassxch           71
lucaszcssching     58
krlosawqicius      52
jeplqiwnini        50
gukashwberis       44
haylasnqjwnmann    42
mateustcqqwelin    41
joanadaoaikj       40
fpridcdsdwac       39
sarakquwhjk        38
Name: username, dtype: int64
```
Hmm, "leassxch" is the person who liked my post the most: out of 90 posts, he liked 71 of them!

Let's plot the distribution of this Top 10:
![image-center]({{ '/assets/images/likers_plot.jpg'|absolute_url}}){: .align-center}

