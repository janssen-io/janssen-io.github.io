---
layout: post
title: "TransactionQL: Introduction"
excerpt_separator: <!--more-->
category: Series Introduction
tags:
    - F#
    - Personal Project
    - TransactionQL
---

Roughly five or six years ago I started tracking the various categories I spent money on in Microsoft Excel.
While this was fine in the beginning, it soon became a tedious process. Checking the bank statement, then figuring out which transaction should be put in which category, it just took way too long. Besides many of these transactions were recurrent, so why not bring in some automation?

<!--more-->

At the time I was using Ruby for work and knew of its powers to create internal DSLs with ease. My first [solution](https://github.com/janssen-io/transaction-ql) seemed pretty nice, but it still left me with integrating it in Excel.

Because working with Excel from a programming language never really appealed to me, I kept inputting the data by hand. After a year or so, I grew tired of these too, so another solution had to be out there. I still didn't want to use web apps, my financial data should stay my data. So I went looking for an alternative and discovered the world of [plain text accounting](https://plaintextaccounting.org/).

## Plain Text Accounting
After installing [ledger-cli](https://ledger-cli.org/) and manually inputting my transactions again for a while, I found that it's a great piece of software to work with. The double-entry bookkeeping appealed to me, because it allowed to group transactions together and validate it. Ledger can help you ensure your bank account balance (in the books) equals the actual balance by using assertions. And the possibility to create sub-accounts that aggregate to the top level automatically is really great.

Now I was still manually inputting all the data into my ledger, but this time it was all just text. That's a much easier format to work with then Excel! Now during this time I also refound my interest in functional programming as I had recently discovered F#. And considering functional languages are great for writing parsers, why not give that shot!

## TransactionQL in F#
For this project, I had a few requirements:

1. Support (csv) exports from any bank
2. Support an arbitrary set of columns
3. Output the categorized transactions in ledger syntax
4. Output uncategorized transactions as comments
5. Ouput all transactions ordered by date (oldest to most recent)

To meet these criteria, the following design emerged:

![high level design](/assets/2020-11-01/tql-design.svg)

The program will start by parsing a csv, this module will be specific for each bank and interchangeable by making use of a plugin architecture.
Alongside the csv, the filters will be parsed ("tql" format). The parsed filters can then be used to filter the data from the csv and fed to an output module. This module will then output the filtered transactions as ledger postings.

In the next posts, we will dive into the implementation of the modules.

Read Part Two [here]({% post_url 2020-11-22-TransactionQL-part-two %})


## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
