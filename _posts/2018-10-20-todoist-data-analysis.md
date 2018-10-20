---
layout: post
title: "Analyzing Todoist data with Python"
excerpt: "If you use Todoist, you may have already wondered how to put all your task data in a single place and generate meaningful reports on top of it. Here is how."
image: 
  path: /assets/images/posts/todoist-data-analysis/todoist-post.png
  thumbnail: /assets/images/posts/todoist-data-analysis/todoist-post.png
categories:
 - data analysis
tags:
 - api
 - python
 - visualization
 - pandas
 - jupyter
---
**Todoist** is a great task management application, and personally my favorite (after testing at least 5 of such apps), which I have been using on a daily basis for the past couple of months.  
The app is really easy to use, with a simple interface, yet very powerful. It is also available on basically every existing platform (iOS, Android, Windows, Mac, Web...)

By the way, if you don't have an account yet, you can register using [this link](https://todoist.com/r/gustavo_saidler_wqcpmz){:target="_blank"}, and you'll get 2 months of premium for free.
{: .notice--danger}
<figure style="width: 220px" class="align-right">
  <img src="{{ '/assets/images/posts/todoist-data-analysis/todoist-report.png' | absolute_url }}" alt="">
  <figcaption>Todoist reporting example.</figcaption>
</figure> 
One of the main things that drew my attention in this app is its reporting feature, which allows you to check completed tasks from the past days and weeks, giving the idea of how productive you are.  

Even beign really useful, there is no easy way to check and visualize further details about your historical taksks, and hence my ideia of expanding this feature.

Fortunately, there is an official **Todoist Sync API**, and even better, an official **Todoist Python library** to interact with it, called `todoist-python`. Both were developed by **Doist** and are publicly available.  
The Sync API documentation can be found [here](https://developer.todoist.com/sync/v7/#overview){:target="_blank"}, and the `todoist-python` library documentation [here](https://todoist-python.readthedocs.io/en/latest/){:target="_blank"}.  
You can install the library via **pip**: `pip install todoist-python`.

Along with the `todoist-python`, I will use `pandas`  in a [Jupyter](https://jupyter.org/){:target="_blank"} environment for this demonstration.

# Let's get started! 

The first thing to do is getting your API token, which is needed to login to your account. It can be easily found inside the Todoist app, you just have to go to **Settings -> Integrations, and scroll down to API token**. It is a big text of letters and numbers.

Then, to the code:

```python
from todoist.api import TodoistAPI
import pandas as pd

token = '6yahsfd7a40efd21b108201781a14379201570d12' # I made this up

# create a TodoistAPI object with the token, which we store to the api variable
api = TodoistAPI(token)

# Fetches the latest updated data from the server.
api.sync()
```
### Creating the "Projects" DataFrame

We are now logged in to Todoist, and can start retrieving data from it.  
The first data I'll retrieve are my projects, which are available in the `api.state['projects']` object:
```python
# .data attribute retrieves a python dictionary rather than todoist.models.Project
projects = [project.data for project in api.state['projects']] 

# I can easily create a DataFrame on the 'projects' list of dicts
df_projects = pd.DataFrame(projects)

df_projects.head()
```
![DataFrame]({{ '/assets/images/posts/todoist-data-analysis/df_projects.png'|absolute_url}}){: .align-center}

### Creating the "Tasks" DataFrame

Next, I will retrieve all the tasks I have created, available at the `api.state['items']` object. It is the same idea as in the step above:
```python
# Same as above
tasks = [task.data for task in api.state['items']]

df_tasks = pd.DataFrame(tasks)

df_tasks.head()
```
![DataFrame]({{ '/assets/images/posts/todoist-data-analysis/df_tasks0.png'|absolute_url}}){: .align-center}

This DataFrame is a bit more detailed, and it deserves a bit of handling before continuing:

```python
# Convert Date strings (in UTC by default) to datetime and format it 
df_tasks['date_added'] = pd.to_datetime(
	(pd.to_datetime(df_tasks['date_added'], utc=True)
	.dt.tz_convert('Europe/Budapest') # my current timezone
	.dt.strftime("%Y-%m-%d %H:%M:%S"))) # easier to handle format

df_tasks['due_date_utc'] = pd.to_datetime(
	(pd.to_datetime(df_tasks['due_date_utc'], utc=True)
	.dt.tz_convert('Europe/Budapest')
	.dt.strftime("%Y-%m-%d %H:%M:%S")))

df_tasks['date_completed'] = pd.to_datetime(
	(pd.to_datetime(df_tasks['date_completed'], utc=True)
    .dt.tz_convert('Europe/Budapest')
    .dt.strftime("%Y-%m-%d %H:%M:%S")))

# Many of my tasks are recurrent, and I want to identify them 
# by searching for the string 'every' in the 'date_sting' field
df_tasks['recurring'] = (df_tasks['date_string'].str.contains('every')
	.map({True: "Yes", False: "No"}))

# The 'project_id' is present in this DataFrame, but I want the actual project name
# So I create a 'mapper' consisting of 'project_id: name' (from df_projects)
# and map it to df_tasks (pd.merge could also have been used)
map_project = dict(df_projects[['id', 'name']].values) 
df_tasks['project_name'] = df_tasks.project_id.map(map_project)

# Check new date formats/new fields
df_tasks.head()
```
![DataFrame]({{ '/assets/images/posts/todoist-data-analysis/df_tasks1.png'|absolute_url}}){: .align-center}

### Creating the "Tasks activities" DataFrame
Now it's time for perhaps the most important dataset: the [**Activity Log**](https://developer.todoist.com/sync/v7/#activity){:target="_blank"}  
The activity log makes it easy to see everything that is happening across projects and tasks. It logs whenever a task is added, updated, completed...  

**Activity log is only available for Todoist Premium.**
```python
# The API limits 100 activities to be retrieved per call, so a loop is needed

# Items are retrieved in descending order by date.
# offset indicates how many items should be skipped
activity_list = []
limit = 100
offset = 0

while True:
    # API call, retrieving between 0 and 100 activities
    activities = api.activity.get(limit=limit, offset=offset)
    
    if not activities: # if it returns an empty list, get out of the loop
        break
    
    activity_list.extend(activities)
    
    # set offset to skip the number of rows that were just returned
    offset += limit 
 ```
 The `activity_list` is now a single list containing multiple dictionaries, and each dictionary represents one activity. The dicts contain the same keys, but the `extra_data` key is a nested object, which can vary between activities.  Therefore, I have to handle it first before creating the DataFrame:
 ```python
# Put 'extra_data' dict on parent dict level, then deletes it
for activity in activity_list:
    activity.update(activity['extra_data'])
    del activity['extra_data']

# Creates DataFrame from activity_list
df_activity = pd.DataFrame(activity_list)

df_activity.head()
 ```
![DataFrame]({{ '/assets/images/posts/todoist-data-analysis/df_activity.png'|absolute_url}}){: .align-center}

This DataFrame is also quite detailed, so I will perform steps similar to the ones performed on the "Tasks" DataFrame:
```python
# Convert Date strings (in UTC by default) to datetime and format it 
df_activity['due_date'] = pd.to_datetime(
    (pd.to_datetime(df_activity['due_date'], utc=True)
     .dt.tz_convert('Europe/Budapest')
     .dt.strftime("%Y-%m-%d %H:%M:%S")))

df_activity['event_date'] = pd.to_datetime(
    (pd.to_datetime(df_activity['event_date'], utc=True)
     .dt.tz_convert('Europe/Budapest')
     .dt.strftime("%Y-%m-%d %H:%M:%S")))

df_activity['last_due_date'] = pd.to_datetime(
    (pd.to_datetime(df_activity['last_due_date'], utc=True)
     .dt.tz_convert('Europe/Budapest')
     .dt.strftime("%Y-%m-%d %H:%M:%S")))

# Set DataFrame index as the EVENT_DATE (will make it easier to plot later)
df_activity = df_activity.set_index('event_date')

# Add project name to DataFrame, using the mapper from before
df_activity['project_name'] = df_activity.parent_project_id.map(map_project)
```

# Visualizing the data

Now that we have all DataFrames, it is time for some visualization. Here are the questions I came up with:
1. What are my Todoist statistics (sums, averages...)?
2. How is my productivity over time?
3. How are my task activities distributed among projects?
4. On which day of the week am I most productive?


### 1. What are my Todoist statistics (sums, averages...)?
```python
# Get DAILY AVERAGE of each event type
df_daily_event_avgs = (df_activity.groupby([df_activity.index,'event_type']).
                 size()
                 .unstack()
                 .resample('D')
                 .sum()
                 .mean()
                )

# Get WEEKLY AVERAGE of each event type
df_weekly_event_avgs = (df_activity.groupby([df_activity.index,'event_type']).
                 size()
                 .unstack()
                 .resample('W')
                 .sum()
                 .mean()
                )

# Get SUM of each event type
df_event_sums = df_activity.groupby('event_type').size()

#--------------------------------------------------
# Profile info
premium = api.state['user']['premium_until']
karma = api.state['user']['karma']
daily_goal = api.state['user']['daily_goal']
weekly_goal = api.state['user']['weekly_goal']

# Dates
start_date = str(df_activity.index[-1])
duration = str(df_activity.index[0] - df_activity.index[-1])

# Averages
daily_avg_adds = df_daily_event_avgs['added']
daily_avg_completes = df_daily_event_avgs['completed']
daily_avg_updates = df_daily_event_avgs['updated']

weekly_avg_adds = df_weekly_event_avgs['added']
weekly_avg_completes = df_weekly_event_avgs['completed']
weekly_avg_updates = df_weekly_event_avgs['updated']

# Sums
sum_adds = df_event_sums['added']
sum_completes = df_event_sums['completed']
sum_updates = df_event_sums['updated']
```
Printed:
```
***************     My Todoist statistics     ***************

Started using at............................: 2018-07-20 21:59:03
Used so far.................................: 79 days 21:02:39
Premium until...............................: Sat 25 May 2019 13:46:52 +0000
Karma points................................: 10376.0


Daily tasks goal............................: 9
Weekly tasks goal...........................: 50

-----------------------------------------------

Total added tasks...........................: 262
Total completed tasks.......................: 871
Total updated (re-scheduled) tasks..........: 372

-----------------------------------------------

Average tasks added per day.................: 3.23
Average tasks completed per day.............: 10.75
Average tasks updated (re-scheduled) per day: 4.59


Average tasks added per week.................: 20.15
Average tasks completed per week.............: 67.0
Average tasks updated (re-scheduled) per week: 28.62
```

### 2. How is my productivity over time?

```python
# Create DF of events per day
df_event_by_day = (df_activity.groupby([df_activity.index,'event_type'])
                   .size()
                   .unstack()
                   .resample('D')
                   .sum())

# Plot completed tasks
daily_activities = (df_event_by_day[['completed']]
                     .plot(figsize=(15,8),
                           lw=3
                  ))

daily_activities.set_title('Completed tasks over days', fontsize=20)

# Add horizontal line with Average completed tasks
daily_activities.axhline(daily_avg_completes, linestyle='--', color='g', label='daily average')
daily_activities.axhline(daily_goal, linestyle=':', color='y', label='goal')
daily_activities.legend(fontsize=12)
```
![DataFrame]({{ '/assets/images/posts/todoist-data-analysis/prod-over-time.png'|absolute_url}}){: .align-center}

### 3. How are my task activities distributed among projects?

```python
df_event_by_project = df_activity.groupby(['project_name','event_type']).size().unstack()

project_counts = (df_event_by_project[['added','completed','updated']]
                     .plot(title='Task activities per project', 
                           figsize=(12,8), 
                           kind='barh',
                           fontsize=12, 
                           width=.7)
                  )

project_counts.set_ylabel('Project Name')
project_counts.set_xlabel('Tasks')
```
![DataFrame]({{ '/assets/images/posts/todoist-data-analysis/act-among-proj.png'|absolute_url}}){: .align-center}

### 4. On which day of the week am I most productive?

```python
df_event_by_weekday = df_activity.groupby([df_activity.index.dayofweek,'event_type']).size().unstack()

weekday_activities = (df_event_by_weekday[['added','completed','updated']]
                     .plot(figsize=(13,8),
                           lw=3, 
                           marker='.', 
                           markersize=12,
                           grid=True
                  ))

weekday_activities.set_xlabel('Weekday')
weekday_activities.set_ylabel('Tasks')
weekday_activities.set_xticklabels(
    [0,'Sun','Mon','Tue','Wed','Thu','Fri','Sat']) # 0 is a workaround

weekday_activities.legend(fontsize=12)
weekday_activities.set_title('Activities vs Day of Week', fontsize=20)
```
![DataFrame]({{ '/assets/images/posts/todoist-data-analysis/tasks-weekday.png'|absolute_url}}){: .align-center}


There are plenty more stuff to be analyzed within Todoist, but I will leave it to another opportunity. Perhaps when I have more data since I am using the app for about 3 months only.