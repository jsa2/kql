

# connect to host
```
ssh user@nstr.dewi.red

cd /user/home/azapp

node app.js

```

then close app with ctrl-c

then 
`` cat tkn.txt `` 


# query
```
let moreLogs = adx('https://nstr.dewi.red:8443/Samples').StormEvents | take 1;
Usage
| count;
```
