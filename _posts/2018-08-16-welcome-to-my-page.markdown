---
layout: post
title: "Exploring your Instagram data"

---

In this post I want to talk a bit about how exploring your own Instagram account and generate interesting insights.
I will be using Python (Jupyter Notebook) for this, with the following packages:
- pandas
- LevPasha/Instagram-API-python (unofficial Instagram API)


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
Once executed, you should receive this return message:
> Login success!




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

```
