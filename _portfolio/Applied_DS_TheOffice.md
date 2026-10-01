---
title: "Applied Data Science to The Office TV Show"
excerpt: "Exploring The Office (U.S) transcript data with `pandas`, `nltk`, `NetworkX` <br/><img src='/images/The_Office_imgs/Emotion_banner.png'>"
collection: portfolio
---

Exploring The Office (U.S) transcipt data with `pandas`, `nltk`, `NetworkX`

![Alt text](/images/The_Office_imgs/Emotion_banner.png)
## Motivation 

After completing a series of courses in Applied Data Science in Python from University of Michigan, I wanted to put the skills and knowledge that I have learned into a fun project.
The Office (U.S) was always around during my college years at Oregon State. My roommate had it playing in the living room all the time. Somehow Prison Mike was a popular sticker on my campus along with DutchBros. Some of my friends even named their cats after Dwight and Kevin from the show.
Then it hit me. I should apply my newfound skills to see the insight on what the data says about the show. From there, I knew what my fun project would be.
Let’s get started!

## Exploring the data
I downloaded the dialogue/transcript from [Kaggle](https://www.kaggle.com/nasirkhalid24/the-office-us-complete-dialoguetranscript). All the code is written in `python` using the popular libraries `pandas`, `matplotlib`, `nltk`, `NetworkX`, `NRClex`(TextBlob)

## The characters
While there are many frequent characters, there are 4 in particular that are arguably the main cast: Michael, Dwight, Jim, Pam. Some people argue that certain characters are more prominent, however let’s see what the data actually says by plotting the lines spoken by the characters.

![line_count](/images/The_Office_imgs/Line_count_analysis.png)

The graph clearly shows 4 main characters, which as stated above are Michael, Dwight, Jim and Pam. Creed, albeit a frequent face in Dunder Mifflin, had the least amount of spoken lines of the 20 characters. He known for being low-key and always covering his true identity.

![most_to_say](/images/The_Office_imgs/Word_count_analysis.png)

The characters have on average around 6 to 10 words in each line. Michael’s, Andy’s and Nellie’s words in each line vary a lot more than other. If you have watched the show, you already know that Michael and Andy usually talk a lot. But what about Nellie? Nellie was disliked by many people at Dunder Mifflin at first, but later she was on good terms with most of them. Looking further, these 3 characters were the manager at Dunder Mifflin at some point. Maybe that is why.

```python
# get 20 characters
main_characters = list(office['speaker'].value_counts().index[:20])
df_word_count = office[np.where(office.speaker.isin(main_characters), True, False)]\
    .groupby('speaker')\
    .agg({'word_count' : ['max', 'count','median']})
df_word_count.columns = ['max', 'count', 'med']
df_word_count.reset_index(inplace=True)
df_word_count = df_word_count.sort_values('max', ascending=False)
df_word_count = df_word_count.sort_values('count', ascending=False)

speaker_line = dict((office['speaker'].value_counts())[:20])
df_speaker_line = pd.DataFrame(df_speaker_line.items(), columns = ['speaker','num_line'])
```

I utilized the `value_counts()`function from pandas to get top 20 characters’s total number of line and number of words in each line. With aggregation function `agg` , I just need to pass in parameters to get maximum and median to get statistic data for box plot. I chose to go with `seaborn` for graphing due to it’s fantastic colors map. I learned how to use `pandas` for data cleaning and `seaborn` for graphing from Introduction to Data Science and Applied Plotting, Charting & Data Representation course.

## Who speaks to whom?
One way to see the relationship between characters is by simply plotting who talk to whom. People tend to talk to someone they like and don’t talk much to people they dislike.

![Interaction](/images/The_Office_imgs/Interaction_NetworkX_flatteern.png)

As the main character and also the manager of Dunder Mifflin, Michael has the most interactions with almost everyone in the show ( minus Nellie). This is illustrated by the thicker lines starting at Michael and flowing to other characters in various directions. Looking at the characters who has relationship in the show, their line connections are signification thicker (Michael — Jan, Michael — Holly, Ryan — Kelly, Andy — Erin). Creed once again proved that he is a hidden puppet master.

```python
import networkx as nx
# get 20 characters
main_characters = list(office['speaker'].value_counts().index[:20])
# create networkx object
G = nx.DiGraph()
# %% Get coversation info betwwen characters
scene_before = ""
episode_id_before = -1
for i in range(len(office)):
    # check if episode and location of text is the same
    if scene_before != office["scene"].iloc[i] or office["episode_id"].iloc[i] != episode_id_before:
        scene_before = office.iloc[i]["scene"]
        episode_id_before = office.iloc[i]["episode_id"]
        continue
    scene_before = office.iloc[i]["scene"]
    episode_id_before = office.iloc[i]["episode_id"]
    # get characters
    character1 = office["speaker"].iloc[i]
    character2 = office["speaker"].iloc[i+1]
    # fail check for character not in the interested list
    if character1 not in main_characters or character2 not in main_characters:
        continue

    sorted_characters = (character1, character2)
    try:
        # add +1 to weight if characters have conversation on the same sence
        G.edges[sorted_characters]["weight"] += 1
    except KeyError:
        G.add_edge(sorted_characters[0], sorted_characters[1], weight=1)

def plot_fig_para(G):
    plt.figure(figsize=(40, 40))
    pos = nx.circular_layout(G)

    edges = G.edges()

    ################
    colors = [G[u][v]['Weight']**0.09 for u, v in edges]

    weights = [G[u][v]['Weight']**0.45 for u, v in edges]

    cmap = matplotlib.cm.get_cmap('Blues')

    nx.draw_networkx(G, pos, width=weights, edge_color=colors,
                     node_color="white", edge_cmap=cmap, with_labels=False)

    labels_pos = {name: [pos_list[0], pos_list[1]-0.04] for name, pos_list in pos.items()}
    nd = nx.draw_networkx_labels(G, labels_pos, font_size=50, font_family="sans-serif",
                                 font_color="#000000", font_weight='bold')

    ax = plt.gca()
    ax.collections[0].set_edgecolor('#000000')
    ax.margins(0.25)
```

What the code doing is that it check for characters who are in the same scene then add +1 to their weight. When character A “interact” with character B, it creates an edge from A to B. “interact” in this case means when character A’s speaking line is right before character B’s speaking line. This method is not a 100% accurate, but it gives a good estimation.

If you are curious about the Network Science and the powerful library NetworkX, I strongly recommend “A First Course In Network Science” book by Professor Menczer, Professor Fortunato and Dr. Davis, from Indiana University. It provides great fundamental knowlege with handon examples.

For those who is curious about the number of “interaction”, here is the heat map.
![Heat_map](/images/The_Office_imgs/Heatmap_whospk.png)

## Sentiment Analysis
Sentiment Analysis is a technique in Natural Language Processing to determine whether the data is neutral, positive or negative. In this case, we are performing sentiment analysis to the words spoken by the characters. The model used is VADER (Valence Aware Dictionary and sEntiment Reasoner). VADER is a "lexicon and rule-based sentiment analysis tool that is specifically attuned to sentiments expressed in social media". VADER model detects the polarity of sentiment (how positive or negative) of a given body of text. Keep in mind that VADER is a pre-trained model sentiment analysis on social media so take my graph with a grant of salt.
![Sentiment](/images/The_Office_imgs/Avg_Sentiment_Scores.png)

There are two primary couples throughout the series: Jim-Pam and Michael-Holly. They scored the top average positive sentiment. Surprisingly Holly’s average negative sentiment score is pretty high. This can be explained by her jokes, bad impressions of Yoda, Master Thespian. She has striking similarities to Michael and his sense of humor. The other characters that have high average negative scores are Meredith, Angela and Stanley.
```python
from nltk.sentiment.vader import SentimentIntensityAnalyzer
sid = SentimentIntensityAnalyzer()
df_sentiment = office.copy()
df_sentiment['sentiment'] = df_sentiment['line_text'].apply(lambda x: sid.polarity_scores(x))
#extract the postive and negative sentiment score
df_sentiment['vader_positive']  = df_sentiment['sentiment'].apply(lambda score_dict: score_dict['pos'])
df_sentiment['vader_negative']  = df_sentiment['sentiment'].apply(lambda score_dict: score_dict['neg'])
df_sentiment = df_sentiment[['speaker','vader_positive','vader_negative']]
#get the average pos and neg sentiment scores for earch character
df_sentiment.groupby('speaker').mean().reset_index()
def graph_sentiment():
    n = len(df_sentiment.vader_positive)
    X = np.arange(n) 
    fig = plt.figure(figsize=(20, 20))
    ax = plt.barh(X, df_sentiment.vader_positive, facecolor='#74c69d', edgecolor='white')
		#plot the negative sentiment scores
    ax = plt.barh(X, -df_sentiment.vader_negative, facecolor='#ffadad', edgecolor='white')

```
I learned about `lambda` function at my first course Introduction to Data Science in Python. It comes in handy in a lot of situations. I am highly recommend people starting use more `lambda`.

I came across a [notebook](https://www.kaggle.com/ruchi798/sentiment-analysis-the-simpsons) by Ruchi Bhatia on Kaggke using NRClex to measure the emotion from show The Simpson. That gave me an idea to use NRClex on the characters in The Office.

[NRClex](https://pypi.org/project/NRCLex/) measures the emotional affect from a body of text. Affect dictionary contains approximately 27,000 words, and is based on the National Research Council Canada (NRC) affect lexicon and the NLTK library’s WordNet synonym sets.
![Emotionals](/images/The_Office_imgs/Emotion_affect.png)

Ryan scored the highest postive affect 23.7%. On the other hand, Stanley, Dwight, Nellie, Kevin , Angela, Holly once again scored high on the negative affect.

```python
from nrclex import NRCLex
#get emotion score using NRCLex
def get_emotion(data=None):
    text_object = NRCLex(' '.join(data))
    sentiment_scores = pd.DataFrame(list(text_object.raw_emotion_scores.items())) 
    sentiment_scores = sentiment_scores.rename(columns={0: "Sentiment", 1: "Count"})
    #sentiment_scores.sort_values(by=['Count'], ascending=False)
    return sentiment_scores.sort_values(by=['Count'], ascending=False)
#for ploting
def pie_plot(data = None, title=None, location=None):
    lb = data['Sentiment']
    mys = sum(data['Count'])
    labels=[f'{np.round(y/mys*100,1)}% {lb[x]}' for x,y in data['Count'].items()]
    sizes = data['Count']    
    cmap = plt.get_cmap('tab20b')
    colors = [cmap(i) for i in np.linspace(0, 1, 10)]
    # Plot
    plt.figure(figsize=(10,10))
    plt.pie(sizes, labels=labels, startangle=0,frame=False, textprops={'fontsize': 22})

    centre_circle = plt.Circle((0,0),0.8,color='black', fc='white',linewidth=0)
    fig = plt.gcf()
    fig.gca().add_artist(centre_circle)
    plt.title(title, fontsize=19, y=-0.1)
    plt.axis('equal')
    plt.tight_layout()
```
NRClex provided `raw_emotion_scores()` to get a list of emotions. Running that function on each character’s lines returns the emotional effect. All I needed to do is sort them out and send it to the `pie_plot` function. I used Affinity Photo to add them all together.

### "That's what she said!"
In order to see what the characters say the most, n-gram analysis is the best fit. So what is n-gram? n-gram is a sequence of n words with n = 1,2,3 … For example, “Toward Data Science” is a 3-grams (tri-grams), “Dunder Mifflin” is a 2-grams (bi-grams) . N-gram can be used to analyze the the meaning of the text. In this post, I visualized the most common n-grams of some characters.

![Michael_ngrams](/images/The_Office_imgs/Michael_ngrams.png)

Looking at the Uni-gram of Michael, Dwight and Pam got mentioned many times. It’s surprising that Jim did not make it to the top 20. The Bi-grams graph shows “conference room” was mentioned 73 times. Michael loves having people in the conference room, most of the time just for some random reasons unrelated to the day’s business. The analysis on Michael’s spoken words can not be complete without his iconic “That’s what she said!” joke.

[![What_she_said](https://img.youtube.com/vi/dBUGfs9rwms/sddefault.jpg)](https://www.youtube.com/watch?v=dBUGfs9rwms)

```python
#convert the text to all lower case
officep['line_text']=office['line_text'].str.lower()
#create dataFrame to hold all spoken_line by Michael
michael_df_joke = office[office['speaker']=='Michael'].reset_index()
mcount =0
for i in range(0,len(michael_df_joke)):
    x = re.search('what she said',michael_df_joke['line_text_lower'][i])
    if type(x)==re.Match:
        mcount = mcount + 1
```

![Alt text](/images/The_Office_imgs/Dwight_ngrams.png)
![Alt text](/images/The_Office_imgs/Jim_ngrams.png)
![Alt text](/images/The_Office_imgs/Pam_ngrams.png)
![Alt text](/images/The_Office_imgs/Kevin_ngrams.png)

## Conclusion
In this post, we went over some steps to explore the data and sentiment analysis. We have touched up on data wrangling with pandas, characters interaction using `NetworkX`, NLP with `nltk` and `NRClex`, data visualization with `seaborn` and `matplotlib`.

Using a show like ‘The Office’ as my subject felt like the perfect opportunity to not only practice but also improve my skill in the field of Data Science. It was an exciting, although at times, a challenging project. I am very impressed with the NLP tools and their power to present deeper insight view from the data.

I hope you enjoyed reading this post as much as I enjoyed writing it. If you have any questions or comments, please feel free to contact me. I would love to hear from you.

## References
- Daniel Gaerber. Github. [interesting_visuals](https://github.com/Gandagorn/interesting_visuals).
- Adam Reevesman. Toward Data Science. [The Simpsons meets data visualization](https://towardsdatascience.com/the-simpsons-meets-data-visualization-ef8ef0819d13).
- Ruchi Bhatia.Kaggle. [Sentiment Analysis The Simpsons](https://www.kaggle.com/ruchi798/sentiment-analysis-the-simpsons).
-  Hutto, C.J. & Gilbert, E.E. (2014). VADER: A Parsimonious Rule-based Model for Sentiment Analysis of Social Media Text. Eighth International Conference on Weblogs and Social Media (ICWSM-14). Ann Arbor, MI, June 2014.
-  Bird, Steven, Edward Loper and Ewan Klein (2009), Natural Language Processing with Python. O’Reilly Media Inc.