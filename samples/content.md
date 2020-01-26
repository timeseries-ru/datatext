# Some heading

Some text...

```
csv data/abalone.csv;
select sum(cast (diameter as float)) as diameter, sex from csv where shucked > 0.5 group by sex;
bars sex, diameter;
```