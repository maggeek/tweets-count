"""tweets.py: Connects to the Twitter API and checks how many times each hashtag from the list given
              by the user (either read from file or entered from the console) was tweeted within the 
              last hour. Then the hashtags and corresponding number of tweets are written into a 
              SQLite database, and shown on a pie chart.""" 

import sqlite3
import re
import sys
import matplotlib.pyplot as plt
from datetime import datetime, timedelta
from TwitterAPI import TwitterAPI
from TwitterAPI import TwitterRestPager

def main():
    print('This program allows you to find out how many times particular hashtags have been'\
          'tweeted in the last hour. It creates a database "data.sqlite3" with the results.')
    print('You can search Twitter by typing a list of expressions or reading them from file.')
    hashtags = choose_hashtags()
    counts = count_tweets(hashtags)
    store_in_database(hashtags, counts)
    make_pie_chart(hashtags, counts)

def choose_hashtags():
    """check user's choice and whether it is a valid option"""    
    while True: 
        choice = input('Enter f for file:\nEnter l for list: ').strip()
        if choice == 'f':
            hashtags = file_source()
            break
        elif choice == 'l':
            hashtags = list_source()
            break
        else: 
            print("You didn't choose an available option. Try again.")
    return hashtags
        
def file_source():              
    """read from file"""
    source = input('Make sure the expressions in the file are either comma or line seperated.'\
                   'Enter the name of the text file including ".txt" extension:\n').strip()
    hashtags = []
    if re.match('^.*\.txt$',source):
        with open(source,'r') as f:
            for line in f:                              #go through each line
                for expression in line.split(r','):     #go through each expression
                    hashtags.append(expression.strip())
        while '' in hashtags:                           #remove empty elements from the list
            hashtags.remove('')                       
    else: 
        print('Wrong file type. Try to run the program again.')
        sys.exit()                                      #exit the program if wrong file type    
    return hashtags

def list_source():
    """read expressions entered by user"""
    source = input('Enter the list of expressions, seperated by comma. Do not'\
                   'add a hashtag symbol at the front of the expression.\n').strip()
    hashtags = source.split(",")                        #split the expressions by comma
    return hashtags
 
def count_tweets(hashtags):
    """connect to the Twitter API and get our counts for each expression"""
    #substitute your API and ACCESS credentials
    api = TwitterAPI('API_KEY','API_SECRET','ACCESS_TOKEN','ACCESS_TOKEN_SECRET')
    all_counts = []                                     #a list of counts for all hashtags
    for hashtag in hashtags                             #iterate through hashtags
        hashtag_count = 0                               #count for one hashtag
        an_hour_ago = datetime.now() - timedelta(hours = 1)       #set time for 1 hour ago
        #we use search/tweets, a REST API endpoint that closes after returning a maximum of 100
        #recent tweets and supports paging
        #TwitterRestPager spaces out successive requests to stay under Twitter API the rate limit
        r = TwitterRestPager(api, 'search/tweets', {'q':'#{}'.format(hashtag), 'count': 100})
        for item in r.get_iterator(wait=6):             #increase the wait interval to 6 seconds
            if 'text' in item:            
                hashtag_count += 1
            #in case we exceed the rate limit, error 88 indicates how long to suspend before
            #making a new request
            elif 'message' in item and item['code'] == 88:
                print('SUSPEND, RATE LIMIT EXCEEDED: %s' % item['message'])
                break
            #covert time when each tweet was created to a datatime type
            created = datetime.strptime(item['created_at'], '%a %b %d %H:%M:%S +0000 %Y')
            if created <= an_hour_ago:    #finish counting if created more than an hour ago
                hashtag_count -= 1        #and don't count that tweet 
                break
        all_counts.append(hashtag_count)  #add a count for one hashtag to our list
    return all_counts          
   
def store_in_database(hashtags, counts):
    """connect to a sqlite3 and create a database with our results"""
    db = sqlite3.connect('data.sqlite3')
    cursor = db.cursor()  
    cursor.execute('''DROP TABLE IF EXISTS Data''')
    cursor.execute('''CREATE TABLE Data (hashtag TEXT PRIMARY KEY, count INTEGER)''')  
    for i in range(len(hashtags)):
        cursor.execute('INSERT INTO Data (hashtag, count) VALUES (?,?)', (hashtags[i].strip(),
                       counts[i]))              
    db.commit()
    db.close             

def make_pie_chart(hashtags, counts):
    """make a pie chart with out hashtags and corresponding counts"""
    fig = plt.figure(figsize=[10,10])
    ax = fig.add_subplot(111)
    counts = sorted(counts)                            #sort counts
    cmap = plt.cm.prism                                #choose a colourmap for our pie chart              
    colors = cmap(np.linspace(0., 1., len(counts)))    #attach colours to counts
    pie_chart = ax.pie(counts, labels=hashtags, shadow=True, colors=colors, startangle=160, 
                       labeldistance=1.05, autopct='%1.1f%%')
    for pie_wedge in pie_chart[0]:                     #set the edges of the pie slices to white
        pie_wedge.set_edgecolor('white')
    ax.axis('equal')                                   #view the plot drop above
    plt.show()
        
main()