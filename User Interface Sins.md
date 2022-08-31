*(This is a work in progress.)*

# User Interface Sins

There are a lot of bad user interfaces out there. This disturbing trend began with the movement from the classic SAA (stand-alone application) to browser based apps. The problem is that HTML and JavaScript, the languages of web pages, seem to designed to prevent goood, flexible, intuitive, and fast user interfaces. I will be going into reasons later, but here I want to catalog some common UI idioms in browser apps. There are a lot of them.

## Paging through reams of data

We've all seen this. You have 2000 records to look through. In an SSA, you'll usually see it all in a table, and you would be able to sort by any column, and scroll through the entire table, so you can move through the fields quickly.

But in a browser, you'll have to page through them. The data won't be in a table, and there may be 20 on a "page," but you'll only be able to see maybe 7 of them. You need to scroll down to see the rest, but when you reach the end, you hit ''next'' to see the next 20. And it takes time to load them. So you have to deal with two dimensions of data, even though there's only one dimension. In a better UI, if only 7 records fit on your screen, each "page" would have only 7 entries. But that improved visual design clashes with the need to move quickly, since the more records there are on each page, the less time you'll waste waiting for the each page to load.

## Sorting through reams of data.

Sometimes, the data is split, with a column of records that you can click on, and a page of data for each record. This is useful, but what if you want the data sorted by, say, the "assignee" field? If that field doesn't appear in the column of records, you will get the data in that order, but you won't be able to scan through the data to see where the assignee changes from one user to another. 

(I will be illustrating these examples with graphics when I have the time.)
