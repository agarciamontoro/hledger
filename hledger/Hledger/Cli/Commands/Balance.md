balance, bal, b\
Show accounts and their balances.

_FLAGS_

The balance command is hledger's most versatile command.
Note, despite the name, it is not always used for showing real-world account balances;
the more accounting-aware [balancesheet](#balancesheet)
and [incomestatement](#incomestatement) may be more convenient for that.

By default, it displays all accounts,
and each account's change in balance during the entire period of the journal.
Balance changes are calculated by adding up the postings in each account.
You can limit the postings matched, by a [query](#queries), to see fewer accounts, changes over a different time period, changes from only cleared transactions, etc.

If you include an account's complete history of postings in the report,
the balance change is equivalent to the account's current ending balance.
For a real-world account, typically you won't have all transactions in the journal;
instead you'll have all transactions after a certain date, and an "opening balances"
transaction setting the correct starting balance on that date.
Then the balance command will show real-world account balances.
In some cases the -H/--historical flag is used to ensure this (more below).

The balance command can produce several styles of report:

### Classic balance report

This is the original balance report, as found in Ledger. It usually looks like this:

```shell
$ hledger balance
                 $-1  assets
                  $1    bank:saving
                 $-2    cash
                  $2  expenses
                  $1    food
                  $1    supplies
                 $-2  income
                 $-1    gifts
                 $-1    salary
                  $1  liabilities:debts
--------------------
                   0
```

By default, accounts are displayed hierarchically, with subaccounts indented below their parent.
At each level of the tree, accounts are sorted by [account code](/manual.html#declaring-accounts) if any, then by account name.
Or with `-S/--sort-amount`, by their balance amount.

"Boring" accounts, which contain a single interesting subaccount and 
no balance of their own, are elided into the following line for more compact output.
(Eg above, the "liabilities" account.) Use `--no-elide` to prevent this. 

Account balances are "inclusive" - they include the balances of any subaccounts.

Accounts which have zero balance (and no non-zero subaccounts) are
omitted. Use `-E/--empty` to show them.

A final total is displayed by default; use `-N/--no-total` to suppress it, eg:

```shell
$ hledger balance -p 2008/6 expenses --no-total
                  $2  expenses
                  $1    food
                  $1    supplies
```

### Customising the classic balance report

You can customise the layout of classic balance reports with `--format FMT`:

```shell
$ hledger balance --format "%20(account) %12(total)"
              assets          $-1
         bank:saving           $1
                cash          $-2
            expenses           $2
                food           $1
            supplies           $1
              income          $-2
               gifts          $-1
              salary          $-1
   liabilities:debts           $1
---------------------------------
                                0
```

The FMT format string (plus a newline) specifies the formatting
applied to each account/balance pair. It may contain any suitable
text, with data fields interpolated like so:

`%[MIN][.MAX](FIELDNAME)`

- MIN pads with spaces to at least this width (optional)
- MAX truncates at this width (optional)
- FIELDNAME must be enclosed in parentheses, and can be one of:

    - `depth_spacer` - a number of spaces equal to the account's depth, or if MIN is specified, MIN * depth spaces.
    - `account`      - the account's name
    - `total`        - the account's balance/posted total, right justified

Also, FMT can begin with an optional prefix to control how
multi-commodity amounts are rendered:

- `%_` - render on multiple lines, bottom-aligned (the default)
- `%^` - render on multiple lines, top-aligned
- `%,` - render on one line, comma-separated

There are some quirks. Eg in one-line mode, `%(depth_spacer)` has no
effect, instead `%(account)` has indentation built in.
<!-- XXX retest:
Consistent column widths are not well enforced, causing ragged edges unless you set suitable widths.
Beware of specifying a maximum width; it will clip account names and amounts that are too wide, with no visible indication.
-->
Experimentation may be needed to get pleasing results.

Some example formats:

- `%(total)`         - the account's total
- `%-20.20(account)` - the account's name, left justified, padded to 20 characters and clipped at 20 characters
- `%,%-50(account)  %25(total)` - account name padded to 50 characters, total padded to 20 characters, with multiple commodities rendered on one line
- `%20(total)  %2(depth_spacer)%-(account)` - the default format for the single-column balance report

### Colour support

The balance command shows negative amounts in red, if:

- the `TERM` environment variable is not set to `dumb`
- the output is not being redirected or piped anywhere

### Flat mode

To see a flat list instead of the default hierarchical display, use `--flat`.
In this mode, accounts (unless depth-clipped) show their full names and "exclusive" balance, excluding any subaccount balances.
In this mode, you can also use `--drop N` to omit the first few account name components.

```shell
$ hledger balance -p 2008/6 expenses -N --flat --drop 1
                  $1  food
                  $1  supplies
```

### Depth limited balance reports

With `--depth N` or `depth:N` or just `-N`, balance reports show accounts only to the specified numeric depth.
This is very useful to summarise a complex set of accounts and get an overview.
```shell
$ hledger balance -N -1
                 $-1  assets
                  $2  expenses
                 $-2  income
                  $1  liabilities
```

Flat-mode balance reports, which normally show exclusive balances, show inclusive balances at the depth limit.

<!-- $ for y in 2006 2007 2008 2009 2010; do echo; echo $y; hledger -f $y.journal balance ^expenses --depth 2; done -->

### Multicolumn balance report

Multicolumn or tabular balance reports are a very useful hledger feature,
and usually the preferred style. They share many of the above features, 
but they show the report as a table, with columns representing time periods.
This mode is activated by providing a [reporting interval](#reporting-interval).

There are three types of multicolumn balance report, showing different information:

1. By default: each column shows the sum of postings in that period,
ie the account's change of balance in that period. This is useful eg
for a monthly income statement:
<!--
multicolumn income statement: 

   $ hledger balance ^income ^expense -p 'monthly this year' --depth 3

or cashflow statement:

   $ hledger balance ^assets ^liabilities 'not:(receivable|payable)' -p 'weekly this month'
-->

    ```shell
    $ hledger balance --quarterly income expenses -E
    Balance changes in 2008:

                       ||  2008q1  2008q2  2008q3  2008q4 
    ===================++=================================
     expenses:food     ||       0      $1       0       0 
     expenses:supplies ||       0      $1       0       0 
     income:gifts      ||       0     $-1       0       0 
     income:salary     ||     $-1       0       0       0 
    -------------------++---------------------------------
                       ||     $-1      $1       0       0 

    ```

2. With `--cumulative`: each column shows the ending balance for that
period, accumulating the changes across periods, starting from 0 at
the report start date:

    ```shell
    $ hledger balance --quarterly income expenses -E --cumulative
    Ending balances (cumulative) in 2008:

                       ||  2008/03/31  2008/06/30  2008/09/30  2008/12/31 
    ===================++=================================================
     expenses:food     ||           0          $1          $1          $1 
     expenses:supplies ||           0          $1          $1          $1 
     income:gifts      ||           0         $-1         $-1         $-1 
     income:salary     ||         $-1         $-1         $-1         $-1 
    -------------------++-------------------------------------------------
                       ||         $-1           0           0           0 

    ```

3. With `--historical/-H`: each column shows the actual historical
ending balance for that period, accumulating the changes across
periods, starting from the actual balance at the report start date.
This is useful eg for a multi-period balance sheet, and when
you are showing only the data after a certain start date:

    ```shell
    $ hledger balance ^assets ^liabilities --quarterly --historical --begin 2008/4/1
    Ending balances (historical) in 2008/04/01-2008/12/31:

                          ||  2008/06/30  2008/09/30  2008/12/31 
    ======================++=====================================
     assets:bank:checking ||          $1          $1           0 
     assets:bank:saving   ||          $1          $1          $1 
     assets:cash          ||         $-2         $-2         $-2 
     liabilities:debts    ||           0           0          $1 
    ----------------------++-------------------------------------
                          ||           0           0           0 

    ```

Note that `--cumulative` or `--historical/-H` disable `--row-total/-T`, 
since summing end balances generally does not make sense.

Multicolumn balance reports display accounts in flat mode by default;
to see the hierarchy, use `--tree`.

With a reporting interval (like `--quarterly` above), the report
start/end dates will be adjusted if necessary so that they encompass
the displayed report periods. This is so that the first and last
periods will be "full" and comparable to the others.

The `-E/--empty` flag does two things in multicolumn balance reports:
first, the report will show all columns within the specified report
period (without -E, leading and trailing columns with all zeroes are
not shown). Second, all accounts which existed at the report start
date will be considered, not just the ones with activity during the
report period (use -E to include low-activity accounts which would
otherwise would be omitted).

The `-T/--row-total` flag adds an additional column showing the total
for each row.

The `-A/--average` flag adds a column showing the average value in
each row.

Here's an example of all three:

```shell
$ hledger balance -Q income expenses --tree -ETA
Balance changes in 2008:

            ||  2008q1  2008q2  2008q3  2008q4    Total  Average 
============++===================================================
 expenses   ||       0      $2       0       0       $2       $1 
   food     ||       0      $1       0       0       $1        0 
   supplies ||       0      $1       0       0       $1        0 
 income     ||     $-1     $-1       0       0      $-2      $-1 
   gifts    ||       0     $-1       0       0      $-1        0 
   salary   ||     $-1       0       0       0      $-1        0 
------------++---------------------------------------------------
            ||     $-1      $1       0       0        0        0 

# Average is rounded to the dollar here since all journal amounts are

```

Limitations:

In multicolumn reports the [`-V/--value` flag](#market-value) uses the
market price on the report end date, for all columns (not the price on
each column's end date).

Eliding of boring parent accounts in tree mode, as in the classic
balance report, is not yet supported in multicolumn reports.

### Budget report

With `--budget`, extra columns are displayed showing budget goals for each account and period, if any.
Budget goals are defined by [periodic transactions](journal.html#periodic-transactions).
This is very useful for comparing planned and actual income, expenses, time usage, etc.
--budget is most often combined with a [report interval](hledger.html#report-intervals).

For example, you can take average monthly expenses in the common expense categories to construct a minimal monthly budget:
```journal
;; Budget
~ monthly
  income  $2000
  expenses:food    $400
  expenses:bus     $50
  expenses:movies  $30
  assets:bank:checking

;; Two months worth of expenses
2017-11-01
  income  $1950
  expenses:food    $396
  expenses:bus     $49
  expenses:movies  $30
  expenses:supplies  $20
  assets:bank:checking

2017-12-01
  income  $2100
  expenses:food    $412
  expenses:bus     $53
  expenses:gifts   $100
  assets:bank:checking
```

You can now see a monthly budget report:
```shell
$ hledger balance -M --budget
Budget performance in 2017/11/01-2017/12/31:

                      ||                      Nov                       Dec 
======================++====================================================
 assets               || $-2445 [  99% of $-2480]  $-2665 [ 107% of $-2480] 
 assets:bank          || $-2445 [  99% of $-2480]  $-2665 [ 107% of $-2480] 
 assets:bank:checking || $-2445 [  99% of $-2480]  $-2665 [ 107% of $-2480] 
 expenses             ||   $495 [ 103% of   $480]    $565 [ 118% of   $480] 
 expenses:bus         ||    $49 [  98% of    $50]     $53 [ 106% of    $50] 
 expenses:food        ||   $396 [  99% of   $400]    $412 [ 103% of   $400] 
 expenses:movies      ||    $30 [ 100% of    $30]       0 [   0% of    $30] 
 income               ||  $1950 [  98% of  $2000]   $2100 [ 105% of  $2000] 
----------------------++----------------------------------------------------
                      ||      0 [              0]       0 [              0] 
```

Note this is different from a normal balance report in several ways:

- Only accounts with budget goals during the report period are shown, by default.

- In each column, in square brackets after the actual amount, budgeted amounts are shown,
  along with the percentage of budget used.

- All parent accounts are always shown, even in flat mode. 
  Eg assets, assets:bank, and expenses above.

- Amounts always include all subaccounts, budgeted or unbudgeted, even in flat mode.

This means that the numbers displayed will not always add up!
Eg above, the `expenses`  actual amount includes the gifts and supplies transactions,
but the `expenses:gifts` and `expenses:supplies` accounts are not
shown, as they have no budget amounts declared.

This can be confusing. When you need to make things clearer, use the `-E/--empty` flag, 
which will reveal all accounts including unbudgeted ones, giving the full picture. Eg:

```shell
$ hledger balance -M --budget --empty
Budget performance in 2017/11/01-2017/12/31:

                      ||                      Nov                       Dec 
======================++====================================================
 assets               || $-2445 [  99% of $-2480]  $-2665 [ 107% of $-2480] 
 assets:bank          || $-2445 [  99% of $-2480]  $-2665 [ 107% of $-2480] 
 assets:bank:checking || $-2445 [  99% of $-2480]  $-2665 [ 107% of $-2480] 
 expenses             ||   $495 [ 103% of   $480]    $565 [ 118% of   $480] 
 expenses:bus         ||    $49 [  98% of    $50]     $53 [ 106% of    $50] 
 expenses:food        ||   $396 [  99% of   $400]    $412 [ 103% of   $400] 
 expenses:gifts       ||      0                      $100                   
 expenses:movies      ||    $30 [ 100% of    $30]       0 [   0% of    $30] 
 expenses:supplies    ||    $20                         0                   
 income               ||  $1950 [  98% of  $2000]   $2100 [ 105% of  $2000] 
----------------------++----------------------------------------------------
                      ||      0 [              0]       0 [              0] 
```


You can roll over unspent budgets to next period with `--cumulative`:
```shell
$ hledger balance -M --budget --cumulative
Budget performance in 2017/11/01-2017/12/31:

                      ||                      Nov                       Dec 
======================++====================================================
 assets               || $-2445 [  99% of $-2480]  $-5110 [ 103% of $-4960] 
 assets:bank          || $-2445 [  99% of $-2480]  $-5110 [ 103% of $-4960] 
 assets:bank:checking || $-2445 [  99% of $-2480]  $-5110 [ 103% of $-4960] 
 expenses             ||   $495 [ 103% of   $480]   $1060 [ 110% of   $960] 
 expenses:bus         ||    $49 [  98% of    $50]    $102 [ 102% of   $100] 
 expenses:food        ||   $396 [  99% of   $400]    $808 [ 101% of   $800] 
 expenses:movies      ||    $30 [ 100% of    $30]     $30 [  50% of    $60] 
 income               ||  $1950 [  98% of  $2000]   $4050 [ 101% of  $4000] 
----------------------++----------------------------------------------------
                      ||      0 [              0]       0 [              0] 
```

For more examples, see [Budgeting and Forecasting](https://github.com/simonmichael/hledger/wiki/Budgeting and forecasting).

#### Nested budgets

You can add budgets to any account in your account hierarchy. If you have budgets on both parent account and some of its children, then budget(s)
of the child account(s) would be added to the budget of their parent, much like account balances behave.

In the most simple case this means that once you add a budget to any account, all its parents would have budget as well. 

To illustrate this, consider the following budget:
```
~ monthly from 2019/01
    expenses:personal             $1,000.00
    expenses:personal:electronics    $100.00
    liabilities
```

With this, monthly budget for electronics is defined to be $100 and budget for personal expenses is an additional $1000, which implicity means
that budget for both `expenses:personal` and `expenses` is $1100.

Transactions in `expenses:personal:electronics` will be counted both towards its $100 budget and $1100 of `expenses:personal` , and transactions in any other subaccount of `expenses:personal` would be
counted towards only towards the budget of `expenses:personal`.

For example, let's consider these transactions:
```journal
~ monthly from 2019/01
    expenses:personal             $1,000.00
    expenses:personal:electronics    $100.00
    liabilities

2019/01/01 Google home hub
    expenses:personal:electronics          $90.00
    liabilities                           $-90.00

2019/01/02 Phone screen protector
    expenses:personal:electronics:upgrades          $10.00
    liabilities

2019/01/02 Weekly train ticket
    expenses:personal:train tickets       $153.00
    liabilities

2019/01/03 Flowers
    expenses:personal          $30.00
    liabilities
```

As you can see, we have transactions in `expenses:personal:electronics:upgrades` and `expenses:personal:train tickets`, and since both of these accounts are without explicitly defined budget,
these transactions would be counted towards budgets of `expenses:personal:electronics` and `expenses:personal` accordingly:

```shell
$ hledger balance --budget -M
Budget performance in 2019/01:

                               ||                           Jan 
===============================++===============================
 expenses                      ||  $283.00 [  26% of  $1100.00] 
 expenses:personal             ||  $283.00 [  26% of  $1100.00] 
 expenses:personal:electronics ||  $100.00 [ 100% of   $100.00] 
 liabilities                   || $-283.00 [  26% of $-1100.00] 
-------------------------------++-------------------------------
                               ||        0 [                 0] 
```

And with `--empty`, we can get a better picture of budget allocation and consumption:
```shell
$ hledger balance --budget -M --empty
Budget performance in 2019/01:

                                        ||                           Jan 
========================================++===============================
 expenses                               ||  $283.00 [  26% of  $1100.00] 
 expenses:personal                      ||  $283.00 [  26% of  $1100.00] 
 expenses:personal:electronics          ||  $100.00 [ 100% of   $100.00] 
 expenses:personal:electronics:upgrades ||   $10.00                      
 expenses:personal:train tickets        ||  $153.00                      
 liabilities                            || $-283.00 [  26% of $-1100.00] 
----------------------------------------++-------------------------------
                                        ||        0 [                 0] 
```

### Output format

The balance command supports [output destination](/manual.html#output-destination) and [output format](/manual.html#output-format) selection.

