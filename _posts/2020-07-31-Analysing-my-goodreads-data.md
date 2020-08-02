---
layout: post
title:  "Analysing my Goodreads data"
categories: [books, python]
location: London, UK
location-link: london
---

I've been curious to analyse my reading behaviour. For the last 7 years, I have been using Goodreads to rate books I've read and line up future reads by adding books to my 'to-read' shelf. This means that I have 7 years' worth of data on my reading behaviour.

Here I'll through some of the things I found out.

<!--description-->

## The dataset

Goodreads allow you to export your dataset through your account. Among the 31 columns in the data, we're interested in:

- Book details - Title, author, ISBN, original publication year
- Goodreads platform data - Rating given by Goodreads members
- Personal reading data - Rating given by user, date read, date added to a shelf, shelf the book is currently on i.e. to-read, read, currently-reading

A key piece of information that I wanted to add to this was the genre or category that the book falls under. After a brief bit of research, it looked like the Google Books API would be the best way to add this information to my data. Even though the categories provided by the Books API are a bit arbitrary and inconsistent, after some cleaning, books were labelled either 'Fiction' or 'Non-fiction.

![Data sample]({{site.baseurl}}\assets\img\goodreads\goodreads_data_sample.jpg)

<!-- TODO: Google Books API -->

With this, we can start analysing the data.

## Exploring the data

### Top level numbers

The dataset contains 808 books. Books are split between four shelves which are mutually exclusive: 'to-read', 'read', 'currently-reading' and 'abandoned' (a shelf I added to Goodreads standard three).

    read                    586
    to-read                 214
    currently-reading       5
    abandoned               3
    Name: Exclusive Shelf, dtype: int64

Fiction makes up 76% of the books listed on all shelves but only 57% of the 'read' shelf. Maybe I'm too unrealistic when it comes to my non-fiction reading.

My average rating for books I've read is 4.29 stars out of 5. Comparing this to the Goodreads average rating for the same set of books, we get 4.10. Maybe not a significant difference but it supports the feeling I have that in general I'm quite generous with my ratings.

The mean average number of pages per book is 388 pages. The median sits at 347 suggesting a potential bias added by some outliers, one potential outlier being Edward Gibbon's 3760 page, 6 volume history of the 'Decline and Fall of the Roman Empire'.

Again, looking at books I've read, the mean is 354 pages and median 335. A pattern is forming maybe suggesting a gap between what I'd like to have read and what I have actually read.

The mean publication year for the books on my shelves is 1969 which the median being 2001. Again there are some notable outliers, namely:

- 'The Iliad' by Homer (750BCE)
- 'The Art of War' by Sun Tzu (500BCE)
- 'The Conquest of Gaul' by Julius Caesar (50BCE)
- 'The Civil War' by Julius Caesar (47BCE)
- 'Meditations' by Marcus Aurelius (180)

Removing these 5 books from our 808 book dataset boosts our mean up to 1983 but keeps the median at 2001. Continuing the theme of me being overly ambitious, I've read just one of these ancient classics ('The Iliad') and the mean publication year for books I've read is a more modern 1980.

Now onto some visualisations.

### Reading habits over time

Unfortunately, one feature I would have liked to extract from Goodreads was the full reading time. When a book moves onto the 'Currently-reading' shelf, this date is logged. Using this with 'Date Read' we would have been able to calculate a reading time for each book. No doubt this information would have correlated closely with the number of pages in the book, but whether the book was fiction or non-fiction may also have had an impact.

Let's first plot the number of books read per month over time.
<!-- TODO: Books over time with genre stack -->

![Books read over time ><]({{site.baseurl}}\assets\img\goodreads\books_read.png)

First month of lockdown kicked off with a bang here with me reading 15 books, the highest tally over the past 5 years. There's also a general trend here of reading more and also more non-fiction books being read as time goes on.

This is even clearer if we look at books read on a yearly basis.

![Books read over time ><]({{site.baseurl}}\assets\img\goodreads\books_read_year.png)

I'm certainly reading a greater number of books, however am I picking shorter books up to get my numbers up?

![Books read over time ><]({{site.baseurl}}\assets\img\goodreads\books_read_vs_pages.png){: height='400px'}

There's a clear correlation here between the number of books read and the number of pages as we expect. If I was actually reading shorter books when reading more of them, we might expect the number of pages to stay fairly level or round off to a certain extent. Instead we have a Pearson R value of 0.86 suggesting a strong positive relationship here.

### How does genre affect the rating I give?

We can see from the below plot that a 5 star rating is the single most likely rating given to fiction books but 4 stars is the mode for non-fiction. Whilst a kernal density plot is not necessarily the best way to present discrete distributions, I think it works here as a way to clearly show the difference between the way fiction and non-fiction values have been rated up till now.

![Rating vs genre ><]({{site.baseurl}}\assets\img\goodreads\ratings_vs_genre.png){: height='400px'}

I also like to use the KDE plot as this rules out any complexity around using the correct number of bins that you would have in a histogram, the more typical method for plotting distribution of values.

### Contemplation time and the let-down score

Goodreads records the date a book was put onto any shelf and further records the date that a book was moved onto the 'read' shelf. From this I decided to create a `contemplation` column which is how long the book sat on the shelf before being read.

I joined Goodreads in 2012 and started logging my reading activity regularly in 2013. Books added to my account have on average sat around for a year before being read but let's look at the distribution of waiting time.

![Dist of contemplation time ><]({{site.baseurl}}\assets\img\goodreads\kde_contemplation.png){: height='400px'}

How is this influenced by different features of the book? Well, surprisingly, despite my having a higher proportion of non-fiction books on my 'to-read' shelf than my 'read' shelf, the genre of the book has no influence on the time I wait before reading it.

How thick the book is in terms of pages on the other hand, definitely seems to influence the time contemplating whether to read a book or not.

![Contemplation vs book length ><]({{site.baseurl}}\assets\img\goodreads\contemplation_vs_book_length.png){: height='400px'}

#### Do books get better with age?

Is it better to be patient and let a book stew for a bit before consuming it. Here I'm not talking about how old a book is but instead how long they have sat on my shelves. The short answer: no, not at all.

In my case, there appears to be no aging benefit for books on my shelf based on the below scatterplot.

![Contemplation vs book length ><]({{site.baseurl}}\assets\img\goodreads\contemplation_vs_rating.png){: height='400px'}

There does seem to be a bit of an upside down pyrmid going on here looking at this plot but there is no link here and that is most likely only down to the higher likelihood of me giving a better rating in general.

There are definitely a couple of data points that stick out here however, books with 3 or 4 year waiting times and then a miserable 1 or 2 star rating. To see which books had a long contemplation period followed by a disappointing reading experience, we'll create a new feature, the `let_down_score`.

This is a simple calculation of the rating I gave minus to contemplation time. However, to give extra weight to higher ratings, I've added a slight exponentional function to the 'My Rating' part of the calculation.

$$Let down score = Rating^{1.2} - Contemplation Time$$

The threshold before a book is declared a let down is 0. Anything below this is classified as a let down. The bump given to the rating means that a 5* rating can bring back a book from the brink of being a let down up until 7 or so years have passed.

Without further ado, here are the let-downs with a `let_down_score < 0`.

![Let downs ><]({{site.baseurl}}\assets\img\goodreads\let_downs.jpg)

'Zen and the Art of Motorcycle Maintenance' was a huge letdown and it tells me the score is working. I was completely baffled by the book and only read it based on a recommendation.

We've so far had only had a small number of let-downs. However, there are books which have sat on my shelf for long time now, meaning they are at risk of falling into the let-down hall of shame.

![Let downs ><]({{site.baseurl}}\assets\img\goodreads\potential_let_downs.jpg)

5+ years of contemplation! These books better be good. The number of pages for each of these books could be a big factor in why I've spent so long contemplating them.

## Conclusion

I thoroughly enjoyed this project and only wish I could have had a couple of extra data points to play with, mostly the reading time for books. Looking back over the time I've spent on this, I would split the time spent as follows:

- Data cleaning and manipulation - 10%
- Calling the Google Books API - 75%
- Data analysis and storytelling - 15%

Requesting extra information from the API was a new experience for me and it took quite some time. Handling HTTP 429 ('Too many requests') errors and working around getting some blank rows, creating a cache to not send the same requests multiple times all took me time to work around.

There is no action to take from this data but I have enjoyed analysing the different features and understanding my reading habits a little further. It will not influence the way I read whatsoever, but I enjoyed embedding my python skills a bit further.
