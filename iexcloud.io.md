## `iexcloud.io`

You'll need to create an account. Once you do that, you'll need to grab your API
key from <https://iexcloud.io/console/tokens>. Look for the "secret" token,
it'll resemble `sk_dcdxxxxxxxxxxxxxxxxxxxxxxxxxea31`.

Get 30 days of data:
```bash
curl -k 'https://cloud.iexapis.com/stable/stock/pvtl/quote?token=sk_XXX&filter=latestTime,latestVolume,latestPrice'
```
Get 30 days, but just the date & closing price:
```bash
curl -k 'https://cloud.iexapis.com/stable/stock/pvtl/chart?token=sk_XXX&filter=date,close' | jq -r .
```
Get 30 days, and use `jq` to create a CSV file to import into Google Sheets.
```bash
curl -k 'https://cloud.iexapis.com/stable/stock/pvtl/chart/max?token=sk_XXX&chartCloseOnly=true' > /tmp/pivotal.json
jq -r '.[] | .date + "," + (.close | tostring)' /tmp/pivotal.json > /tmp/pivotal.csv
```

You'll need a for-pay account to get insider activity. I signed up for a 1-month
for $19 & then immediately canceled after I had downloaded the insider activity.

```bash
curl -k 'https://cloud.iexapis.com/stable/stock/pvtl/insider-transactions?token=sk_XXX’ \
  > ~/aa/finances/pivotal_insiders.json
```

The dates are unreadable—the number of milliseconds in [UNIX Epoch
time](https://en.wikipedia.org/wiki/Unix_time) does not lend itself to easy
reading. `1550102400000` is cryptic, but `2019-02-14` is not. Let's convert to a
human-readable date.

```bash
jq -r '.[].effectiveDate |= (. / 1000 | strftime("%Y-%m-%d")) | .[]' ~/aa/finances/pivotal_insiders.json
```

Much better, but we're not done: we need to filter the transactions. We are only
interested in transactions whose `tranValue` is negative (which is a stock
sale).

```bash
jq -r 'map(select(.tranValue < 0))
    | map(select(.tranValue != null))
    | .[].effectiveDate |= (. / 1000 | strftime("%Y-%m-%d"))
    | .[]
    | .fullName + "," +
      .effectiveDate + "," +
      (.tranValue|tostring) + "," +
      (.tranShares|tostring)' \
    ~/aa/finances/pivotal_insiders.json \
    | sort -t , -k 1,2 \
    > /tmp/pivotal_insiders.csv
```

### Deciphering Insider Activity

Let's look at an example: On May 1, 2019, Pivotal president William Cook
exercised & sold 30,000 shares. Here's the [SEC filing](https://www.sec.gov/Archives/edgar/data/1574135/000112760219017152/xslF345X03/form4.xml).

And here are the corresponding JSON entries from IEX Cloud:

```json
{
  "fullName": "William Cook",
  "reportedTitle": "President",
  "effectiveDate": "2019-05-01",
  "tranShares": -30000,
  "tranPrice": null,
  "tranValue": null
}
{
  "fullName": "William Cook",
  "reportedTitle": "President",
  "effectiveDate": "2019-05-01",
  "tranShares": -30000,
  "tranPrice": 21.11,
  "tranValue": -633300
}
{
  "fullName": "William Cook",
  "reportedTitle": "President",
  "effectiveDate": "2019-05-01",
  "tranShares": 30000,
  "tranPrice": 5.06,
  "tranValue": 151800
}
```

Note:

- The first transaction, with null values for `tranPrice` and `tranValue`, is
  confusing—I'm ignoring it for the time being.
- The second transaction, with a `tranValue` of -633,300, is the sale.  This is
  the important transaction.
- The third transaction is the purchase (the exercise of securities).

Take-away: **I am only interested in negative `tranValue` entries**.

### SEC Filings

To decipher Insider Activity, browse the SEC filings, e.g.
<https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK=0001574135&type=&dateb=&owner=include&count=40>.

- Select `Ownership?` to `include` and then press search.

- Look in the Filings columns for the ones that say `4`; those are Insider
  Activity reports
