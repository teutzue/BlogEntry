# BlogEntry

	
##Overview

	By reading this blog post you can find discussions based on credible sources, theoretical analysis and the design of the solution for how to deal with big amount of data in Neo4j, avoid RAM overflow, bottlenecks when it comes to aggregated and ordered data. Addressing these serious issues can bring new life to your database and enable almost unlimited scaling. 
				
##Reliable Data from Credible Sources 

Although the problem we talk about in this article is described, analysed and has some solutions proposed on multiple websites (this fact shows that the identified matter is real and of interest for a lot of programmers), after a careful investigation, we have selected only a few sources which offer a complete analysis and verified solutions based on knowledge and experience. 
The problem identified by us is acknowledged to be valid by Louise Söderström, one of the main contributors to the Neo4j database who considers the fix a top priority for future releases. Moreover, by reading the Neo4j manual, we identified that the “order by” clause doesn’t use indexes although this fact is not explicitly said (https://neo4j.com/docs/developer-manual/current/cypher/clauses/order-by/, accessed on 10th of December 2017).
Meanwhile, Andrey Nikishaev has dedicated almost 1 year to studying the problem in production identifying valid solutions (104 upvotes) in practice (Andrey Nikishaev -“Life after 1 year of using Neo4J”,https://hackernoon.com/life-after-1-year-of-using-neo4j-4eca5ce95bf5 accesed on the 10th of December 2017). Definitely basing himself on knowledge and experience, he also proposes a way to perform deletions of big amount of data - task which is directly linked to RAM overflow.
The same author, in another source, reveals that the absence of a query watcher is generating RAM overflow hazards stressing that it’s the responsibility of the developer to write better optimized queries to prevent database crashes (https://www.slideshare.net/anikishaev/neo4j-after-1-year-in-production, accessed on 10th of December 2017).

Design constraints, alternatives and assumptions 

	The database which exposes problems is part of a bigger system that is meant to be a HackerNews clone (https://news.ycombinator.com) and fulfills the following features:

Displays a set of stories or comments (on comments) on the system's front page
Stories or comments are posted by users, which have to be registered to and logged into the system to be able to post
Uses REST API
The application has to be fast enough not to cause a poor user experience.

	For reaching this design goal, a database which is capable of storing, fast traversing and retrieving highly connected data is required. Moreover, it has to be robust and prove itself reliable when the dataset grows so we do not run into scalability problems. The website would require the system to process millions of data entries without any crashes.
	In these situation, the alternatives were SQL databases (relational) and MongoDB (document oriented, schema less). On one hand, SQL databases perform weakly when multiple joins have to be executed and in our case, the comment on comment feature would cause problems. On the other hand, MongoDB runs very well when the schema changes a lot which again, is not the case for us.
	In contrast with all the competitor databases, Neo4j has the advantage of being good at retrieving linked data, fast for traversing and retrieving information and scales without problems according to their claims (https://neo4j.com/product/ , accessed on 10th of December 2017). Backed up by theory we assumed that Neo4j was a good choice as our main data source but later on problems appeared when dealing with millions of records. The manual describes that neo is very fast but in our use case we found the one situation in which it slows down as the dataset grows big. 

Theoretical explanations

	The analysis of the database’s performance problems led us to the following explanations about RAM memory usage and Order by clause:

  Neo4j databases use big amounts of RAM to perform operations because the internal implementation does not use a query watcher which puts more pressure on the developer to carefully design efficient queries.

  The “Order by” clause does not make use of indexes determining data to be read from the hard drive which causes queries to exhibit poor performance. On the other hand, indexed data is always ready to be used residing in the cache memory because it has to be accessed more often. A possible solution for this matter is to use a “where” statement which limits the amount of data in the query.

  When it comes to querying aggregated data Neo4j is sluggish because data can not be stored in a sorted matter. Thus, every time such a query is invoked Neo4j has to store all information in memory and then return the data needed.


  Observation: The Neo4j manual does not explain properly how the order by is being implemented, if it uses indexes or not. Usually databases than can be used on large scale systems have this feature implemented when possible. The order by clause is not complex and the fact that it doesn’t have indexes can be only noticed when the data set grows. In cases like this one, data has to be migrated into smaller, manageable subsets. 


Design criteria, sample calculations and Simulations


During a simulation the following criteria has to be met:

- Use a dataset of minimum 5 million timestamped entries
- Store data as it is without subsets
- Retrieve data based on the timestamp in an ordered manner
- Observe poor performance when as data grows

	Below we analyse the behaviour of 2 similar queries which perform differently performance-wise. Because the way they are structured, the first one has to work with bigger amount of data compared to the second one. In the parenthesis, the number of nodes in memory are displayed.
 
(post_parent:-1 means that we only get stories)
(Bad query ) 12s average, sometimes ram overflow
Match (par:Post{post_parent:-1})
with par ORDER BY par.timestamp desc skip (30*skip) limit limit 
with par
return par as post, size((par)-[:Parent *1..]->()) as numberOfcomments;

Number of nodes in total at this very moment: 7284148
Flow of the above: Get all nodes(7284148)->match stories(1479250)->order them(1479250)->skip skip amount of posts(1479250-skip)--> limit limit amount of posts(limit)->return posts(limit)->find and count their comments(limit)

Solution:
(Good query) 4,5 s average time no ram overflow
timeLimit is current time of writing this article-8 hours
Match (par:Post{post_parent:-1}) where par.timestamp>timeLimit
with par skip (30*skip) limit limit 
with par ORDER BY par.timestamp desc
return par as post, size((par)-[:Parent *1..]->()) as numberOfcomments;

Flow of the above: Get all nodes(7284148)->match stories(1479250)-> match stories newer than timeLimit (20631)->order by posts(20631)->skip skip amount of posts(20631-skip)-> limit limit amount of posts(limit)->return posts-(limit)>find and count their comments(limit)

The difference between first and second query is that the 2nd is performing ORDER BY on much smaller number of nodes(1479250 vs 20631). That is because the scope has been narrowed by WHERE clause. Paying close close attention, we can notice that in both queries neo4j is scanning all timestamps nevertheless. The key difference is that WHERE clause is utilizing indexes whereas ORDER BY is not.  


Previous work/future work 

	In this article we’ve analysed and listed credible sources which discuss and propose solutions for the RAM overflow, aggregated data and “order by” clause at the same time expressing our opinions about them. Later we presented our constraints and explanations related to the project in which the database problem is present. A series of explanations based on theory create the background for the simulation for the design of an applicable solution.

  In the future our attention will be on implementing better designs having in mind large data sets, finding more sources to rely on with the community’s help, implement/report new feature requests to enhance the capabilities of databases and last by not least, monitor and compare performance of databases simulating industrial use.


