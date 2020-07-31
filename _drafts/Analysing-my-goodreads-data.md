---
layout: post
title:  "Analysing my Goodreads data"
categories: [books, python]
location: London, UK
location-link: london
---

Skills it shows: data cleaning, data manipulation, data visualisation, programming, api usage, data analysis

I have been curious to analyse my reading behaviour. For the last 7 years, I have been using Goodreads to rate books I've read and line up future reads by adding books to my 'to-read' shelf. This means that I have 7 years' worth of data on my reading behaviour.

Here I'll through some of the things I found out.

<!--description-->

## The dataset

<!-- TODO Add row from the data -->

Goodreads allow you to export your dataset through your account. Within this, key data exported includes:

- Book details - Title, author, ISBN, original publication year
- Goodreads platform data - Rating given by Goodreads members
- Personal reading data - Rating given by user, date read, date added to a shelf, shelf the book is currently on i.e. to-read, read, currently-reading

A key piece of information that I wanted to add to this was the genre or category that the book falls under. After a brief bit of research, it looked like the Google Books API would be the best way to add this information to my data. Even though the categories provided by the Books API are a bit arbitrary and inconsistent, after some cleaning, books were labelled either 'Fiction' or 'Non-fiction.

![Data sample]({{site.baseurl}}\assets\img\goodreads\goodreads_data_sample.jpg)

<!-- TODO: Google Books API -->

With this, we can start analysing the data.

## Exploring the data
<!-- TODO: EDA -->

### Reading habits over time

Unfortunately, one feature I would have liked to extract from Goodreads was the full reading time. When a book moves onto the 'Currently-reading' shelf, this date is logged. Using this with 'Date Read' we would have been able to calculate a reading time for each book. No doubt this information would have correlated closely with the number of pages in the book, but whether the book was fiction or non-fiction may also have had an impact.

Let's first plot the number of books read per month over time.
<!-- TODO: Books over time with genre stack -->




<!-- TODO: Top level statistics -->

<!-- TODO: Contemplation time and letdowns -->
<!-- TODO: Rating differences based on genre -->
<!-- TODO: Feature Engineering -->