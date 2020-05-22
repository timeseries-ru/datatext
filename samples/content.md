# Some heading

Some text...

---

Other text.


```
csv data/abalone.csv;
select avg(cast (diameter as float)) as mean_diameter, sex from csv where shucked > 0.5 group by sex;
bars sex, mean_diameter;
```
