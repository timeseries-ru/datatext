# DataText

***Under development! Not ready yet.***

![Screenshot](screenshots/screenshot.png) 

## TO BE DONE
- [X] Fix of multiple rendering
- [ ] CSV load with alasql
- [ ] Previews
- [ ] Toolbar: New/Save/Load/Export (with images) 

## How to make a query and visualize it

Example with MySQL:
```
mysql localhost abalone_database root password;
select diameter, viscera, sex from data;
scatter;
```

Example with Sqlite:
```
sqlite data/abalone.db;
select diameter, viscera, shucked, sex from data;
scatter viscera, diameter, sex;
scatter diameter, shucked;
```

0. Lines should be separated by `;`,
1. First line should be connection information (examples are self-explanatory), only sqlite and mysql are supported for now,
2. Second line should `select` query with at least two columns to fetch. Or three columns, if you need to group your data on chart,
3. Third and further lines are visualization types: `scatter`, `lines` or `bars`. When you don't provide fields, it takes first column as X axis, second as Y axis, and third as grouping (if present). You may specify other order, for example `scatter column_2, column_1` or `lines column_3, column_2, column_1`, where `column_*` - is the name of the fetched fields.


## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```
