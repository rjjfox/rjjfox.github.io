---
layout: post
title:  "Jupyter Notebook"
categories: [books, python]
location: London, UK
location-link: london
---

<!--description-->

# Analysing my Goodreads export

I recently got an export from Goodreads of all the reading activity I've logged with Goodreads and will try and create some interesting visualisations with this.


```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import matplotlib.dates as mdates
import datetime as dt

sns.set()

# import matplotlib.style as style
# style.use('seaborn-dark')

pd.plotting.register_matplotlib_converters()
```

I will first drop any of the columns which only contain null values, specifying parameter *how* to equal *'all'*. This includes columns such as 'My Review', 'Private Notes'.


```python
df = pd.read_csv('goodreads_library_export.csv')
df.dropna(axis=1, inplace=True, how='all')
df.drop(labels=['Owned Copies', 'Publisher','Binding', 'Author l-f', 'Additional Authors', 'Year Published', 'ISBN', 'Bookshelves', 'Bookshelves with positions'], axis=1, inplace=True)
print(f'Our Goodreads file has {df.shape[0]} books and {df.shape[1]} columns.')
df['ISBN13'] = df['ISBN13'].str.strip('=').str.strip('"')
df['Exclusive Shelf'] = df['Exclusive Shelf'].astype('category')
df['Date Added'] = pd.to_datetime(df['Date Added'])
df['Date Read'] = pd.to_datetime(df['Date Read'])
df['My Rating'].replace(0, np.nan, inplace=True)
df.head()
```

    Our Goodreads file has 808 books and 12 columns.
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>17874847</td>
      <td>The Signal and the Noise: The Art and Science ...</td>
      <td>Nate Silver</td>
      <td>9780141975658</td>
      <td>4.0</td>
      <td>3.98</td>
      <td>534.0</td>
      <td>2012.0</td>
      <td>2020-06-26</td>
      <td>2018-03-16</td>
      <td>read</td>
    </tr>
    <tr>
      <td>2</td>
      <td>12042357</td>
      <td>Think Stats</td>
      <td>Allen B. Downey</td>
      <td>9781449307110</td>
      <td>NaN</td>
      <td>3.63</td>
      <td>138.0</td>
      <td>2011.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
    </tr>
    <tr>
      <td>3</td>
      <td>14514306</td>
      <td>Think Python</td>
      <td>Allen B. Downey</td>
      <td>9781449330729</td>
      <td>NaN</td>
      <td>4.10</td>
      <td>300.0</td>
      <td>2002.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
    </tr>
    <tr>
      <td>4</td>
      <td>18711042</td>
      <td>Think Bayes</td>
      <td>Allen B. Downey</td>
      <td></td>
      <td>NaN</td>
      <td>3.85</td>
      <td>210.0</td>
      <td>2012.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
    </tr>
  </tbody>
</table>
</div>



## Accessing the Google Books API

My aim in using the Google Books API is to be able to attach a category to the books in the dataset. This will allow me to see whether the types of books I'm reading informs my reading habits.


```python
import requests
import time
import numpy as np
from tqdm import tqdm

payload = {'q': 'The Eye of the World',
           'inauthor': 'Robert Jordan',
           # This hopefully helps to narrow down the search
           'printType': 'books',
           # We specify the country as Google asks that users do this
           'country': 'UK',
           # The fields argument shaves off a small amount of processing time as we do not need to receive
           # every field back from the API
           'fields': 'items(id,volumeInfo(title,authors,categories))',
           # Sometimes disabling the key allows for more requests and avoids HTTP 429 errors
#            'key': 'AIzaSyC3137OKW0Y9d27xa6xTPgdNNT6bN9aPDc'
          }
url = 'https://www.googleapis.com/books/v1/volumes'

r = requests.get(url, params=payload)

print('Status_code:', r.status_code)
print('Status_code ok:', r.status_code == requests.codes.ok)
print('Raise_for_status == None:', r.raise_for_status() == None)
```

    Status_code: 200
    Status_code ok: True
    Raise_for_status == None: True
    

### Defining functions


```python
def get_metadata_from_df(row):
    """Extracts the title, author and ISBN from a row in a dataframe
    
    Arguments
    ---------
    row: pd.Series
        Row from a DataFrame.
        
    Returns
    -------
    metadata: dict
        Dictionary formatted to be parsed as the payload for a get request to the Google Books API.
    
    """
    metadata = {
        'q': row.Title,
        'inauthor': row.Author,
        'isbn': row.ISBN13,
        'printType': 'books',
        'country': 'UK',
        'fields': 'items(id,volumeInfo(title,authors,categories))',
#         'key': 'AIzaSyC3137OKW0Y9d27xa6xTPgdNNT6bN9aPDc'
    }
    return metadata
```


```python
def search_books_api(metadata):
    """Returns Books API response parsed into a dictionary
    
    Arguments
    ---------
    metadata: dict
        Parameters for the Books API request. Dictionary should contain 'q' key at minimum to run.
        
    Returns
    -------
    book_dict: dict
        'status' key in dictionary indicates whether the book was 'found', 'not_found' or error
        was returned, e.g. 'HTTP429'.
        
    Notes
    -----
    HTTP errors will not stop the function, but no books will be returned. Another call will need to be
    made to fill these in. Index is attached as key in dict for easy replacement.
    """
    try:
        s = requests.Session()
        r = s.get('https://www.googleapis.com/books/v1/volumes', params=metadata)
        
        if r.status_code == requests.codes.ok:
            # Good response from HTTP
            
            if r.json().get('items'):
                # Book found in API database
                api_result = r.json().get('items')[0]
                book_dict = {
                    'goodreads_title': metadata['q'],
                    'google_id': api_result['id'],
                    'title': api_result['volumeInfo']['title'],
                    # 'author' and 'categories' values come in list form. Using the join function converts them
                    # into a string which is easier to use later
                    'author': ', '.join(api_result['volumeInfo']['authors']) if 'authors' in \
                        api_result['volumeInfo'] else np.nan,
                    'categories': ', '.join(api_result['volumeInfo']['categories']) if 'categories' in \
                        api_result['volumeInfo'] else np.nan,
                    'status': 'found'
                 }
                
            else:
                # Book not found in API database
                book_dict = {
                    'goodreads_title': metadata['q'],
                    'google_id': np.nan,
                    'title': np.nan,
                    'author': np.nan,
                    'categories': np.nan,
                    'status': 'not_found'
                }
#                 print(metadata['q'], 'appended to not_found')
                not_found.append(metadata['q'])
                
        else:
            # HTTP error returned
            book_dict = {
                'goodreads_title': metadata['q'],
                'google_id': np.nan,
                'title': np.nan,
                'author': np.nan,
                'categories': np.nan,
                'status': 'HTTP'+str(r.status_code),
            }
#             print('HTTP error:', str(r.status_code), metadata['q'], 'appended to error_list')
            error_list.append(metadata['q'])

        return book_dict
    
    except Exception as e:
        print(e)
```

Here's a simple random function to pull out a random book and run it through the functions.


```python
books_list = []
not_found = []
error_list = []

result = search_books_api(get_metadata_from_df(df.loc[np.random.choice(range(len(df)))]))
result
```




    {'goodreads_title': 'The Big Sleep (Philip Marlowe, #1)',
     'google_id': 'ZgfbDwAAQBAJ',
     'title': 'The Big Sleep',
     'author': 'Raymond Chandler',
     'categories': 'Fiction',
     'language': 'en',
     'status': 'found'}



### Checking cache in case this is not our first time

Sending requests to the API can take a long time, if we have already sent requests to the Google Books API, let's reuse the results from this to save time. We will check for a cache csv and then send requests only for the new books.

Requesting data for 800+ books can take over 20 minutes, we don't want to have to do this everytime we do an export from Goodreads as the number of new books should not be too high.


```python
try:
    cache = pd.read_csv('google_books_cache.csv')
    new_books = df[~df['Title'].isin(cache['goodreads_title'])]
    print(f'Cache found - {len(cache)} books in cache.')
    print(f'{len(new_books)} new books to send requests for. \'new_books\' variable made with new additions.')
except FileNotFoundError:
    print('File not found - no cache available.\nNext code cell will search for all books.')
    new_books = df
```

    Cache found - 816 books in cache.
    0 new books to send requests for. 'new_books' variable made with new additions.
    

### Sending requests to the Books API

We are finally ready to create a dataframe with data pulled from the Google Books API.


```python
books_list = []
not_found = []
error_list = []

if len(new_books)>0:
    for index, rows in new_books.iterrows():
        # First we extract relevant column data for the Books API request. We take
        # Title, Author, ISBN13
        metadata = get_metadata_from_df(rows)

        # We use this metadata to search the API and it returns the top result
        book_dict = search_books_api(metadata)
        book_dict['index'] = index

        books_list.append(book_dict)

        # Sleep function to reduce HTTP 429 errors (Too many requests)
        time.sleep(np.random.random())

    google_books = pd.DataFrame(books_list)

    display(google_books.head())

    display(google_books['status'].value_counts())
    print(f'{len(error_list)} errors returned')

    filename = 'google_books_' + dt.datetime.today().strftime('%Y%m%d') + '.csv'
    google_books.to_csv(filename)
```

### Filling missing values

We can use the 'status' column to iterrate over only those rows that we have an error. We will replace the values using the `loc` selector and running the `search_books_api` function. The returned dictionary can be used to fill the values for the missing rows.


```python
if len(error_list)>0:
    not_found = []
    error_list = []

    for index, rows in new_books.reset_index()[google_books['status']!='found'].iterrows():
        # First we extract relevant column data for the Books API request. We take
        # Title, Author, ISBN13
        metadata = get_metadata_from_df(rows)

        # We use this metadata to search the API and it returns the top result
        book_dict = search_books_api(metadata)
        google_books.loc[google_books['index']==index, :'status'] = book_dict.values()

        # Sleep function to not get blocked by the API
        time.sleep(np.random.random())

    print(f'{len(error_list)} errors returned')
    google_books.to_csv(filename)
    google_books.head()
```

### Creating a cache

Here we check if there's a cache. If there is, we add the new rows to it, otherwise, we create a cache for future use.

We'll then do a left-join between the latest Goodreads file and the full cache which we pull from the filepath.


```python
try:
    import os

    if not os.path.isfile('google_books_cache.csv'):
        # if file does not exist write header 
        google_books.to_csv('google_books_cache.csv')
    else: 
        # else it exists so append without writing the header
        google_books.to_csv('google_books_cache.csv', mode='a', header=False)
except NameError:
    pass

full_cache = pd.read_csv('google_books_cache.csv')
# In case we have accidentally added duplicates to our dataset
full_cache.drop_duplicates(subset=['goodreads_title'], inplace=True)
```

### Merging datasets

We now have a full dataset corresponding the original Goodreads export. We will now merge the datasets, adding the `categories` column.

The two datasets share the same index and also `df['title']` is the same as `google_books['goodreads_title']`.


```python
data = pd.merge(df, full_cache, left_on='Title', right_on='goodreads_title', how='left')
data.drop(labels=['goodreads_title', 'title', 'author', 'google_id', 'status', 'index', 'Unnamed: 0.1', 'Unnamed: 0'], axis=1, inplace=True)
display(data.head(3))
print(f'Our full dataset now has {data.shape[0]} books and {data.shape[1]} columns. This matches the original Goodreads dataset of {df.shape[0]} books.')
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>2049823</td>
      <td>Superior Saturday (The Keys to the Kingdom, #6)</td>
      <td>Garth Nix</td>
      <td>9780439700894</td>
      <td>4.0</td>
      <td>3.92</td>
      <td>336.0</td>
      <td>2008.0</td>
      <td>2020-03-30</td>
      <td>2020-07-28</td>
      <td>read</td>
      <td>1</td>
      <td>Juvenile Fiction</td>
      <td>en</td>
    </tr>
    <tr>
      <td>1</td>
      <td>17874847</td>
      <td>The Signal and the Noise: The Art and Science ...</td>
      <td>Nate Silver</td>
      <td>9780141975658</td>
      <td>4.0</td>
      <td>3.98</td>
      <td>534.0</td>
      <td>2012.0</td>
      <td>2020-06-26</td>
      <td>2018-03-16</td>
      <td>read</td>
      <td>1</td>
      <td>Mathematics</td>
      <td>en</td>
    </tr>
    <tr>
      <td>2</td>
      <td>12042357</td>
      <td>Think Stats</td>
      <td>Allen B. Downey</td>
      <td>9781449307110</td>
      <td>NaN</td>
      <td>3.63</td>
      <td>138.0</td>
      <td>2011.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
    </tr>
  </tbody>
</table>
</div>


    Our full dataset now has 808 books and 14 columns. This matches the original Goodreads dataset of 808 books.
    

### Fixing genres/categories

I'm very interested to see the split between the different genres and how this impacts my reading behaviour. First however, we need to clean up and group the data a little.

Ideally, we would have a column dictating either fiction or non-fiction. Let's see if we can generalise intelligently.

Unfortunately, it is not completely clear-cut which categories are fiction or non-fiction


```python
data.categories.value_counts()
```




    Fiction                                     353
    Juvenile Fiction                             59
    History                                      28
    Biography & Autobiography                    25
    Comics & Graphic Novels                      18
                                               ... 
    Miniature books                               1
    Caulfield, Holden (Fictitious character)      1
    Humorous stories                              1
    Alchemists                                    1
    Paris (France)                                1
    Name: categories, Length: 125, dtype: int64




```python
import re

fiction_genres = [
    'Comics & Graphic Novels', 'Great Britain', 'Juvenile Fiction',
    'Young Adult Fiction', 'Drama', 'Adventure stories',
    'Performing Arts', 'Historical fiction', 'Aubrey, Jack (Fictitious character)',
    'Napoleonic Wars, 1800-1815', 'Discworld (Imaginary place)',
    'Badajoz (Spain)', 'Children\'s stories', 'Greece', 'Poetry',
    'Literary Collections', 'Greeks', 'Gangs', 'Arctic regions',
    'Gangs', 'African Americans', 'Alchemists', 'Accident victims',
    'Animals', 'Anglo-Saxons', 'Adultery', 'Audiobooks',
    'Bangkok (Thailand)', 'Art museum curators', 'Benedictine monasteries',
    'Book burning', 'Boys', 'Blessing and cursing', 'Brothers',
    'Brothers and sisters', 'British', 'Byzantine Empire',
    'Cathedrals', 'Cancer', 'Courtship', 'Executions and executioners',
    'Future life', 'Fourth dimension', 'Galvanic skin response',
    'Gladiators', 'Guardian and ward', 'India', 'Japan',
    'Judgment Day', 'Kidnapping victims', 'Learning, Psychology of',
    'Literary Criticism', 'Man-woman relationships',
    'Murder', 'Miniature books', 'Single women', 'Study Aids'
]

nonfiction_genres = [
    'Biography & Autobiography', 'History', 'Business & Economics',
    'Self-Help', 'Religion', 'Psychology', 'Philosophy',
    'Mathematics', 'Science', 'Social Science',
    'Computers','Body, Mind & Spirit', 'Art', 'Family & Relationships',
    'Travel', 'Sports & Recreation', 'Foreign Language Study',
    'Technology & Engineering', 'Health & Fitness', 'True Crime',
    'Forecasting', 'Factory farms', 'Design', 'Activities',
    'Art and music', 'Belarus', 'Authors, Iranian',
    'Children of Holocaust survivors', 'Cerebrovascular disease',
    'Cooking', 'Cooking, Mexican', 'Homelessness', 'Humor', 'Latin America',
    'Mathematicians', 'Nature', 'Naval battles', 'Netherlands',
    'Political Science', 'Photography'
    
]

data['genre'] = data['categories']
data.loc[data.categories.isin(fiction_genres), 'genre'] = 'Fiction'
data.loc[data.Author == 'Bernard Cornwell', 'genre'] = 'Fiction'
data.loc[data.categories.str.contains(pat='Ficti|stories|Comic|english|england|france',\
                                      regex=True, flags=re.IGNORECASE, na=False), 'genre'] = 'Fiction'
data.loc[data.categories.isin(nonfiction_genres), 'genre'] = 'Non-fiction'
data.loc[data['Author']=='Robert M. Pirsig', 'genre'] = 'Fiction'

data['genre'].value_counts()

```




    Fiction        570
    Non-fiction    187
    Name: genre, dtype: int64



That was painful. I'm going the whole way and I'm going to manually categorise the uncategorised books. I can count roughly 8 or so non-fiction books so I'll label those and the rest are be fiction.


```python
nonfiction_ISBN13 = ['9781408870556', '9780618680009', '9780241380376', '9781973181460', '9781783963461', '9781841157917']
nonfiction_title = ['Why I\'m No Longer Talking to White People About Race', 'The God Delusion', 'Spanish Short Stories For Beginners: 8 Unconventional Short Stories to Grow Your Vocabulary and Learn Spanish the Fun Way!']
data.loc[data['ISBN13'].isin(nonfiction_ISBN13), 'genre'] = 'Non-fiction'
data.loc[data['Title'].isin(nonfiction_title), 'genre'] = 'Non-fiction'
data.loc[~data['genre'].isin(['Fiction', 'Non-fiction']), 'genre'] = 'Fiction'
data.to_csv('full_books_df.csv')
```

It's done. All books have been labelled either `Fiction` or `Non-fiction`


```python
data['genre'].value_counts()
```




    Fiction        614
    Non-fiction    194
    Name: genre, dtype: int64



# Exploratory Data Analysis

## Finished books

Now taking a look at only those books that have been read... I've dropped books read before 2013 as I joined Goodreads in February 2012 but only started updating my Goodreads in 2013.


```python
read = data[data['Date Read'].notnull()][['Title', 'Author', 'Number of Pages', 'Date Read', 'genre']].sort_values('Date Read', ascending=False)
read['Date Read'] = pd.to_datetime(read['Date Read'])
read.set_index('Date Read', inplace=True)
read.rename(columns={'Title': 'Books'}, inplace=True)
read = read[:'2014']
read.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Books</th>
      <th>Author</th>
      <th>Number of Pages</th>
      <th>genre</th>
    </tr>
    <tr>
      <th>Date Read</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2020-07-18</td>
      <td>Viking Britain</td>
      <td>Thomas J.T. Williams</td>
      <td>416.0</td>
      <td>Non-fiction</td>
    </tr>
    <tr>
      <td>2020-07-18</td>
      <td>Girl, Woman, Other</td>
      <td>Bernardine Evaristo</td>
      <td>453.0</td>
      <td>Fiction</td>
    </tr>
    <tr>
      <td>2020-07-12</td>
      <td>Héraclès La jeunesse du héros</td>
      <td>Clotilde Bruneau</td>
      <td>56.0</td>
      <td>Fiction</td>
    </tr>
    <tr>
      <td>2020-07-12</td>
      <td>Shenzhen</td>
      <td>Guy Delisle</td>
      <td>144.0</td>
      <td>Non-fiction</td>
    </tr>
    <tr>
      <td>2020-07-08</td>
      <td>The Hanging Tree (Peter Grant, #6)</td>
      <td>Ben Aaronovitch</td>
      <td>387.0</td>
      <td>Fiction</td>
    </tr>
  </tbody>
</table>
</div>



To be able to use the `genre` column in a stacked bar chart, I believe I need to make it into two columns.


```python
def fiction_col(row):
    if row['genre'] == 'Fiction':
        return 1
    else:
        return 0
    
def nonfiction_col(row):
    if row['genre'] == 'Non-fiction':
        return 1
    else:
        return 0

read['Fiction'] = read.apply(fiction_col, axis=1)
read['Nonfiction'] = read.apply(nonfiction_col, axis=1)
read.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Books</th>
      <th>Author</th>
      <th>Number of Pages</th>
      <th>genre</th>
      <th>Fiction</th>
      <th>Nonfiction</th>
    </tr>
    <tr>
      <th>Date Read</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2020-07-18</td>
      <td>Viking Britain</td>
      <td>Thomas J.T. Williams</td>
      <td>416.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2020-07-18</td>
      <td>Girl, Woman, Other</td>
      <td>Bernardine Evaristo</td>
      <td>453.0</td>
      <td>Fiction</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2020-07-12</td>
      <td>Héraclès La jeunesse du héros</td>
      <td>Clotilde Bruneau</td>
      <td>56.0</td>
      <td>Fiction</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <td>2020-07-12</td>
      <td>Shenzhen</td>
      <td>Guy Delisle</td>
      <td>144.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2020-07-08</td>
      <td>The Hanging Tree (Peter Grant, #6)</td>
      <td>Ben Aaronovitch</td>
      <td>387.0</td>
      <td>Fiction</td>
      <td>1</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>



Generally, I read a lot more fiction than non-fiction. Here we see 73% of the books read are fiction and 27% non-fiction.


```python
read['genre'].value_counts(normalize=True)
```




    Fiction        0.737752
    Non-fiction    0.262248
    Name: genre, dtype: float64



### Time Series 

#### Monthly data

Here I'm resampling the dataset to to give me a monthly count of books and pages read based on date.


```python
read_monthly = read.resample('M').agg({'Books': 'count', 'Number of Pages': 'sum', 'Fiction': 'sum', 'Nonfiction': 'sum'})
read_monthly['Pages per Book'] = read_monthly['Number of Pages'] / read_monthly['Books']
read_monthly.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Books</th>
      <th>Number of Pages</th>
      <th>Fiction</th>
      <th>Nonfiction</th>
      <th>Pages per Book</th>
    </tr>
    <tr>
      <th>Date Read</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2015-01-31</td>
      <td>7</td>
      <td>2254.0</td>
      <td>7</td>
      <td>0</td>
      <td>322.00</td>
    </tr>
    <tr>
      <td>2015-02-28</td>
      <td>4</td>
      <td>1325.0</td>
      <td>4</td>
      <td>0</td>
      <td>331.25</td>
    </tr>
    <tr>
      <td>2015-03-31</td>
      <td>2</td>
      <td>726.0</td>
      <td>2</td>
      <td>0</td>
      <td>363.00</td>
    </tr>
    <tr>
      <td>2015-04-30</td>
      <td>2</td>
      <td>1011.0</td>
      <td>2</td>
      <td>0</td>
      <td>505.50</td>
    </tr>
    <tr>
      <td>2015-05-31</td>
      <td>8</td>
      <td>1492.0</td>
      <td>7</td>
      <td>1</td>
      <td>186.50</td>
    </tr>
  </tbody>
</table>
</div>




```python

fig, ax = plt.subplots(figsize=(10, 6))
ax.bar(read_monthly.index, read_monthly.Fiction, label='Fiction', width=1)
ax.bar(read_monthly.index, read_monthly.Nonfiction, label='Nonfiction', bottom=read_monthly.Fiction,\
       width=1)
# ax.set_ylabel('Books read')
# ax.set_title('Number of books read per month')
ax.legend()
plt.show()
```

    C:\Users\ryanf\AppData\Roaming\Python\Python37\site-packages\matplotlib\axes\_axes.py:2180: FutureWarning: Addition/subtraction of integers and integer-arrays to Timestamp is deprecated, will be removed in a future version.  Instead of adding/subtracting `n`, use `n * self.freq`
      dx = [convert(x0 + ddx) - x for ddx in dx]
    


![png](output_40_1.png)


It looks like I've started to read more non-fiction. One month, I read 7 non-fiction books! This was during the height of my self-help phase while in Bali.


```python
non_fiction = read[read['genre']=='Non-fiction'].sort_values('Date Read', ascending=False)
non_fiction.loc['2019-12']
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Books</th>
      <th>Author</th>
      <th>Number of Pages</th>
      <th>genre</th>
      <th>Fiction</th>
      <th>Nonfiction</th>
    </tr>
    <tr>
      <th>Date Read</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2019-12-29</td>
      <td>The Courage to Be Disliked: How to Free Yourse...</td>
      <td>Ichiro Kishimi</td>
      <td>288.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2019-12-28</td>
      <td>Mindfulness in Plain English</td>
      <td>Henepola Gunaratana</td>
      <td>208.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2019-12-21</td>
      <td>True Love: A Practice for Awakening the Heart</td>
      <td>Thich Nhat Hanh</td>
      <td>120.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2019-12-21</td>
      <td>Thinking, Fast and Slow</td>
      <td>Daniel Kahneman</td>
      <td>512.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2019-12-17</td>
      <td>The Subtle Art of Not Giving a F*ck: A Counter...</td>
      <td>Mark Manson</td>
      <td>224.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2019-12-10</td>
      <td>The Gifts of Imperfection</td>
      <td>Brené Brown</td>
      <td>138.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2019-12-06</td>
      <td>Zen Mind, Beginner's Mind: Informal Talks on Z...</td>
      <td>Shunryu Suzuki</td>
      <td>138.0</td>
      <td>Non-fiction</td>
      <td>0</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>



#### Quarterly data


```python
read_quarterly = read.resample('QS').agg({'Books': 'count', 'Number of Pages': 'sum', 'Fiction': 'sum', 'Nonfiction': 'sum'})

fig, ax = plt.subplots(figsize=(10, 6))
ax.bar(read_quarterly.index, read_quarterly.Fiction, width=1)
ax.bar(read_quarterly.index, read_quarterly.Nonfiction, bottom=read_quarterly.Fiction, width=1)

ax.set_ylabel('Books read')
ax.set_title('Number of books read per quarter')
plt.show()
```


![png](output_44_0.png)


#### Yearly data

A tip with yearly and quarterly data. If you use resample alias `Y`, this leaves the year label as the end of the year. When plotting this, often Pandas will place this under the following year, i.e. 2021 instead of 2020. Using alias `YS` uses the Year Start as the label, making for more sensible plotting labels.


```python
read_yearly = read.resample('YS').agg({'Books': 'count', 'Number of Pages': 'sum', 'Fiction': 'sum', 'Nonfiction': 'sum'})
read_yearly['Average Book Length'] = read_yearly['Number of Pages']/read_yearly['Books']
fig, ax = plt.subplots(figsize=(10, 6))
ax.bar(read_yearly.index, read_yearly.Fiction, width=1, label='Fiction')
ax.bar(read_yearly.index, read_yearly.Nonfiction, bottom=read_yearly.Fiction, width=1, label='Non-fiction')

ax.set_ylabel('Books read')
ax.set_title('Number of books read per year')
ax.legend(loc='upper left')
plt.show()
```


![png](output_46_0.png)


I am reading more overall, there is definitely an upward trend. Remember that we are only halfway through 2020 so this value will certainly increase.

However, the number of fiction books has stayed fairly steady, it's the non-fiction books which are driving the increase, based on this chart.

## Categories

Let's look at categories briefly. 


```python
data['categories'].value_counts().head(15).index
```




    Index(['Fiction', 'Juvenile Fiction', 'History', 'Biography & Autobiography',
           'Comics & Graphic Novels', 'Business & Economics', 'Great Britain',
           'Self-Help', 'Mathematics', 'Computers', 'Religion', 'Philosophy',
           'Psychology', 'Young Adult Fiction', 'Science'],
          dtype='object')




```python
fig, ax = plt.subplots(figsize=(10, 6))
data['categories'].value_counts().head(15).plot(kind='barh')
ax.invert_yaxis()
```


![png](output_50_0.png)



```python
for category in data['categories'].value_counts().head(15).index:
    print(category)
    display(data[(data['categories']==category)&(data['Exclusive Shelf']=='to-read')].sort_values('Average Rating', ascending=False).head())
```

    Fiction
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>304</td>
      <td>9188338</td>
      <td>The Way of Kings (The Stormlight Archive, #1)</td>
      <td>Brandon Sanderson</td>
      <td></td>
      <td>NaN</td>
      <td>4.65</td>
      <td>1137.0</td>
      <td>2010.0</td>
      <td>NaT</td>
      <td>2016-03-17</td>
      <td>to-read</td>
      <td>0</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>4.372603</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>10</td>
      <td>43450333</td>
      <td>Traitors of Rome (Eagle #18)</td>
      <td>Simon Scarrow</td>
      <td></td>
      <td>NaN</td>
      <td>4.47</td>
      <td>464.0</td>
      <td>2019.0</td>
      <td>NaT</td>
      <td>2020-07-23</td>
      <td>to-read</td>
      <td>0</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.019178</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>286</td>
      <td>102030</td>
      <td>Musashi</td>
      <td>Eiji Yoshikawa</td>
      <td>9784770019578</td>
      <td>NaN</td>
      <td>4.46</td>
      <td>970.0</td>
      <td>1935.0</td>
      <td>NaT</td>
      <td>2019-06-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.095890</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>303</td>
      <td>6547258</td>
      <td>The Final Empire (Mistborn, #1)</td>
      <td>Brandon Sanderson</td>
      <td>9780575089914</td>
      <td>NaN</td>
      <td>4.45</td>
      <td>647.0</td>
      <td>2006.0</td>
      <td>NaT</td>
      <td>2016-03-17</td>
      <td>to-read</td>
      <td>0</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>4.372603</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>315</td>
      <td>426504</td>
      <td>Ficciones</td>
      <td>Jorge Luis Borges</td>
      <td>9780802130303</td>
      <td>NaN</td>
      <td>4.45</td>
      <td>174.0</td>
      <td>1944.0</td>
      <td>NaT</td>
      <td>2019-05-03</td>
      <td>to-read</td>
      <td>0</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.243836</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Juvenile Fiction
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>435</td>
      <td>370493</td>
      <td>The Giving Tree</td>
      <td>Shel Silverstein</td>
      <td>9780060256654</td>
      <td>NaN</td>
      <td>4.38</td>
      <td>64.0</td>
      <td>1964.0</td>
      <td>NaT</td>
      <td>2017-12-10</td>
      <td>to-read</td>
      <td>0</td>
      <td>Juvenile Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>2.638356</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>181</td>
      <td>25614492</td>
      <td>Salt to the Sea</td>
      <td>Ruta Sepetys</td>
      <td>9780399160301</td>
      <td>NaN</td>
      <td>4.36</td>
      <td>391.0</td>
      <td>2016.0</td>
      <td>NaT</td>
      <td>2019-10-12</td>
      <td>to-read</td>
      <td>0</td>
      <td>Juvenile Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.800000</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>256</td>
      <td>10138607</td>
      <td>Habibi</td>
      <td>Craig Thompson</td>
      <td>9780375424144</td>
      <td>NaN</td>
      <td>4.03</td>
      <td>672.0</td>
      <td>2011.0</td>
      <td>NaT</td>
      <td>2019-08-24</td>
      <td>to-read</td>
      <td>0</td>
      <td>Juvenile Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.934247</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>283</td>
      <td>334123</td>
      <td>The Amulet of Samarkand (Bartimaeus, #1)</td>
      <td>Jonathan Stroud</td>
      <td>9780786818594</td>
      <td>NaN</td>
      <td>4.01</td>
      <td>462.0</td>
      <td>2003.0</td>
      <td>NaT</td>
      <td>2019-07-07</td>
      <td>to-read</td>
      <td>0</td>
      <td>Juvenile Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.065753</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>585</td>
      <td>47520</td>
      <td>Castle in the Air (Howl's Moving Castle, #2)</td>
      <td>Diana Wynne Jones</td>
      <td>9780064473453</td>
      <td>NaN</td>
      <td>3.91</td>
      <td>298.0</td>
      <td>1990.0</td>
      <td>NaT</td>
      <td>2015-10-25</td>
      <td>to-read</td>
      <td>0</td>
      <td>Juvenile Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>4.767123</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    History
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>362</td>
      <td>36992483</td>
      <td>Erebus: The Story of a Ship</td>
      <td>Michael Palin</td>
      <td>9781847948120</td>
      <td>NaN</td>
      <td>4.32</td>
      <td>334.0</td>
      <td>2018.0</td>
      <td>NaT</td>
      <td>2018-12-09</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.641096</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>119</td>
      <td>40664392</td>
      <td>Chernobyl: History of a Tragedy</td>
      <td>Serhii Plokhy</td>
      <td>9780141988351</td>
      <td>NaN</td>
      <td>4.23</td>
      <td>404.0</td>
      <td>2018.0</td>
      <td>NaT</td>
      <td>2019-03-15</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.378082</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>579</td>
      <td>56781</td>
      <td>The Civil War</td>
      <td>Gaius Julius Caesar</td>
      <td>9780140441871</td>
      <td>NaN</td>
      <td>4.06</td>
      <td>368.0</td>
      <td>-47.0</td>
      <td>NaT</td>
      <td>2015-12-06</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>4.652055</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>146</td>
      <td>23522</td>
      <td>Mythology</td>
      <td>Edith Hamilton</td>
      <td>9780316341516</td>
      <td>NaN</td>
      <td>4.00</td>
      <td>497.0</td>
      <td>1942.0</td>
      <td>NaT</td>
      <td>2020-01-19</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.528767</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>580</td>
      <td>592167</td>
      <td>The Conquest of Gaul</td>
      <td>Gaius Julius Caesar</td>
      <td>9780140444339</td>
      <td>NaN</td>
      <td>4.00</td>
      <td>269.0</td>
      <td>-50.0</td>
      <td>NaT</td>
      <td>2015-12-06</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>4.652055</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Biography & Autobiography
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>149</td>
      <td>25899336</td>
      <td>When Breath Becomes Air</td>
      <td>Paul Kalanithi</td>
      <td></td>
      <td>NaN</td>
      <td>4.36</td>
      <td>208.0</td>
      <td>2016.0</td>
      <td>NaT</td>
      <td>2020-01-15</td>
      <td>to-read</td>
      <td>0</td>
      <td>Biography &amp; Autobiography</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.539726</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>434</td>
      <td>1617</td>
      <td>Night  (The Night Trilogy, #1)</td>
      <td>Elie Wiesel</td>
      <td></td>
      <td>NaN</td>
      <td>4.33</td>
      <td>115.0</td>
      <td>1956.0</td>
      <td>NaT</td>
      <td>2017-12-10</td>
      <td>to-read</td>
      <td>0</td>
      <td>Biography &amp; Autobiography</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>2.638356</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>44</td>
      <td>17851885</td>
      <td>I Am Malala: The Story of the Girl Who Stood U...</td>
      <td>Malala Yousafzai</td>
      <td>9780316322409</td>
      <td>NaN</td>
      <td>4.12</td>
      <td>327.0</td>
      <td>2012.0</td>
      <td>NaT</td>
      <td>2020-07-09</td>
      <td>to-read</td>
      <td>0</td>
      <td>Biography &amp; Autobiography</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.057534</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>468</td>
      <td>522534</td>
      <td>Geisha, a Life</td>
      <td>Mineko Iwasaki</td>
      <td>9780743444293</td>
      <td>NaN</td>
      <td>3.93</td>
      <td>297.0</td>
      <td>2002.0</td>
      <td>NaT</td>
      <td>2017-03-01</td>
      <td>to-read</td>
      <td>0</td>
      <td>Biography &amp; Autobiography</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>3.416438</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>323</td>
      <td>38897135</td>
      <td>The Cow Book: A Story of Life on a Family Farm</td>
      <td>John Connell</td>
      <td></td>
      <td>NaN</td>
      <td>3.86</td>
      <td>205.0</td>
      <td>2018.0</td>
      <td>NaT</td>
      <td>2019-03-15</td>
      <td>to-read</td>
      <td>0</td>
      <td>Biography &amp; Autobiography</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.378082</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Comics & Graphic Novels
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>124</td>
      <td>17287068</td>
      <td>Showa, 1926-1939: A History of Japan</td>
      <td>Shigeru Mizuki</td>
      <td>9781770461352</td>
      <td>NaN</td>
      <td>4.26</td>
      <td>533.0</td>
      <td>2013.0</td>
      <td>NaT</td>
      <td>2018-11-12</td>
      <td>to-read</td>
      <td>0</td>
      <td>Comics &amp; Graphic Novels</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.715068</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>253</td>
      <td>5805</td>
      <td>V for Vendetta</td>
      <td>Alan Moore</td>
      <td>9781401207922</td>
      <td>NaN</td>
      <td>4.25</td>
      <td>296.0</td>
      <td>1990.0</td>
      <td>NaT</td>
      <td>2019-08-24</td>
      <td>to-read</td>
      <td>0</td>
      <td>Comics &amp; Graphic Novels</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.934247</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>340</td>
      <td>21611</td>
      <td>The Forever War (The Forever War, #1)</td>
      <td>Joe Haldeman</td>
      <td>9780060510862</td>
      <td>NaN</td>
      <td>4.15</td>
      <td>278.0</td>
      <td>1974.0</td>
      <td>NaT</td>
      <td>2019-02-25</td>
      <td>to-read</td>
      <td>0</td>
      <td>Comics &amp; Graphic Novels</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.427397</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>255</td>
      <td>34072</td>
      <td>Jimmy Corrigan: The Smartest Kid on Earth</td>
      <td>Chris Ware</td>
      <td>9780224063975</td>
      <td>NaN</td>
      <td>4.09</td>
      <td>380.0</td>
      <td>2000.0</td>
      <td>NaT</td>
      <td>2019-08-24</td>
      <td>to-read</td>
      <td>0</td>
      <td>Comics &amp; Graphic Novels</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.934247</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Business & Economics
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>168</td>
      <td>13588356</td>
      <td>Daring Greatly: How the Courage to Be Vulnerab...</td>
      <td>Brené Brown</td>
      <td>9781592407330</td>
      <td>NaN</td>
      <td>4.28</td>
      <td>287.0</td>
      <td>2012.0</td>
      <td>NaT</td>
      <td>2019-12-10</td>
      <td>to-read</td>
      <td>0</td>
      <td>Business &amp; Economics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.638356</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>290</td>
      <td>12658</td>
      <td>Hawaii</td>
      <td>James A. Michener</td>
      <td>9780375760372</td>
      <td>NaN</td>
      <td>4.20</td>
      <td>1136.0</td>
      <td>1959.0</td>
      <td>NaT</td>
      <td>2019-06-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Business &amp; Economics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.095890</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>129</td>
      <td>6061484</td>
      <td>Outliers: The Story of Success</td>
      <td>Malcolm Gladwell</td>
      <td>9780141036243</td>
      <td>NaN</td>
      <td>4.16</td>
      <td>365.0</td>
      <td>2008.0</td>
      <td>NaT</td>
      <td>2016-01-01</td>
      <td>to-read</td>
      <td>0</td>
      <td>Business &amp; Economics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>4.580822</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>52</td>
      <td>38315</td>
      <td>Fooled by Randomness: The Hidden Role of Chanc...</td>
      <td>Nassim Nicholas Taleb</td>
      <td>9780812975215</td>
      <td>NaN</td>
      <td>4.07</td>
      <td>368.0</td>
      <td>2001.0</td>
      <td>NaT</td>
      <td>2020-06-20</td>
      <td>to-read</td>
      <td>0</td>
      <td>Business &amp; Economics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.109589</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>130</td>
      <td>873537</td>
      <td>Blink: The Power of Thinking Without Thinking</td>
      <td>Malcolm Gladwell</td>
      <td>9780141014593</td>
      <td>NaN</td>
      <td>3.93</td>
      <td>277.0</td>
      <td>2005.0</td>
      <td>NaT</td>
      <td>2016-01-01</td>
      <td>to-read</td>
      <td>0</td>
      <td>Business &amp; Economics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>4.580822</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Great Britain
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>85</td>
      <td>7826803</td>
      <td>Wolf Hall (Thomas Cromwell, #1)</td>
      <td>Hilary Mantel</td>
      <td>9780312429980</td>
      <td>NaN</td>
      <td>3.88</td>
      <td>604.0</td>
      <td>2009.0</td>
      <td>NaT</td>
      <td>2020-03-25</td>
      <td>to-read</td>
      <td>0</td>
      <td>Great Britain</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.347945</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Self-Help
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
</div>


    Mathematics
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>7</td>
      <td>17397466</td>
      <td>An Introduction to Statistical Learning: With ...</td>
      <td>Gareth James</td>
      <td>9781461471370</td>
      <td>NaN</td>
      <td>4.61</td>
      <td>426.0</td>
      <td>2013.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Mathematics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>8</td>
      <td>148009</td>
      <td>The Elements of Statistical Learning: Data Min...</td>
      <td>Trevor Hastie</td>
      <td>9780387952840</td>
      <td>NaN</td>
      <td>4.38</td>
      <td>552.0</td>
      <td>2001.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Mathematics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>4</td>
      <td>18711042</td>
      <td>Think Bayes</td>
      <td>Allen B. Downey</td>
      <td></td>
      <td>NaN</td>
      <td>3.85</td>
      <td>210.0</td>
      <td>2012.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Mathematics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Computers
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>6</td>
      <td>24072897</td>
      <td>Deep Learning</td>
      <td>Ian Goodfellow</td>
      <td></td>
      <td>NaN</td>
      <td>4.45</td>
      <td>787.0</td>
      <td>NaN</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>9</td>
      <td>19148900</td>
      <td>Understanding Machine Learning</td>
      <td>Shai Shalev-Shwartz</td>
      <td>9781107057135</td>
      <td>NaN</td>
      <td>4.30</td>
      <td>410.0</td>
      <td>2014.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>5</td>
      <td>22535757</td>
      <td>Bayesian Methods for Hackers: Probabilistic Pr...</td>
      <td>Cameron Davidson-Pilon</td>
      <td>9780133902839</td>
      <td>NaN</td>
      <td>4.15</td>
      <td>256.0</td>
      <td>2014.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>37</td>
      <td>33279921</td>
      <td>Algorithms to Live By: The Computer Science of...</td>
      <td>Brian Christian</td>
      <td>9780007547999</td>
      <td>NaN</td>
      <td>4.15</td>
      <td>368.0</td>
      <td>2016.0</td>
      <td>NaT</td>
      <td>2020-07-18</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.032877</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>3</td>
      <td>14514306</td>
      <td>Think Python</td>
      <td>Allen B. Downey</td>
      <td>9781449330729</td>
      <td>NaN</td>
      <td>4.10</td>
      <td>300.0</td>
      <td>2002.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Religion
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>139</td>
      <td>173666</td>
      <td>Radical Acceptance: Embracing Your Life With t...</td>
      <td>Tara Brach</td>
      <td>9780553380996</td>
      <td>NaN</td>
      <td>4.21</td>
      <td>333.0</td>
      <td>2000.0</td>
      <td>NaT</td>
      <td>2020-01-16</td>
      <td>to-read</td>
      <td>0</td>
      <td>Religion</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.536986</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>524</td>
      <td>30933</td>
      <td>Brideshead Revisited</td>
      <td>Evelyn Waugh</td>
      <td>9780316926348</td>
      <td>NaN</td>
      <td>4.00</td>
      <td>351.0</td>
      <td>1945.0</td>
      <td>NaT</td>
      <td>2016-05-04</td>
      <td>to-read</td>
      <td>0</td>
      <td>Religion</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>4.241096</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Philosophy
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>107</td>
      <td>39669805</td>
      <td>Enlightenment Now: The Case for Reason, Scienc...</td>
      <td>Steven Pinker</td>
      <td>9780141979090</td>
      <td>NaN</td>
      <td>4.23</td>
      <td>576.0</td>
      <td>2018.0</td>
      <td>NaT</td>
      <td>2020-02-06</td>
      <td>to-read</td>
      <td>0</td>
      <td>Philosophy</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.479452</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>121</td>
      <td>30659</td>
      <td>Meditations</td>
      <td>Marcus Aurelius</td>
      <td>9780140449334</td>
      <td>NaN</td>
      <td>4.23</td>
      <td>303.0</td>
      <td>180.0</td>
      <td>NaT</td>
      <td>2019-02-25</td>
      <td>to-read</td>
      <td>0</td>
      <td>Philosophy</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.427397</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>112</td>
      <td>43532557</td>
      <td>Outgrowing God: A Beginner’s Guide</td>
      <td>Richard Dawkins</td>
      <td>9781787631212</td>
      <td>NaN</td>
      <td>4.01</td>
      <td>304.0</td>
      <td>2019.0</td>
      <td>NaT</td>
      <td>2019-10-15</td>
      <td>to-read</td>
      <td>0</td>
      <td>Philosophy</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.791781</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Psychology
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>164</td>
      <td>6265779</td>
      <td>The Mindful Path to Self-Compassion: Freeing Y...</td>
      <td>Christopher K. Germer</td>
      <td>9781593859756</td>
      <td>NaN</td>
      <td>4.16</td>
      <td>306.0</td>
      <td>2009.0</td>
      <td>NaT</td>
      <td>2019-12-20</td>
      <td>to-read</td>
      <td>0</td>
      <td>Psychology</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.610959</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>108</td>
      <td>11848476</td>
      <td>Mindsight: Transform Your Brain with the New S...</td>
      <td>Daniel J. Siegel</td>
      <td>9781851687930</td>
      <td>NaN</td>
      <td>4.15</td>
      <td>314.0</td>
      <td>2009.0</td>
      <td>NaT</td>
      <td>2020-02-06</td>
      <td>to-read</td>
      <td>0</td>
      <td>Psychology</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.479452</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>116</td>
      <td>10865206</td>
      <td>The Willpower Instinct: How Self-Control Works...</td>
      <td>Kelly McGonigal</td>
      <td>9781583334386</td>
      <td>NaN</td>
      <td>4.15</td>
      <td>275.0</td>
      <td>2011.0</td>
      <td>NaT</td>
      <td>2019-06-28</td>
      <td>to-read</td>
      <td>0</td>
      <td>Psychology</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.090411</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>110</td>
      <td>66354</td>
      <td>Flow: The Psychology of Optimal Experience</td>
      <td>Mihaly Csikszentmihalyi</td>
      <td>9780060920432</td>
      <td>NaN</td>
      <td>4.11</td>
      <td>303.0</td>
      <td>1990.0</td>
      <td>NaT</td>
      <td>2020-01-24</td>
      <td>to-read</td>
      <td>0</td>
      <td>Psychology</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.515068</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>111</td>
      <td>16248196</td>
      <td>The Art of Thinking Clearly</td>
      <td>Rolf Dobelli</td>
      <td>9780062219688</td>
      <td>NaN</td>
      <td>3.85</td>
      <td>384.0</td>
      <td>2011.0</td>
      <td>NaT</td>
      <td>2020-01-24</td>
      <td>to-read</td>
      <td>0</td>
      <td>Psychology</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.515068</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Young Adult Fiction
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>276</td>
      <td>42441881</td>
      <td>The Missing of Clairdelune (The Mirror Visitor...</td>
      <td>Christelle Dabos</td>
      <td></td>
      <td>NaN</td>
      <td>4.56</td>
      <td>540.0</td>
      <td>2015.0</td>
      <td>NaT</td>
      <td>2018-09-21</td>
      <td>to-read</td>
      <td>0</td>
      <td>Young Adult Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.857534</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>263</td>
      <td>34728667</td>
      <td>Children of Blood and Bone (Legacy of Orïsha, #1)</td>
      <td>Tomi Adeyemi</td>
      <td>9781250170972</td>
      <td>NaN</td>
      <td>4.14</td>
      <td>544.0</td>
      <td>2018.0</td>
      <td>NaT</td>
      <td>2019-07-31</td>
      <td>to-read</td>
      <td>0</td>
      <td>Young Adult Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.000000</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>280</td>
      <td>37586</td>
      <td>The Lost Colony (Artemis Fowl #5)</td>
      <td>Eoin Colfer</td>
      <td></td>
      <td>NaN</td>
      <td>4.00</td>
      <td>385.0</td>
      <td>2006.0</td>
      <td>NaT</td>
      <td>2019-07-04</td>
      <td>to-read</td>
      <td>0</td>
      <td>Young Adult Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.073973</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>512</td>
      <td>31122</td>
      <td>I Capture the Castle</td>
      <td>Dodie Smith</td>
      <td>9780312181109</td>
      <td>NaN</td>
      <td>4.00</td>
      <td>343.0</td>
      <td>1948.0</td>
      <td>NaT</td>
      <td>2016-05-04</td>
      <td>to-read</td>
      <td>0</td>
      <td>Young Adult Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>4.241096</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    Science
    


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>113</td>
      <td>41552709</td>
      <td>The Uninhabitable Earth: Life After Warming</td>
      <td>David Wallace-Wells</td>
      <td>9780525576709</td>
      <td>NaN</td>
      <td>4.09</td>
      <td>310.0</td>
      <td>2019.0</td>
      <td>NaT</td>
      <td>2019-08-28</td>
      <td>to-read</td>
      <td>0</td>
      <td>Science</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.923288</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>322</td>
      <td>100005</td>
      <td>In Search of Schrödinger's Cat</td>
      <td>John Gribbin</td>
      <td>9780552125550</td>
      <td>NaN</td>
      <td>4.04</td>
      <td>302.0</td>
      <td>1984.0</td>
      <td>NaT</td>
      <td>2019-03-15</td>
      <td>to-read</td>
      <td>0</td>
      <td>Science</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.378082</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>335</td>
      <td>9593</td>
      <td>Galápagos</td>
      <td>Kurt Vonnegut Jr.</td>
      <td>9780385333870</td>
      <td>NaN</td>
      <td>3.88</td>
      <td>324.0</td>
      <td>1985.0</td>
      <td>NaT</td>
      <td>2019-02-25</td>
      <td>to-read</td>
      <td>0</td>
      <td>Science</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.427397</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>126</td>
      <td>36356862</td>
      <td>A Map of the Invisible: Journeys into Particle...</td>
      <td>Jon Butterworth</td>
      <td></td>
      <td>NaN</td>
      <td>3.63</td>
      <td>289.0</td>
      <td>2018.0</td>
      <td>NaT</td>
      <td>2018-10-17</td>
      <td>to-read</td>
      <td>0</td>
      <td>Science</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.786301</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


## 'My Rating' for each book

### Rating vs. Genre

We can see from the kernel density plot below that there is a clear difference in the way fiction books are rated compared to non-fiction. The median rating given for fiction is 5 by a clear margin where are for non-fiction it is a 4. Maybe non-fiction books 


```python
fig, ax = plt.subplots(figsize=(6, 5))
sns.kdeplot(data[data['genre']=='Fiction']['My Rating'], shade=True, label='Fiction')
sns.kdeplot(data[data['genre']=='Non-fiction']['My Rating'], shade=True, label='Non-fiction')
# ax.set_xlabel('My rating')
ax.set_xticks(range(1, 6))
ax.set_xticklabels(['1 star', '2 stars', '3 stars', '4 stars', '5 stars'])
ax.set_title('Distribution of ratings per genre')
plt.show()
```


![png](output_54_0.png)


## Contemplation

How long is spent contemplating the books? How long is there between date added and date read?

For this, we'll first need to create a new column which simply shows the length of time between books being added to my shelves and them either being read or staying in my 'to-read' shelf.

Some things to consider:
- Some books have been read but don't have a 'Date Read' value - this is from books I read before signing up with Goodreads
- Books not on the 'Read' shelf will have a contemplation time between today and the 'Date added' value
- Books on the 'Read' shelf without a 'Date read' value will be excluded
- Some books have a 'Date Read' before 'Date Added', this is from when I have decided to manually add a date for a date before I joined Goodreads.

We'll define a function to create this logic and then apply it to the column.


```python
today = dt.datetime.today()

def contemplation(row):
    if row['Exclusive Shelf'] != 'read':
        return (today - row['Date Added']).days / 365
    else:
        if row['Date Read']:
            if ((row['Date Read'] - row['Date Added']).days >= 0) & (row['Read Count'] < 2):
                return (row['Date Read'] - row['Date Added']).days / 365
            else:
                return np.nan
        else:
            return np.nan
        
data['contemplation'] = data.apply(contemplation, axis=1)
data.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>2049823</td>
      <td>Superior Saturday (The Keys to the Kingdom, #6)</td>
      <td>Garth Nix</td>
      <td>9780439700894</td>
      <td>4.0</td>
      <td>3.92</td>
      <td>336.0</td>
      <td>2008.0</td>
      <td>2020-03-30</td>
      <td>2020-07-28</td>
      <td>read</td>
      <td>1</td>
      <td>Juvenile Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>1</td>
      <td>17874847</td>
      <td>The Signal and the Noise: The Art and Science ...</td>
      <td>Nate Silver</td>
      <td>9780141975658</td>
      <td>4.0</td>
      <td>3.98</td>
      <td>534.0</td>
      <td>2012.0</td>
      <td>2020-06-26</td>
      <td>2018-03-16</td>
      <td>read</td>
      <td>1</td>
      <td>Mathematics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>2.282192</td>
    </tr>
    <tr>
      <td>2</td>
      <td>12042357</td>
      <td>Think Stats</td>
      <td>Allen B. Downey</td>
      <td>9781449307110</td>
      <td>NaN</td>
      <td>3.63</td>
      <td>138.0</td>
      <td>2011.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
    </tr>
    <tr>
      <td>3</td>
      <td>14514306</td>
      <td>Think Python</td>
      <td>Allen B. Downey</td>
      <td>9781449330729</td>
      <td>NaN</td>
      <td>4.10</td>
      <td>300.0</td>
      <td>2002.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Computers</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
    </tr>
    <tr>
      <td>4</td>
      <td>18711042</td>
      <td>Think Bayes</td>
      <td>Allen B. Downey</td>
      <td></td>
      <td>NaN</td>
      <td>3.85</td>
      <td>210.0</td>
      <td>2012.0</td>
      <td>NaT</td>
      <td>2020-07-26</td>
      <td>to-read</td>
      <td>0</td>
      <td>Mathematics</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>0.010959</td>
    </tr>
  </tbody>
</table>
</div>




```python
fig, ax = plt.subplots(figsize=(10, 6))
sns.kdeplot(data['contemplation'], shade=True, legend=False)
ax.set_xlabel('Number of years contemplating')
ax.set_ylabel('Frequency')
ax.set_title('Kernel Density plot of contemplation time')
sns.despine(offset=10, trim=True)
plt.show()
```


![png](output_57_0.png)


### Contemplation time vs. Number of Pages

Do longer books have a longer hesitation time.


```python
bins = [0, 300, 500, np.inf]
labels = ['short', 'medium', 'long']
data['book_length'] = pd.cut(data['Number of Pages'], bins, labels=labels)

fig, ax = plt.subplots(figsize=(10, 6))
sns.kdeplot(data=data[data['book_length']=='short']['contemplation'], label='Less than 300 pages')
sns.kdeplot(data=data[data['book_length']=='medium']['contemplation'], label='Between 300 and 500 pages')
sns.kdeplot(data=data[data['book_length']=='long']['contemplation'], label='More than 500 pages')
ax.set_xlabel('Number of years contemplating')
ax.set_ylabel('Frequency')
ax.set_title('Distribution of contemplation time')
sns.despine(offset=10, trim=True)
plt.show()
```


![png](output_59_0.png)


It's clear from this graph that books that are generally shorter spend less time waiting before being read.


```python
bins = [0, 300, 500, np.inf]
labels = ['short', 'medium', 'long']
data['book_length'] = pd.cut(data['Number of Pages'], bins, labels=labels)

fig, ax = plt.subplots(figsize=(10, 6))
data.groupby('book_length')['contemplation'].mean().plot(kind='barh')
ax.set_ylabel('Book length')
ax.set_xlabel('Time contemplating')
ax.set_xticks([0, 1, 2])
ax.set_xticklabels(['0', '1 year', '2 years'])
plt.show()
```


![png](output_61_0.png)


### Contemplation vs. Genre 

There is not such a distinction here. It seems fiction books might sit for slightly longer on my shelf.


```python
fig, ax = plt.subplots(figsize=(10, 6))
data.groupby('genre')['contemplation'].mean().plot(kind='barh')
ax.set_ylabel('')
ax.set_xlabel('Years contemplating')
ax.set_title('Average time spent contemplating before reading')
plt.show()
```


![png](output_64_0.png)


### My Rating vs Contemplation 

Do books mature with age. Are the books that I've left stewing on my shelves more fulfilling a read. How does the contemplation time affect the rating given.


```python
data['My Rating'].replace(0, np.nan, inplace=True)
fig, ax = plt.subplots(figsize=(10, 6))
sns.scatterplot(x='contemplation', y='My Rating', data=data)
ax.set_xlabel('')
ax.set_ylabel('')
ax.set_yticks([1, 2, 3, 4, 5])
ax.set_yticklabels(['1 star', '2 stars', '3 stars', '4 stars', '5 stars'])
ax.set_xticks([0, 1, 2, 3, 4, 5])
ax.set_xticklabels([0, '1 year', '2 years', '3 years', '4 years', '5 years'])
ax.set_title('Years contemplated vs. rating given')

# ax.annotate("Zen and the Art\nof Motorcycle Maintenance", 
#             xy=(data[data['Title'].str.contains('Zen and the Art of Motorcycle')]['contemplation'],
#                 data[data['Title'].str.contains('Zen and the Art of Motorcycle')]['My Rating']), 
#             ha='left', textcoords='offset points', xytext=(10, 0)
#            )

plt.show()
```


![png](output_66_0.png)



```python
fig, ax = plt.subplots(figsize=(10, 6))
sns.swarmplot(x='My Rating', y='contemplation', data=data[data['My Rating']>0])
ax.set_xlabel('')
ax.set_ylabel('')
ax.set_xticklabels(['1 star', '2 stars', '3 stars', '4 stars', '5 stars'])
ax.set_yticks([0, 1, 2, 3, 4, 5])
ax.set_yticklabels([0, '1 year', '2 years', '3 years', '4 years', '5 years'])
ax.set_title('Years contemplated vs. rating given')
plt.show()
```

    C:\Users\ryanf\Anaconda3\lib\site-packages\seaborn\categorical.py:1326: RuntimeWarning: invalid value encountered in less
      off_low = points < low_gutter
    C:\Users\ryanf\Anaconda3\lib\site-packages\seaborn\categorical.py:1330: RuntimeWarning: invalid value encountered in greater
      off_high = points > high_gutter
    


![png](output_67_1.png)



```python
from scipy.stats import pearsonr
corr = pearsonr(data[(~data['My Rating'].isnull()) & (~data['contemplation'].isnull())]['contemplation'], data[(~data['My Rating'].isnull()) & (~data['contemplation'].isnull())]['My Rating'])
print(f'There\'s no apparent correlation here with a pearson r correlation coefficient of {corr[0]:.2f}.')
```

    There's no apparent correlation here with a pearson r correlation coefficient of -0.03.
    

There doesn't seem to be a sort of ageing benefit for books on my shelf apparent based on that correlation coefficient. There's no clear link between time spent contemplating a book and a good rating. However, a couple of books certainly stick out for having sat on my shelf for a long time but offering a poor reading experience.

There do seem to be a couple of let-downs here. Books with long contemplation times of 3 years but given only a 1 or 2 star. Which are they? 

To see which books have had a long contemplation period followed by a disappointing read, we'll create a feature, the `let_down_score`. 

This is a simple calculation of the rating I gave minus to contemplation time. However, to give extra weight to higher ratings, I've added a slight exponentional function to the 'My Rating' part of the calculation.

$$Let down score = Rating^{1.2} - Contemplation Time$$

The threshold before a book is declared a let down is 0. Anything below this is classified as a let down.

The bump given to the rating means that a 5* rating can bring back a book from the brink of being a let down up until 7 or so years have passed.

Without further ado, here are the let-downs.


```python
data['let_down_score'] = data['My Rating']**1.2 - data['contemplation']
data.sort_values('let_down_score', ascending=False).head()

let_down_threshold = 0

data.loc[data['let_down_score'] < let_down_threshold].sort_values('let_down_score', ascending=True)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>60</td>
      <td>629</td>
      <td>Zen and the Art of Motorcycle Maintenance: An ...</td>
      <td>Robert M. Pirsig</td>
      <td>9780060589462</td>
      <td>1.0</td>
      <td>3.77</td>
      <td>540.0</td>
      <td>1974.0</td>
      <td>2020-05-13</td>
      <td>2016-04-11</td>
      <td>read</td>
      <td>1</td>
      <td>Psychology</td>
      <td>en</td>
      <td>Fiction</td>
      <td>4.090411</td>
      <td>long</td>
      <td>-3.090411</td>
    </tr>
    <tr>
      <td>91</td>
      <td>17810</td>
      <td>In the Miso Soup</td>
      <td>Ryū Murakami</td>
      <td>9780143035695</td>
      <td>2.0</td>
      <td>3.59</td>
      <td>224.0</td>
      <td>1997.0</td>
      <td>2020-03-20</td>
      <td>2017-03-01</td>
      <td>read</td>
      <td>1</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>3.054795</td>
      <td>short</td>
      <td>-0.757398</td>
    </tr>
    <tr>
      <td>105</td>
      <td>4980</td>
      <td>Breakfast of Champions</td>
      <td>Kurt Vonnegut Jr.</td>
      <td>9780385334204</td>
      <td>3.0</td>
      <td>4.07</td>
      <td>303.0</td>
      <td>1973.0</td>
      <td>2020-02-22</td>
      <td>2015-10-25</td>
      <td>read</td>
      <td>1</td>
      <td>English fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>4.331507</td>
      <td>medium</td>
      <td>-0.594314</td>
    </tr>
  </tbody>
</table>
</div>



'Zen and the Art of Motorcycle Maintenance' was a huge letdown and it tells me the score is working. I was completely baffled by the book and only read it based on a recommendation.

We've so far had only had a small number of let-downs. However, there are books which have sat on my shelf for long time now, meaning they are at risk of falling into the let-down hall of shame.


```python
print('POTENTIAL LET-DOWNS:\n')

for index, book in data[data['contemplation'] > 5].sort_values('contemplation', ascending=False).iterrows():
    print('- {}, by {}\n\tContemplation time: {:.3} years\n\tPages: {:.0f}'.format(book['Title'], book['Author'], book['contemplation'], book['Number of Pages']))
```

    POTENTIAL LET-DOWNS:
    
    - Anna Karenina, by Leo Tolstoy
    	Contemplation time: 6.1 years
    	Pages: 964
    - The Decline and Fall of the Roman Empire, by Edward Gibbon
    	Contemplation time: 5.8 years
    	Pages: 3760
    - Shantaram, by Gregory David Roberts
    	Contemplation time: 5.38 years
    	Pages: 936
    

6 years of contemplation! These books better be good. The number of pages for each of these books could be a big factor in why I've spent so long contemplating them.

### Respecting the elderly

Are older books given better ratings? Do they get the respect they deserve?

Older books are still being read because they are of a high quality so there is a bias here, but what does the data say?


```python
fig, ax = plt.subplots(figsize=(10, 6))
sns.scatterplot(x='Original Publication Year', y='My Rating', data=data)
plt.show()
```


![png](output_75_0.png)



```python
fig, ax = plt.subplots(figsize=(10, 6))
sns.scatterplot(x='Original Publication Year', y='My Rating', data=data[data['Original Publication Year']>0])
plt.show()
```


![png](output_76_0.png)



```python
data.loc[data['Original Publication Year']<1800].sort_values('Original Publication Year')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Book Id</th>
      <th>Title</th>
      <th>Author</th>
      <th>ISBN13</th>
      <th>My Rating</th>
      <th>Average Rating</th>
      <th>Number of Pages</th>
      <th>Original Publication Year</th>
      <th>Date Read</th>
      <th>Date Added</th>
      <th>Exclusive Shelf</th>
      <th>Read Count</th>
      <th>categories</th>
      <th>language</th>
      <th>genre</th>
      <th>contemplation</th>
      <th>book_length</th>
      <th>let_down_score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>632</td>
      <td>437433</td>
      <td>The Iliad: A New Prose Translation</td>
      <td>Homer</td>
      <td>9780140444445</td>
      <td>4.0</td>
      <td>3.87</td>
      <td>459.0</td>
      <td>-750.0</td>
      <td>2014-10-17</td>
      <td>2014-09-24</td>
      <td>read</td>
      <td>1</td>
      <td>Poetry</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.063014</td>
      <td>medium</td>
      <td>5.215018</td>
    </tr>
    <tr>
      <td>117</td>
      <td>10534</td>
      <td>The Art of War</td>
      <td>Sun Tzu</td>
      <td>9781590302255</td>
      <td>NaN</td>
      <td>3.97</td>
      <td>273.0</td>
      <td>-500.0</td>
      <td>NaT</td>
      <td>2019-06-21</td>
      <td>to-read</td>
      <td>0</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>1.109589</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>580</td>
      <td>592167</td>
      <td>The Conquest of Gaul</td>
      <td>Gaius Julius Caesar</td>
      <td>9780140444339</td>
      <td>NaN</td>
      <td>4.00</td>
      <td>269.0</td>
      <td>-50.0</td>
      <td>NaT</td>
      <td>2015-12-06</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>4.652055</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>579</td>
      <td>56781</td>
      <td>The Civil War</td>
      <td>Gaius Julius Caesar</td>
      <td>9780140441871</td>
      <td>NaN</td>
      <td>4.06</td>
      <td>368.0</td>
      <td>-47.0</td>
      <td>NaT</td>
      <td>2015-12-06</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>4.652055</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>121</td>
      <td>30659</td>
      <td>Meditations</td>
      <td>Marcus Aurelius</td>
      <td>9780140449334</td>
      <td>NaN</td>
      <td>4.23</td>
      <td>303.0</td>
      <td>180.0</td>
      <td>NaT</td>
      <td>2019-02-25</td>
      <td>to-read</td>
      <td>0</td>
      <td>Philosophy</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>1.427397</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>616</td>
      <td>18135</td>
      <td>Romeo and Juliet</td>
      <td>William Shakespeare</td>
      <td>9780743477116</td>
      <td>5.0</td>
      <td>3.75</td>
      <td>368.0</td>
      <td>1595.0</td>
      <td>2013-01-01</td>
      <td>2015-02-22</td>
      <td>read</td>
      <td>1</td>
      <td>Miniature books</td>
      <td>en</td>
      <td>Fiction</td>
      <td>NaN</td>
      <td>medium</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>694</td>
      <td>13006</td>
      <td>Julius Caesar</td>
      <td>William Shakespeare</td>
      <td>9780198320272</td>
      <td>4.0</td>
      <td>3.69</td>
      <td>128.0</td>
      <td>1599.0</td>
      <td>NaT</td>
      <td>2013-09-16</td>
      <td>read</td>
      <td>1</td>
      <td>Juvenile Nonfiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>NaN</td>
      <td>short</td>
      <td>NaN</td>
    </tr>
    <tr>
      <td>590</td>
      <td>2932</td>
      <td>Robinson Crusoe (Robinson Crusoe #1)</td>
      <td>Daniel Defoe</td>
      <td></td>
      <td>4.0</td>
      <td>3.67</td>
      <td>320.0</td>
      <td>1719.0</td>
      <td>2015-10-15</td>
      <td>2013-03-29</td>
      <td>read</td>
      <td>1</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>2.547945</td>
      <td>medium</td>
      <td>2.730086</td>
    </tr>
    <tr>
      <td>648</td>
      <td>7733</td>
      <td>Gulliver's Travels</td>
      <td>Jonathan Swift</td>
      <td>9780141439495</td>
      <td>3.0</td>
      <td>3.57</td>
      <td>306.0</td>
      <td>1726.0</td>
      <td>2014-06-23</td>
      <td>2014-05-29</td>
      <td>read</td>
      <td>1</td>
      <td>Fiction</td>
      <td>en</td>
      <td>Fiction</td>
      <td>0.068493</td>
      <td>medium</td>
      <td>3.668700</td>
    </tr>
    <tr>
      <td>120</td>
      <td>10224776</td>
      <td>The Decline and Fall of the Roman Empire</td>
      <td>Edward Gibbon</td>
      <td>9780307700766</td>
      <td>NaN</td>
      <td>3.96</td>
      <td>3760.0</td>
      <td>1776.0</td>
      <td>NaT</td>
      <td>2014-10-13</td>
      <td>to-read</td>
      <td>0</td>
      <td>History</td>
      <td>en</td>
      <td>Non-fiction</td>
      <td>5.800000</td>
      <td>long</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
sns.pairplot(data, hue='genre', diag_kind='kde')
```




    <seaborn.axisgrid.PairGrid at 0x22c8e47add8>




![png](output_78_1.png)

