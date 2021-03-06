------------------------------------------------------------------------

layout: post
title: Multi-value LIKE parameters in SQL
—-

Thinking in SQL, thinking in terms of sets took me a while to achieve. In the beginning I often lapsed into procedural thinking and relied heavily on stored procedures. Often I find myself of learning one more feature that makes me re-evaluate how I have written SQL in the past.

Learning about [recursive common table expressions](http://msdn.microsoft.com/en-us/library/ms186243.aspx) made me change the way I queried hierarchial data. Table valued functions had a similar impact particularly when using [CROSS APPLY](http://www.sqlteam.com/article/using-cross-apply-in-sql-server-2005) . However sometimes it is just the combination of simple features that produce something interesting.

I was tasked with creating a report searching for people based on a search string that a user supplies. The search string is space separated and all the terms in the search need to be matched. So if a user supplies the search phrase “Jam Wa” then the query must return return a person named “Walker, James” and “James Walker” but not “Brian Walker”.

The solution is presented below. Once I realized that the rvalue of the LIKE operator could be a column expression then it was simple to use a common table expression MatchSet, a table of like clauses. The only trick was ensuring that the number of rows returned joined against MatchSet was equal to the number of rows in MatchSet. This ensured that all terms in the query were matched.

{% highlight sql %}

DECLARE `SearchString VARCHAR(50)
SET `SearchString = ‘Jam Wal’

WITH MatchSet(Match) AS (
SELECT
‘’ + Value + ’’ as Match
FROM
Warehouse.fnSplit(‘ ’, @SearchString)
WHERE Value != ’’
)
SELECT \* FROM tblPhysicalUnit WHERE ID IN (
SELECT
P.ID
FROM
tblPhysicalUnit P
JOIN MatchSet M ON P.DisplayString LIKE M.Match
GROUP BY P.ID
HAVING COUNT (\*) = (SELECT COUNT (\*) FROM MatchSet)
)

CREATE FUNCTION Warehouse.fnSplit
(
`sep CHAR(1),
  `s VARCHAR (512)
)
RETURNS table
AS
RETURN (
WITH Pieces(PartNumber, start, stop) AS (
SELECT 1, 1, CHARINDEX (@sep, @s)
UNION ALL
SELECT PartNumber + 1, stop + 1, CHARINDEX (@sep, @s, stop + 1)
FROM Pieces
WHERE stop &gt; 0
)
SELECT
PartNumber AS ID,
SUBSTRING (@s,
                start, 
                CASE
                  WHEN stop &gt; 0 THEN stop-start
                  ELSE 512
                END) AS Value
FROM Pieces
)
{% endhighlight %}

**Updated September 12, 2010:** Added fnSplit funciton.
