---
layout: post
title:  "Iterate on a date range with Scala"
date:   2020-02-14 03:58:56 +0100
tags: [scala, spark]
---
Iterate over a date range in Scala, when your dates are string and when you output has also to be a string.

The title alwary says everything. I didn't find any code to copy&paste on StackOverflow to iterate on a range of dates using Scala to easily delete (or iterate over) partitions in Spark.

Lot of old posts and SO answers were suggesting to use joda-time, many also raised the point that from Java 8 we should use java.time.

I am pretty sure that other people had the same problem, but I was not able to fine an end-to-end solution, so here is the code:

```scala
import java.time.LocalDate
import java.time.format.DateTimeFormatter

val format = DateTimeFormatter.ofPattern("yyyy-MM-dd")
val start_dt = LocalDate.parse("2020-02-01", format)
val end_dt = LocalDate.parse("2020-02-20", format)
val date_diff = end_dt.toEpochDay() - start_dt.toEpochDay()

for (i <- 0l to date_diff by 1) {
  val s = start_dt.plusDays(i)
  println(s)
}
```

Don't forget that you can replace `to date_diff` with `until date_diff` to exclude the last date from your range.

In my case, the `println(s)` was replace by:
```scala
dbutils.fs.rm(f"$s3_path%s/dt=$dt%s", true)
```

I am writing this here just to google less in the future.
