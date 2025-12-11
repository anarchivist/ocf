---
title: dataverse storage use by user
date: 2025-10-30T16:21:00-07:00
lastmod: 2025-12-10T18:44:55-08:00
type: post
subtitle: for when you don't have a quota
categories:
- dataverse
meta: true
hideReadTime: true
---

for a [faculty data storage allocation pilot](https://technology.berkeley.edu/research-storage), we needed to track the storage use by user within a dataverse collection. the existing [metrics API methods](https://guides.dataverse.org/en/latest/api/metrics.html) didn't work, nor did any of the existing [reporting tools and common queries](https://guides.dataverse.org/en/latest/admin/reporting-tools-and-queries.html). enter in handy dandy SQL.

there are ~~two~~ three approaches that i've rolled out depending on the accuracy needed. in our case, our dataverse only has two levels of hierarchies for the collections: the root collection, and then its subcollections.

## approach 1: by dataset

this is a "fast" approach, which uses the `storageuse` table (a realtime record, but can be disabled for performance sake). it aggregates all the datasets in a given collection, and then gathers the `storageuse` data for each dataset, grouping by use. {{< sidenote >}} the issue with this query is that it presumes that the user who has created the dataset record should have their storage tracked. {{< /sidenote >}}

```sql 
SELECT authenticateduser.email,
    SUM(storageuse.sizeinbytes) AS sizeinbytes,
    SUM(storageuse.sizeinbytes) / (1024 ^ 3) AS sizeingigabytes,
    SUM(storageuse.sizeinbytes) / (1024 ^ 4) AS sizeinterabytes
FROM dvobject
JOIN dataset ON dataset.id = dvobject.id
JOIN storageuse ON dataset.id = storageuse.dvobjectcontainer_id 
JOIN authenticateduser ON authenticateduser.id = dvobject.creator_id
WHERE dvobject.owner_id = $ID -- edit this for your collection's ID
GROUP BY authenticateduser.email;
```

## approach 2: by uploaded file

alternately, this approach finds all the datafiles within datasets in a collection{{< sidenote >}} the issue with this query is that it's not yet recursive if you have multiple levels of collections. {{< /sidenote >}}, and then groups and sums the file sizes by user.

```sql
WITH sfr_datafile AS (
    SELECT datafile.*,
        dvobject.creator_id,
        dvobject.owner_id 
    FROM datafile 
    JOIN dvobject on dvobject.id = datafile.id
    JOIN dataset on dataset.id = dvobject.owner_id
    JOIN dvobject AS dvds on dvds.id = dvobject.owner_id
    WHERE dvds.owner_id = $ID -- edit this for your collection's ID
)
SELECT authenticateduser.email,
    SUM(sfr_datafile.filesize) AS sizeinbytes,
    SUM(sfr_datafile.filesize) / (1024 ^ 3) AS sizeingigabytes,
    SUM(sfr_datafile.filesize) / (1024 ^ 4) AS sizeinterabytes
FROM sfr_datafile
JOIN authenticateduser
    ON authenticateduser.id = sfr_datafile.creator_id
GROUP BY authenticateduser.email;
```

## approach 3: approach 2 improved

slightly more straightforward, as it doesn't require you to know your collection's
ID - just the alias that Dataverse uses. 

```sql 
WITH sfr_datafile AS (
    SELECT datafile.*,
        dvobject.creator_id,
        dvobject.owner_id 
    FROM datafile 
    JOIN dvobject on dvobject.id = datafile.id
    JOIN dataset on dataset.id = dvobject.owner_id
    JOIN dvobject AS dvds on dvds.id = dvobject.owner_id
    JOIN dataverse on dataverse.id = dvds.owner_id
    WHERE dataverse.alias = 'oskidata'
)
SELECT authenticateduser.email,
    SUM(sfr_datafile.filesize) AS sizeinbytes,
    SUM(sfr_datafile.filesize) / (1024 ^ 3) AS sizeingigabytes,
    SUM(sfr_datafile.filesize) / (1024 ^ 4) AS sizeinterabytes
FROM sfr_datafile
JOIN authenticateduser
    ON authenticateduser.id = sfr_datafile.creator_id
GROUP BY authenticateduser.email;
```