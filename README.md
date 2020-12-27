# Realtime scaled leaderboard system design

### Assumptions

1. Score is not getting updated by user. Some third party is doing that for us.
2. Schedular update data automatically in every time 't'. 
3. Lower the score, lower the rank ( e.g score=[5,1,3], rank=[3rd, 1st, 2nd]. 
4. The storage and querying is done through ElastiCache for Redis instead of through relational databases.
5. Assuming there are N pools and every pools have some entry lets say 500,000. where, (1<=N<=100,000)
6. Score can be same of different entries but entry ID must be unqiue.

### Flowchart
![alt text](https://github.com/Saifu0/realtime-leaderboard/blob/main/flowchart.png?raw=true)

Here, score is getting updated by some third party software/updater in database in a specific pool. Pools are serving as
core APIs which is responsible for storing all data regarding users, score, assets, long/short, weightage, etc of all entries in a pool. 
Further, product APIs are extended from core APIs to reduce the load on database. We can also add local cache for storing frequently requested 
data and distributed cache for less frequently requested data. But we have to invalidate cache data after some interval of time.

Varnish is introduced between product APIs and core APIs to handle heavy request of same data coming from product APIs and collecting them 
then sending only one request of same type to core APIs and then deliver data to all collected APIs of same type. This will reduce load on
database too. 

Binder is something which binds core APIs and product APIs to work together.

Since, we want our leaderboard to be updated in real-time, we have to introduce a schedular which call APIs needed to render on learderboard screen
in every time 't'. Once time 't' is passed and we get our data, then we send our data into Delta which basically have information about previous
data. What this does is, it checks whether previous data ( data already Redis have) and new data came after time 't' have some common data, if yes
then it will not send same data again to Redis instead send only unique data or to be updated data. 

Now, our Redis store data in sorted set order which mean we can prioritized fields according to requirement. In our case, we want minimum score 
come first on leaderboard which can be done by setting set "Score" field in ascending order. It takes O(logN) time to insert, find & delete an entry where N is number of entries it already have.

In last, Redis send data to websockets and can be rendered on leaderboard screen. 


