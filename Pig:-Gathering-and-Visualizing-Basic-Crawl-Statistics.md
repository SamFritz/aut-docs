Let's start off by first finding out what's in the crawl. This Pig script counts the number of pages in each crawl:

```
register 'target/warcbase-0.1.0-SNAPSHOT-fatjar.jar';

DEFINE ArcLoader org.warcbase.pig.ArcLoader();
DEFINE WarcLoader org.warcbase.pig.WarcLoader();

arc = load '/collections/webarchives/CanadianPoliticalParties/arc/' using ArcLoader as
  (url: chararray, date: chararray, mime: chararray, content: bytearray);
warc = load '/collections/webarchives/CanadianPoliticalParties/warc/' using WarcLoader as
  (url: chararray, date: chararray, mime: chararray, content: bytearray);
raw = union arc, warc;

a = filter raw by mime == 'text/html' and date is not null;
b = foreach a generate SUBSTRING(date, 0, 6) as date, url;
c = group b by date;
d = foreach c generate group, COUNT(b);
e = order d by $0;

dump e;
```

This script counts the number of links for each crawl:

```
register 'target/warcbase-0.1.0-SNAPSHOT-fatjar.jar';

DEFINE ArcLoader org.warcbase.pig.ArcLoader();
DEFINE WarcLoader org.warcbase.pig.WarcLoader();
DEFINE ExtractLinks org.warcbase.pig.piggybank.ExtractLinks();

arc = load '/collections/webarchives/CanadianPoliticalParties/arc/' using ArcLoader as
  (url: chararray, date: chararray, mime: chararray, content: bytearray);
warc = load '/collections/webarchives/CanadianPoliticalParties/warc/' using WarcLoader as
  (url: chararray, date: chararray, mime: chararray, content: bytearray);
raw = union arc, warc;

a = filter raw by mime == 'text/html' and date is not null;
b = foreach a generate SUBSTRING(date, 0, 6) as date, url, FLATTEN(ExtractLinks((chararray) content, url));
c = group b by $0;
d = foreach c generate group, COUNT(b);
e = order d bt $0, $1;

store e into 'cpp.linkcounts/';
```

This script counts the number of pages within each top-level domain, grouped by crawl date:

```
register 'target/warcbase-0.1.0-SNAPSHOT-fatjar.jar';

DEFINE ArcLoader org.warcbase.pig.ArcLoader();
DEFINE WarcLoader org.warcbase.pig.WarcLoader();
DEFINE ExtractTopLevelDomain org.warcbase.pig.piggybank.ExtractTopLevelDomain();

arc = load '/collections/webarchives/CanadianPoliticalParties/arc/' using ArcLoader as
  (url: chararray, date: chararray, mime: chararray, content: bytearray);
warc = load '/collections/webarchives/CanadianPoliticalParties/warc/' using WarcLoader as
  (url: chararray, date: chararray, mime: chararray, content: bytearray);
raw = union arc, warc;

a = filter raw by mime == 'text/html' and date is not null;
b = foreach a generate SUBSTRING(date, 0, 6) as date, REPLACE(ExtractTopLevelDomain(url), '^\\s*www\\.', '') as url;
c = group b by (date, url);
d = foreach c generate FLATTEN(group), COUNT(b) as count;
e = filter d by count > 10;
f = order e by $0, $1;

store f into 'cpp.sitecounts/';
```

You may want to visualize these results. Using [these files](https://github.com/lintool/warcbase/tree/master/vis/crawl-sites), you can generate a bar graph. Replace `raw.txt` with your own data from the above script (i.e. `part-m-00000` in `cpp.sitecounts/`). 

Run:

```
python process.py > data.csv
```

And then 

```
python -m SimpleHTTPServer 1234
```

Navigate to <http://localhost:1234> in your web browser and you will see the following interactive diagram.

![](https://github.com/ianmilligan1/WAHR/blob/master/walkthroughs/images/crawl-viz.png)