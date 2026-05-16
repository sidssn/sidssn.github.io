---
title: "How Glue and Athena Keep Schooling Me"
date: 2026-05-06
description: "A quick field note on the mistakes I keep making with Glue crawlers, Athena partitions, and S3 log data."
tags: ["aws", "glue", "athena", "s3", "lessons-learned"]
---

I have now reached that stage of AWS life where Glue and Athena do not scare me, but they absolutely still humble me.

Every few months I do the same thing: I feel confident, make one "small" assumption, lose half a day, and rediscover a lesson I should already know.

This post is mostly a note to future me.

## Where I keep getting it wrong

The classic thought process goes like this:

"Data is in S3. Crawler will pick it up. Athena will query it. Easy."

Then reality hits. Folder names decide whether partitions work or explode, crawler mode changes behavior more than you expect, one wrong schema type gives weird query output, and one stray JSON file in a Parquet path can ruin the whole thing.

## Lesson 1: S3 path shape is everything

I kept hoping Glue would magically infer intent from folder names like this:

cost-usage-reports/team-export/data/BILLING_PERIOD=YYYY-MM/

What Glue did was logical, just not what I wanted. It treated each account folder as its own table root, produced multiple tables, and made onboarding new accounts annoying.

What finally worked was being strict:

cost-usage-reports/accountId=<account>/BILLING_PERIOD=YYYY-MM/

Once I gave in to key=value folders everywhere, the system became predictable.

Although this [documentation](https://docs.aws.amazon.com/glue/latest/dg/add-partitions.html) states that the crawler can infer partitions from predictable folder structures, in reality, it is very particular about the folder naming convention. If it has predictable folder names like `myproject-<account_id>-cost-report/BILLING_PERIOD=YYYY-MM/`, it cannot infer partitions correctly. It treats each account folder as a separate table root, resulting in multiple tables instead of one with partitions.

## Lesson 2: Crawler target type is a big deal

I used to ignore this and just click through setup. Bad idea. In practice, S3 target means the crawler can infer and update table shape, while catalog target means I define the schema and the crawler focuses on partitions.

For BCM exports, catalog target was way better for me. I wanted schema control, not surprise type changes.

Also, one config setting saved me: AddOrUpdateBehavior = InheritFromTable.

Without it, partitions can drift into slightly different schemas and Athena starts acting cursed.


## Lesson 3: When nothing works, build your own crawler

There comes a point where the native Glue crawler just refuses to cooperate. Wrong partition inference, schema drift you cannot override, edge case in the source format, or some combination of all three. I hit this wall.

The escape hatch: a custom Lambda function that registers partitions directly via the Glue catalog API.

The idea is simple. Lambda reads the S3 prefix, derives the partition keys from the folder structure, and calls `glue:BatchCreatePartition` (or `create_partition`) to register them manually. No crawler involved at all.

```python
import boto3

glue = boto3.client("glue")

def register_partition(database, table, partition_values, s3_location):
    glue.create_partition(
        DatabaseName=database,
        TableName=table,
        PartitionInput={
            "Values": partition_values,
            "StorageDescriptor": {
                # copy from existing table, override Location
                "Location": s3_location,
                "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
                "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
                "SerdeInfo": {"SerializationLibrary": "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe"},
            },
        },
    )
```

You can trigger this Lambda on S3 event notifications (new prefix created), on a schedule, or manually. The table schema stays exactly as you defined it, partitions appear immediately, and Athena picks them up on the next query.

The gotcha here is that you own the logic now. You have to handle duplicates (`AlreadyExistsException`), decide what to do when S3 structure changes, and make sure the `StorageDescriptor` you pass matches the table definition closely enough that Athena does not get confused.

But honestly, for awkward data shapes or tightly controlled schemas, this beats fighting the crawler every time.

## What I tell myself now

Glue and Athena are not hard because they are mysterious.
They are hard because they are strict.

If the fundamentals are right, life is decent: Hive-style folder layout, stable schema defined up front, crawler focused on partitions, and clean file format boundaries.

If one of these slips, I get schooled again.

## Future me note

Before debugging anything fancy, I now force myself to check the basics first: S3 path structure, partition key names and casing, crawler target mode, mixed file types in data paths, and IAM for table and partition updates.

Glue and Athena did not become easier.
I just got less stubborn about their rules.
