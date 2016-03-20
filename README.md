# Personality Insight App Overview

The Personality Insight app demonstrates a simple, reusable Python web application that uses Watson **Personality Insights** and **Insights for Twitter** service in Bluemix. If you follow the steps carefully, you'll learn the followings:

* How to use Bluemix and cloud foundry (cf) comman line interfaces (cli)?
* How to create application and services via cf cli?
* How to bind a service to an application?
* How to remove application and services via cf cli?
* How to run the Python code locally?
* How to retrieve tweets from *Insights for Twitter Service*?
* How to retrieve insights from *Watson Personality Insights* service?


## Prerequisites

1. Install [cloud foundry command line interface] (https://github.com/cloudfoundry/cli/releases)
2. Install [Bluemix command line interface] (http://clis.ng.bluemix.net/ui/home.html)


## Run the app in Bluemix

1. Download the code to your computer: `git clone git@github.com:bulutmf/PerInsight.git`.
2. cd into the app directory: `cd PerInsight`
3. Open the `manifest.yml` file and edit `name` and `host` fields to something else. Don't use the **PerInsight** as it is already taken. Your host name should be unique in Bluemix, i.e. not taken by someone else. Take a note of the **[APP_NAME]**
4. Set the api end point to dedicated bluemix: `cf api https://api.w3ibm.bluemix.net`
5. Login: `cf login`.
6. Push the application code to Bluemix: `cf push`
7. Create a **Insights for Twitter** service: `cf create-service twitterinsights Free twitter-insights-service`. We named the service *twitter-insights-service*, you can name it whatever you want, however we use the same name to bind the service to the application in next step.
8. Bind *twitter-insights-service* to your application *[APP_NAME]*: `cf bind-service [APP_NAME] twitter-insights-service`.
9. Create a **Personality Insights** service: `cf create-service personality_insights tiered personality-insights-service`. We named the service *personality-insights-service*, you can name it whatever you want, however we use the same name to bind the service to the application in next step.
10. Bind *personality-insights-service* to your application *[APP_NAME]*: `cf bind-service [APP_NAME] personality-insights-service`
11. Restage your application so that env variables can be accessed from the application: `cf restage [APP_NAME]`

## Run the app locally

**Note**: You should first follow above steps and run the app in Bluemix.

1. [Install Python][] (if you haven't installed it yet). Note that the app uses Python version 3.5.
2. Download the code to your computer: `git clone git@github.com:bulutmf/PerInsight.git`.
3. cd into the app directory: `cd PerInsight`
4. Install the required Python packages: `pip3.5 install -r requirements.txt`
5. Open the `server.py` file and set the values for `TWITTER_USERNAME`, `TWITTER_PASSWORD`, `PERSONALITY_INSIGHT_USERNAME` and `PERSONALITY_INSIGHT_PASSWORD`. You can find these values from Bluemix: Dashboard => Click on the app => Look for `Show Credentials` link on each of the service tiles.
6. Run `python3.5 server.py`
7. Access the running app in a browser at: [http://localhost:8000] (http://localhost:8000)

[Install Python]: https://www.python.org/downloads/


## Clean up

1. Run this command to get [APP_NAME]: `cf apps`
2. Delete: `cf delete [APP_NAME]` (replace [APP_NAME] with your application name).
3. Run this command to get [SERVICES_NAME] for both Insights for Twitter service (*[SERVICE_1]*) and Watson Personality Insights service ((*[SERVICE_2]*)): `cf services`
4. Delete Insights for Twitter Service: `cf delete-service [SERVICE_1]`
5. Delete Personality Insight Service: `cf delete-service [SERVICE_2]`


## How it works?

The app provides a REST API end point `analyze?twitterHandle=fatih_bulut` which accepts `GET` requests with user's twitter handle as a parameter. Once the twitter handle is retrieved the app will function in three steps: *Retrieving tweets from Twitter*, *Retrieving insights from Watson service* and finally *Presenting insights to end user*. Backend code is in `server.py` file whereas the front end code is in `static/index.html` file. Below shows the `analze` implementation.

```Python
def analyze(self, twitterHandle):
      tweets = self.getTweets(twitterHandle)
      jsonContentItems = self.tweetsToContentItem(tweets)
      insights = self.getInsights(jsonContentItems)
      return insights
```

### Retrieving tweets from Twitter

In order to get the tweets from Twitter, the app uses the *Insights for Twitter* service of Bluemix. Once the service is created and bound to the application, credentials such as username, password and url to be queried will appear on the service tile. Below shows the function of `server.py` which is used to retrieve the tweets. Basically, it send a http get requests to Twitter service with parameters as: `q=from:fatih_bulut&lan=en&size=20`. The result returned from Twitter service is a JSON object. For more details see: [Insights for Twitter service documentation] (https://console.ng.bluemix.net/docs/services/Twitter/index.html#twitter)

```Python
def getTweets(self, twitterHandle):
      payload = {"q": "from:" + twitterHandle, "lan": "en", "size": self.NO_OF_TWEETS_TO_RETRIEVE}
      response = requests.get("https://cdeservice.mybluemix.net:443/api/v1/messages/search", params=payload, auth=(self.TWITTER_USERNAME, self.TWITTER_PASSWORD))
      tweets = json.loads(response.text)
      return tweets
```


### Retrieving insights from Watson service

Once the tweets are retrieved, next step would be to give tweets to *Watson Personality Insights* service and retrieve the insights. However, we first need to change the JSON format of tweets so that it conforms to the JSON input format that the Watson Personality Insights service is expecting. Below shows how we change the format.

```Python
def tweetsToContentItem(self, tweets):
      contentItems = []
      for tweet in tweets["tweets"]:
          item = {
              "id": tweet["message"]["id"],
              "userid":tweet["message"]["actor"]["id"],
              "created": "",
              "updated": "",
              "contenttype": "text/plain",
              "charset": "UTF-8",
              "language": "en-us",
              "content": tweet["message"]["body"],
              "parentid": "",
              "reply": False,
              "forward": False
          }
          contentItems.append(item)
      return json.dumps({"contentItems": contentItems})
```

Once the format is appropriate, we send an http post request to Watson service with the tweets as JSON body. Retuned result is what we return from the `analyze` end point. For more information about the Watson Personality Insights Service please see the [documentation] (https://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/personality-insights.html).

```Python
def getInsights(self, jsonContentItems):
      headers = {'content-type': 'application/json'}
      response = requests.post("https://gateway.watsonplatform.net/personality-insights/api/v2/profile", headers=headers, data=jsonContentItems, auth=(self.PERSONALITY_INSIGHT_USERNAME, self.PERSONALITY_INSIGHT_PASSWORD))
      return response.text
```

### Presenting insights

The last step would be the present personality insights results to the end user. `static/index.html` file used to serve that purpose. Basically, it retrieves the twitter handle from the user and call `analyze` end point in our backend to retrieve the personality insights results. Once it's retrieved, it parses it and present to the user.
