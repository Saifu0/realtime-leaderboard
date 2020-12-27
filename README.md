# Realtime scaled leaderboard system design

### Assumptions

1. Score is not getting updated by user. Some third party is doing that for us.
2. Schedular update data automatically in every time 't'. 
3. Lower the score, lower the rank ( e.g score=[5,1,3], rank=[3rd, 1st, 2nd]. 
4. The storage and querying is done through ElastiCache for Redis instead of through relational databases.
5. Assuming there are N pools and every pools have some entry lets say 500,000. where, (1<=N<=100,000)
6. Score can be same of different entries but entry ID must be unqiue.
7. After creating new entry, it is being inserted into sorted set. But pools are not inserted into sorted set.
8. Every pool is independent to each other i.e "Score Updator" update score of an entry in a pool independently. 
9. Assuming, one entry takes 1 kilobyte of memory.
10. Modern PC can take upto 10^8 operations per second.

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

### Pros of my design
1. Frees up database resources for other types of requests.
2. Redis offers one highly efficient and scalable solution.
3. Due to sorted set, storage and querying is highly optimised. 
4. Product APIs help in reducing load on database by making fewer requests.
5. Local cache and distributed cache also helps in reducing load on database.
6. Due to fast querying and storage, it can take a large number of requests in optimized way.

### Cons of my design
1. Too much traffic may break the design.
2. We cannot handle too many pools of max limit entries (which is 500,000 in our case).

### When my design will break
Since every Pools are stored in a Database independently. It will take O(N) times to create only pools. And every pools have atmost 500,000 entries, lets denote
it as 'E'. 

now, each entry take O(logE) time to update or create.
so for one pool, O(E*logE) time

for N pools, O(N*E*logE) time.

If N= 1millions pools is being created and E=500,000 entries is also created at the same time. Then it will take approx 96000 seconds. Which is too much.
Once, Pools are created. It will take approx 0.09 second to update an entries. 

So, 1 millions pools and 500,000 entries at the same time will take too much time and too much memory.

### Overkill or not?
When pools are 10 with 1000 entries in each, It would be better or preferable to use traditional relational database method.
Typically, these include:
    Create a table.
    Insert or update scores as they change.
    Query the table to retrieve the ranking by score in ascending order.
