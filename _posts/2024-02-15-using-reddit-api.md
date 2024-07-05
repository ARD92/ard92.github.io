---
layout: post
title:  Using Reddit APIs 
tags: python NLP
---

I dont know about others, but I spend insane amount of time on reddit. While using the app is fun, there may be situations where we may want to use APIs to gather data. Multiple projects for educational/professional purposes can be done. This blog aims to demonstrate how to interact with reddit using the APIs 

There are few things needed to interact with reddit 
1. praw python library
2. create a client secret in reddit 
3. use the client id, user agent, password, client secret 

First, login to your reddit account and navigate to [https://www.reddit.com/prefs/apps/](https://www.reddit.com/prefs/apps/) and create an app.
This will provide
- client_id (which will be under the app name as an alphanumeric string)
- user_agent (which will be the application name which  you provide to create)
- username (your login user) 
- password ()
- client_secret (This will get generated at the end of app creation)

Use these as part of your praw login params.

```
def login(clientid, secret, username, password, useragent):
        reddit = praw.Reddit(
                client_id=clientid,
                client_secret=secret,
                username=username,
                password = password,
                user_agent = useragent
        )
        return reddit
```

Let us look through an example 
### Subscribe and unsubscribe to subreddits 
In this example, we list out all the subreddits and unsubscribe 
```
def unsubscribe(reddit, unsub):
        for i in unsub:
                try:
                        reddit.subreddit(i).unsubscribe()
                        print("unsubscribed {}".format(i))
                except prawcore.exceptions.NotFound:
                        continue
```

Full code available [here](https://github.com/ARD92/reddit_scraper)

### Simple sentiment analysis using NLTK
In this example, we are performing sentiment analysis and classifying responses from spectrum mobile subreddit as postive, negative or neutral.

#### Required modules
```
pip install nltk
pip install praw

import nltk
nltk.download('all')
```

#### Full code performing a sentiment analysis
```
import praw
import pprint
import json
import argparse
import nltk
import pymongo

from requests import Session
from nltk.sentiment.vader import SentimentIntensityAnalyzer
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize
from nltk.stem import WordNetLemmatizer

"""
Login details. Reddit script app must be 
created in order to get the values for params
"""
def login(clientid, secret, username, password, useragent):
	reddit = praw.Reddit(
		client_id=clientid, 
		client_secret=secret,
		username=username,
		password = password,
		user_agent = useragent
		)
	return reddit

"""
Read subreddit. Can be single or multiple
"""
def readSubreddit(reddit, sub_reddit):
	reddit.read_only = True
	subreddit = reddit.subreddit(sub_reddit)
	print(subreddit.display_name)
	return subreddit


"""
currently parses only titles and not each comment
freq can be "week, month, year, day, hour"
"""
def parseTopPosts(subreddit, freq, limit ):
	posts = []
	for submission in subreddit.top(time_filter=freq, limit=limit):
		data = {}
		data["title"] = submission.title
		data["upvotes"] = submission.score
		data["timestamp"] = submission.created
		if submission.author == "None":
			data["user"] = "user deleted"
		else:
			data["user"] = str(submission.author)
		posts.append(data)
	return posts


def printPosts(posts):
	pp = pprint.PrettyPrinter(indent=4)
	pp.pprint(posts)
	print("------")

"""
Preprocess text 
1. tokenize
2. removing stopwords
3. lemmatization 
"""
def preProcess(posts):
	processed_list = []
	for post in posts:
		final = {}
		tokens = word_tokenize(post["title"].lower())
		filtered_tokens = [token for token in tokens if token not in stopwords.words('english')]
		#print(filtered_tokens)
		lemmatizer = WordNetLemmatizer()
		lemmatized_token = [lemmatizer.lemmatize(token) for token in filtered_tokens]
		processed_text = ' '.join(lemmatized_token)
		#print(processed_text)
		final["title"] = processed_text
		final["time"] = post["date"]
		final["upvotes"] = post["upvotes"]
		final["user"] = post["user"]
		processed_list.append(final)
	return processed_list

"""
use nltk vader to get a sentiment score
"""
def sentiAnalyze(processedTxt):
	sia = SentimentIntensityAnalyzer()
	return sia.polarity_scores(processedTxt)

def insertToMongo(table_data):
    #print(table_data.items())
    
    try:
        dbclient = pymongo.MongoClient(host="s2-mongo", port=27017)
        if 'selector' in dbclient.list_database_names():
            print('selector exists')
            pass
        db = dbclient['selector']
        dbcoll = db["Reddit"]
        print(dbcoll.insert_one(table_data))
        dbclient.close()
    except Exception as e:
        print(f"Something went wrong: {str(e)}")
    return


if __name__ == "__main__":
	#To_add: argument for login file and subreddit input
	with open('login.json', 'r') as f:
		fdata = json.load(f)

	fr = open('result_all.json', 'a+')
	fw = open('result_negative.json', 'a+')

	reddit = login(fdata["client_id"], fdata["client_secret"], fdata["username"], fdata["password"], fdata["user_agent"])
	subreddits = ["Spectrum", "SpectrumMobile"]

	all_list = []
	final_list = [] 
	for s in subreddits:
		test = []
		sub = readSubreddit(reddit, s)
		posts = parseTopPosts(sub, "month", 1000)
		#processed = preProcess(posts)
		for i in posts:
			pfinal= {}
			val = sentiAnalyze(i["title"])
			pfinal["title"] = i["title"]
			if val["compound"] > 0:
				pfinal["sentiment"] = "Positive"
			elif val["compound"] == 0:
				pfinal["sentiment"] = "Neutral"
			elif val["compound"] < 0:
				pfinal["sentiment"] = "Negative"
			pfinal["timestamp"] = i["timestamp"]
			pfinal["upvotes"] = i["upvotes"]
			pfinal["user"] = i["user"]
			pfinal["subreddit"] = s
			insertToMongo(pfinal)
			all_list.append(pfinal)
			if val['compound'] < 0:
				#printPosts(pfinal)
				test.append(pfinal)
		final_list.append(test)
		#print("total posts: ", len(processed))
		#print("total negative:", len(test))
        
	#fr.write(json.dumps(all_list, indent=4))
	#fw.write(json.dumps(final_list, indent=4))
```

## References
- [api docs](https://www.reddit.com/dev/api)
- [reddit apps](https://www.reddit.com/prefs/apps/)


