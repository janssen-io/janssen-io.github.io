---
layout: post
title: "TransactionQL: Parsing"
excerpt_separator: <!--more-->
category: Software Design
tags:
    - F#
    - Personal Project
    - TransactionQL
---

In the [first post]({% post_url 2020-11-08-TransactionQL-part-one %})
of this series, I gave quick overview of the requirements and
architecture of TransactionQL. In this post, I'll talk about the
csv and tql parsers.

![high level design](/assets/2020-11-01/tql-design.svg)
<!--more-->

## Parsing csv files
Parsing CSVs with F# is super easy. Using [Type Providers](https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/type-providers/)
we can use an example file to define the columns and the rest will
be taken care of.

Considering that the example file contains a header row, you can
use the column names directly in the code.

Example: transactions.csv
```csv
Date,From,To,Amount
2020/11/01,Bob,Alice,20.54
```

```fsharp
type BankTransactions = CsvProvider<"transactions.csv">
type Transaction = {
    Date: DateTime
    Amount: decimal
    Sender: string
    Receiver: string
}

BankTransactions
    .Load((new StreamReader("transactions-november.csv")))
    .Rows
|> Seq.map (fun row -> {
            Date = DateTime.Parse(row.Date)
            Amount = decimal.Parse(row.Amount)
            Sender = row.From
            Receiver = row.To
        }
    )
```

In TransactionQL the transactions are mapped to a `Map<string, string>`.
Generally, I'm not a fan of 'stringly' typed data, but in this
case I wanted to keep the tool as flexible as possible. Meaning,
the user defining the filters should have control over the data
that is being used to categorize the transactions.

The data is coming from the csv parser, so the filters and the
parsers are coupled together. They need to agree on which columns
can be used.

A benefit of this approach is that TransactionQL does not need to
changed and rebuilt if in the future more data is available in
the bank exports.

A disadvantage is that changing the parser might mean all filters
have to be changed too. But this is also the reason why TQL does
not depend on the parser directly, but instead accepts any assembly
that implements the interface. Part three will explain more about
the plugin architecture.

## Parsing tql files
The second file type that needs parsing is the file with the
filters. A filter consists out of three parts:

```
# Posting Title
Column = "Value"
Column > 100.00

posting {
  Account:Category  € (Expression)
  Account:Category  € 100.00
}
```

The filter starts with a title. This title will then be used in
the output as the title of the posting.

Following the title, are the conditions. All conditions have to
be met for a transaction to be categorized by this filter.
Alternatively, a condition can start with 'or' to categorize
the transaction if either the first or second condition is met.

Finally, the actual categorization and assignment of funds is
defined in the `post { ... }` block. Between parentheses values
can be added up and columns defined by the parser can be used.

### FParsec
As the structure of this file is more complex than a simple csv
file and it is not a default format, I decided to write my own
parser using [FParsec](http://www.quanttec.com/fparsec/).
I believe there are plenty of other blog posts and tutorials for
FParsec out there, so I won't dive into it here. In case you're
interested, you can find the final implementation on [GitHub](https://github.com/janssen-io/TransactionQL-fsharp/blob/master/TransactionQL.Parser/QLParser.fs)

## Filtering transactions
The parsed transactions are then fed into the [interpreter](https://github.com/janssen-io/TransactionQL-fsharp/blob/master/TransactionQL.Parser/QLInterpreter.fs)
together with the parsed filters. The result is a collection of
in-memory postings ready to be printed by the [output formatter](https://github.com/janssen-io/TransactionQL-fsharp/blob/master/TransactionQL.Console/Formatter.fs).

## Comments
_Wish to comment? You can add a comment to this post by sending me a [pull request](https://github.com/janssen-io/janssen-io.github.io#readme)._
