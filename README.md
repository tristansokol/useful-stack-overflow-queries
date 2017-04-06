Here are some useful queries for the [Stack Exchange Data Explorer](https://data.stackexchange.com/)


###Response time, number of Questions, number of views for questions each month. 
```SQL
declare @FirstId int = (SELECT min(Id) FROM dbo.Posts WHERE PostTypeId = 1);
DECLARE @FirstQuestion datetime = (SELECT CreationDate FROM dbo.Posts WHERE Id = @FirstId);
DECLARE @LastQuestion datetime = (SELECT MAX(CreationDate) FROM dbo.Posts WHERE PostTypeId = 1);

SELECT TagName,
TTA = CASE WHEN MIN(c.CreationDate) > MIN(a.CreationDate) THEN 
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(a.CreationDate)
      ),
    9999999 * 60
  )
WHEN  MIN(c.CreationDate) < MIN(a.CreationDate) THEN
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(c.CreationDate)
      ),
    9999999 * 60
  )
WHEN MIN(c.CreationDate) IS NULL AND MIN(a.CreationDate) IS NOT NULL THEN
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(a.CreationDate)
      ),
    9999999 * 60
  )
  WHEN MIN(a.CreationDate) IS NULL AND MIN(c.CreationDate) IS NOT NULL THEN
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(c.CreationDate)
      ),
    9999999 * 60
  )
  ELSE 9999999
 END ,
  q.CreationDate,
  q.ViewCount,
  q.Id AS [Post Link],
  MIN(q.CreationDate) as QuestionDate,
  Min(c.CreationDate) as CommentDate,
  Min(a.CreationDate) as AnswerDate
INTO #AnswerTimes
FROM dbo.Posts q
     join PostTags on q.Id = PostId
     join Tags t on t.Id = TagId 
     left JOIN dbo.Comments c ON c.PostId = q.Id AND c.UserId <> q.OwnerUserId
     left JOIN dbo.Posts a ON a.ParentId = q.Id AND a.OwnerUserId <> q.OwnerUserId
     
WHERE q.PostTypeId = 1 
  AND q.CreationDate >= @FirstQuestion 
  AND q.ClosedDate IS NULL
  AND q.Score >= 0
  --and c.CreationDate > q.CreationDate
  and t.TagName in ('##tag##')
GROUP BY q.Id, q.CreationDate, q.ViewCount, TagName, q.Id
  
  -- SELECT * from #AnswerTimes
  
  
  SELECT
  MonthStart, round(MAX(Median), 0) hours, numberOfQuestions, SUM(numberOfViews) views
FROM 
(
  SELECT
    CONVERT(datetime, CONCAT('01.',MONTH(X.CreationDate),'.',YEAR(X.CreationDate)),104) AS MonthStart,
    X.ViewCount AS numberOfViews,
    COUNT(TTA) OVER (PARTITION BY dateadd(month, datediff(month, '20000101', X.CreationDate) , '20000101')) numberOfQuestions,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY TTA) OVER (PARTITION BY dateadd(month, datediff(month, '20000101', X.CreationDate) , '20000101')) / 3600.0 Median
  FROM #AnswerTimes X
) RelevantQuestionsByWeek
GROUP BY MonthStart, numberOfQuestions
ORDER BY MonthStart
```

Weekly
```SQL
declare @FirstId int = (SELECT min(Id) FROM dbo.Posts WHERE PostTypeId = 1);
DECLARE @FirstQuestion datetime = (SELECT CreationDate FROM dbo.Posts WHERE Id = @FirstId);
DECLARE @LastQuestion datetime = (SELECT MAX(CreationDate) FROM dbo.Posts WHERE PostTypeId = 1);

SELECT TagName,
TTA = CASE WHEN MIN(c.CreationDate) > MIN(a.CreationDate) THEN 
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(a.CreationDate)
      ),
    9999999 * 60
  )
WHEN  MIN(c.CreationDate) < MIN(a.CreationDate) THEN
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(c.CreationDate)
      ),
    9999999 * 60
  )
WHEN MIN(c.CreationDate) IS NULL AND MIN(a.CreationDate) IS NOT NULL THEN
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(a.CreationDate)
      ),
    9999999 * 60
  )
  WHEN MIN(a.CreationDate) IS NULL AND MIN(c.CreationDate) IS NOT NULL THEN
  ISNULL(
    DATEDIFF(
      second,
      q.CreationDate,
     MIN(c.CreationDate)
      ),
    9999999 * 60
  )
  ELSE 9999999
 END ,
  q.CreationDate,
  q.ViewCount,
  q.Id AS [Post Link],
  MIN(q.CreationDate) as QuestionDate,
  Min(c.CreationDate) as CommentDate,
  Min(a.CreationDate) as AnswerDate
INTO #AnswerTimes
FROM dbo.Posts q
     join PostTags on q.Id = PostId
     join Tags t on t.Id = TagId 
     left JOIN dbo.Comments c ON c.PostId = q.Id AND c.UserId <> q.OwnerUserId
     left JOIN dbo.Posts a ON a.ParentId = q.Id AND a.OwnerUserId <> q.OwnerUserId
     
WHERE q.PostTypeId = 1 
  AND q.CreationDate >= @FirstQuestion 
  AND q.ClosedDate IS NULL
  AND q.Score >= 0
  --and c.CreationDate > q.CreationDate
  and t.TagName in ('##tag##')
GROUP BY q.Id, q.CreationDate, q.ViewCount, TagName, q.Id
  
  -- SELECT * from #AnswerTimes
  
  
SELECT
WeekNum,Yr, round(MAX(Median), 0) hours, COUNT(numberOfQuestions), SUM(numberOfViews) views
FROM 
(
  SELECT
    --CONVERT(datetime, CONCAT('01.',MONTH(X.CreationDate),'.',YEAR(X.CreationDate)),104) AS MonthStart,
    DATEPART(ISO_WEEK,X.CreationDate) AS WeekNum,
    YEAR(X.CreationDate) AS Yr,
    X.ViewCount AS numberOfViews,
   -- COUNT(TTA) OVER (PARTITION BY dateadd(month, datediff(month, '20000101', X.CreationDate) , '20000101')) numberOfQuestions,
   1 AS numberOfQuestions,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY TTA) OVER (PARTITION BY dateadd(week, datediff(week, '20000101', X.CreationDate) , '20000101')) / 3600.0 Median
  FROM #AnswerTimes X
) RelevantQuestionsByWeek
GROUP BY WeekNum, Yr
ORDER BY Yr,WeekNum
```
