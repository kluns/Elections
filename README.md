# Tweet for President

A real time sentiment analysis of the 2016 Presidential Candidates on Twitter.

By pulling in tweets and analyzing them we can get an idea of how people are feeling, in real time, about a candidate. This analysis gives an interesting look into how people on Twitter are reacting to current events and news stories pertaining to the election.

This project consists of four main parts:
 - [streaming of tweets from Twitter](https://github.com/CUBigDataClass/Elections#streaming)
 - [sentiment analysis of tweets](https://github.com/CUBigDataClass/Elections#sentiment)
 - [web server to format data and handle connections](https://github.com/CUBigDataClass/Elections#server)
 - [web site to display streams of data](https://github.com/CUBigDataClass/Elections#web)

## Streaming

We stream tweets in real time using [Twitter's API](https://dev.twitter.com/streaming/public) and [tweepy](https://github.com/tweepy/tweepy). These tweets are filtered on terms related to the candidates, and formatted with the candidate name and tweet status. The data is then passed off to a Kafka queue, which handles all of the communication between different pieces of the project.

## Sentiment

After being filtered, an analyzer reads off of the Kafka queue and tags the tweet `pos` or `neg`, before passing it back into a different Kafka queue.

The analyzer is built on top of [nltk's naive bayes](http://www.nltk.org/book/ch06.html) classifier. It's trained on a dataset of 800,000 tweets that have been tagged `pos` or `neg` and uses this data as the source of truth. We experimented with more complex classifiers, however naive bayes was a good fit for this project, balancing accuracy and speed.

A classifier can be trained using the `trainer.py` script, which may take a few minutes to complete (you will need the dataset). After a classifier is generated, it can be used by an analyzer to tag tweets. The default classifier should be placed at `sentiment/default_classifier.pickle`, however you can configure the analyzer to use any classifier if you wish.

> note: classifiers are generally very large and any program using one will need some time to load it into memory.

To test a classifier manually on tweets or phrases, use the `cli.py` script provided. You can feed it phrases and it will classify them, which is helpful to debug your classifier. An example may look as follows:

```
$ python3 cli.py
? i love this
pos
? i hate this
neg
```

## Server

The webserver is built to handle multiple connections and stream data to any connections listening. It reads off of the Kafka sentiment queue, does some simple formatting and calculations and sends data to the clients. This messaging to the clients takes advantage of [server sent events](https://www.wikiwand.com/en/Server-sent_events) and broadcasts a message out every 500ms, giving it a streaming nature.

## Web

The web site takes in data from the server using [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) and queues it to display in a streaming graph. The graphs are powered by [D3](https://d3js.org/) and display a % positive and % negative for each candidate. The graphs maintain their own sense of 'time' and transition data on and off of the graph. This approach helps buffer inconsistencies in data from the server, and a line is drawn through the data by interpolating the space in between points. Putting all of these pieces together we get a smooth flow on our graphs.

## Running Locally

### Installation and Setup

Clone this repository locally
```sh
$ git clone <repo>
$ cd Elections
```

Install dependencies
```sh
# activate virtualenv
$ pip install -r requirements.txt
$ go get github.com/Shopify/sarama
```

Install Kafka ([directions here](https://kafka.apache.org/quickstart#quickstart_download))

Create a `.env` file at `Config_Utils/.env` with your twitter access credentials ([see here for getting access](https://dev.twitter.com/oauth/overview)). It must follow the form below
```
TWITTER_CONSUMER_KEY="<key>"
TWITTER_CONSUMER_SECRET="<secret>"
TWITTER_ACCESS_TOKEN_KEY="<key>"
TWITTER_ACCESS_TOKEN_SECRET="<secret>"
```

Download the [trained classifier](https://www.dropbox.com/s/0j472jfhw3crvsf/default_classifier.pickle?dl=0) into `sentiment/default_classifier.pickle`
> You can also use your own dataset, using `trainer.py`, however the script is fairly specific to the dataset we used. It would be a nice improvement to have this generalized and/or better documented

### Running

This project has many moving parts. You'll need to have them all running together, we use tmux for this, but pick your favorite option.

First you'll need to start up Kafka, refer to their [documentation](https://kafka.apache.org/quickstart#quickstart_startserver) for instructions.

Next you'll need to start the pipeline. Tweets flow from extract.py --> analyze.py --> go server

```sh
$ python3 extract.py
# in new tmux session or terminal tab/window
$ python3 analyze.py
# in new tmux session or terminal tab/window
$ cd website/
$ go run main.go conn_manager.go
```

Visit localhost:9092 in your browser

## License

MIT
