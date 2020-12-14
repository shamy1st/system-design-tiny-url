# TinyURL Design

Let's design a URL shortening service like TinyURL. This service will provide short aliases redirecting to long URLs. 

Similar services: bit.ly, goo.gl, qlink.me, etc.

Difficulty Level: Easy 

## What is TinyURL?

* URL shortening is used to create shorter aliases for long URLs.
* Users are redirected to the original URL when they hit these short links.
* Short links save a lot of space when displayed, printed, messaged, or tweeted.
* Additionally, users are less likely to mistype shorter URLs.
* The shortened URL is nearly one-third the size of the actual URL.
* URL shortening is used for optimizing links across devices, tracking individual links to analyze audience and campaign performance, and hiding affiliated original URLs.
* example:
      
      https://github.com/shamy1st/system-design-tiny-url/blob/main/README.md
      http://tinyurl.com/uxs6dlc

## 1. Requirements

### Functional Requirements

1. Given a URL, our service should generate a shorter and unique alias of it. This is called a short link. This link should be short enough to be easily copied and pasted into applications.
2. When users access a short link, our service should redirect them to the original link.
3. Users should optionally be able to pick a custom short link for their URL.
4. Links will expire after a standard default timespan. Users should be able to specify the expiration time.

### Non-functional Requirements

1. The system should be highly available. This is required because, if our service is down, all the URL redirections will start failing.
2. URL redirection should happen in real-time with minimal latency.
3. Shortened links should not be guessable (not predictable).

### Out of Scope Requirements

1. Analytics; e.g., how many times a redirection happened?
2. Our service should also be accessible through REST APIs by other services.

## 2. Design Considerations

## 3. Estimation

### Traffic Estimates

* Assuming, we will have **500M New URL** shortenings per month.
* Let’s assume a **100:1 ratio** between read (redirection) and write (new url).
* Then redirections per month: 100 * 500M = 50B
* **New URLs per second**: 500M / (30 * 24 * 60 * 60) =~ **200 URL/sec**
* Considering 100:1 read/write ratio, URLs **redirections per second**: 200 * 100 =~ **20K/sec**

### Storage Estimates

* Let’s assume we store every URL shortening request for **5 years**.
* Since we expect to have 500M new URLs per month, the **total number of objects** we expect to store will be: 500M * 12 * 5 = **30 Billion**.
* Let’s assume that **each stored object** will be approximately **500 bytes** (just a ballpark estimate – we will dig into it later).
* Then **total storage** will be: 30Billion * 500 Byte = **15TB**.

### Bandwidth estimates

* For write requests, since we expect **200 New URLs per second**, total incoming data for our service will be: 200 * 500 bytes = **100KB/sec**
* For read requests, since every second we expect **~20K URLs redirections**, total outgoing data for our service will be: 20K * 500 bytes = **10MB/sec**

### Memory estimates

* If we want to cache some of the hot URLs that are frequently accessed, how much memory will we need to store them?
* If we follow the 80-20 rule, meaning 20% of URLs generate 80% of traffic, we would like to cache these 20% hot URLs.
* Since we have 20K requests per second, we will be getting 1.7 billion requests per day: 20K * 24 * 60 * 60 =~ **1.7 Billion request/day**
* To cache 20% of these requests, we will need **170GB of memory**: 0.2 * 1.7 billion * 500 bytes = ~170GB.
* One thing to note here is that since there will be a lot of duplicate requests (of the same URL), therefore, our actual memory usage will be less than 170GB.

### High-level Estimates

* Assuming 500 million new URLs per month and 100:1 read:write ratio, following is the summary of the high level estimates for our service:

Metric              | Estimate
--------------------|---------
New URLs            | 200/s
URL redirections    | 20K/s
Incoming data       | 100KB/s
Outgoing data       | 10MB/s
Storage for 5 years | 15TB
Memory for cache    | 170GB

## 4. High-level Design

![](https://github.com/shamy1st/system-design-tiny-url/blob/main/hld.png)

## 5. System APIs

* We can have SOAP or REST APIs to expose the functionality of our service.

1. **createURL**(api_dev_key, original_url, custom_alias=None, user_name=None, expire_date=None)

      **Parameters**:
      * **api_dev_key** (string): The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.
      * **original_url** (string): Original URL to be shortened.
      * **user_name** (string): Optional user name to be used in the encoding.
      * **expire_date** (string): Optional expiration date for the shortened URL.

      **Return**: (string)
      A successful insertion returns the shortened URL; otherwise, it returns an error code.

2. **deleteURL**(api_dev_key, url_key)

      * Where “url_key” is a string representing the shortened URL to be retrieved. 
      * A successful deletion returns ‘URL Removed’.

## 6. Database Model

### Schema 
1. table for storing information about the URL mappings. 
2. table for the user’s data who created the short link.

![](https://github.com/shamy1st/system-design-tiny-url/blob/main/database-model.png)

### Which kind of database should we use?
* Since we anticipate storing billions of rows, and we don’t need to use relationships between objects
* NoSQL store like DynamoDB, Cassandra or Riak is a better choice.
* NoSQL would also be easier to scale.

## 7. Low-level Design

* The problem we are solving here is, how to generate a short and unique key for a given URL.

### Solution 01: Encoding Actual URL

* Since we expect to have 500M new URLs per month, then we need **30 billion per 5 years**
* Using base64 encoding, a **6 letters** long key would result in 64^6 =~ **68.7 billion possible strings**
* In base64 each **6 bit encoded to 1 character**, because 64 = 2^6
* If we use the MD5 algorithm as our hash function, it’ll produce a **128-bit** hash value.
* 128 bit / 6 bit **> 21 character**
* Now we only have space for 6 characters per short key.
* We can take the **first 6 letters for the key**.
* This could result in **key duplication**.
* To resolve that, we can choose some other characters out of the encoding string or swap some characters.
* **Problems for this solution**
      1. If multiple users enter the same URL, they can get the same shortened URL, which is not acceptable.
      2. What if parts of the URL are URL-encoded? (identical except for the URL encoding)

            http://www.educative.io/distributed.php?id=design
            http://www.educative.io/distributed.php%3Fid%3Ddesign 

### Solution 02: Generating Keys Offline

* We can have a standalone Key Generation Service (KGS) that generates random six-letter strings beforehand and stores them in a database (let’s call it key-DB).
* Whenever we want to shorten a URL, we will just take one of the already-generated keys and use it.
* This approach will make things quite simple and fast.
* Not only are we not encoding the URL, but we won’t have to worry about duplications or collisions.
* KGS will make sure all the keys inserted into key-DB are unique.
* What would be the key-DB size? 6 (characters per key) * 68.7B (unique keys) = **412 GB**

![](https://github.com/shamy1st/system-design-tiny-url/blob/main/lld.png)

## 8. Bottlenecks

### Data Partitioning and Replication

* To scale out our DB, we need to partition it so that it can store information about billions of URLs.
* We need to come up with a partitioning scheme that would divide and store our data into different DB servers.

1. **Range Based Partitioning**

      * We can store URLs in separate partitions based on the first letter of the hash key.
      * Hence we save all the URLs starting with letter ‘A’ (and ‘a’) in one partition, save those that start with letter ‘B’ in another partition and so on.
      * We can even combine certain less frequently occurring letters into one database partition.
      * The main problem with this approach is that it can lead to unbalanced DB servers.

2. **Hash-Based Partitioning**

      * We take a hash of the object we are storing.
      * We then calculate which partition to use based upon the hash.
      * In our case, we can take the hash of the ‘key’ or the short link to determine the partition in which we store the data object.
      * Our hashing function will randomly distribute URLs into different partitions (e.g., our hashing function can always map any ‘key’ to a number between [1…256]), and this number would represent the partition in which we store our object.
      * This approach can still lead to overloaded partitions, which can be solved by using Consistent Hashing.

### Cache

* We can cache URLs that are frequently accessed.
* We can use some off-the-shelf solution like Memcached, which can store full URLs with their respective hashes.
* The application servers, before hitting backend storage, can quickly check if the cache has the desired URL.
* **How much cache memory should we have?**
      * We can start with 20% of daily traffic and, based on clients’ usage pattern, we can adjust how many cache servers we need.
      * As estimated above, we need 170GB memory to cache 20% of daily traffic.
      * Since a modern-day server can have 256GB memory, we can easily fit all the cache into one machine.
      * Alternatively, we can use a couple of smaller servers to store all these hot URLs.
* **Which cache eviction policy would best fit our needs?**
      * When the cache is full, and we want to replace a link with a newer/hotter URL, how would we choose?
      * Least Recently Used (LRU) can be a reasonable policy for our system.
      * Under this policy, we discard the least recently used URL first.
      * We can use a Linked Hash Map or a similar data structure to store our URLs and Hashes, which will also keep track of the URLs that have been accessed recently.
      * To further increase the efficiency, we can replicate our caching servers to distribute the load between them.
* **How can each cache replica be updated?**
      * Whenever there is a cache miss, our servers would be hitting a backend database.
      * Whenever this happens, we can update the cache and pass the new entry to all the cache replicas.
      * Each replica can update its cache by adding the new entry.
      * If a replica already has that entry, it can simply ignore it.

### Load Balancer (LB)

* We can add a Load balancing layer at three places in our system:
      1. Between Clients and Application servers
      2. Between Application Servers and database servers
      3. Between Application Servers and Cache servers
* Initially, we could use a simple Round Robin approach that distributes incoming requests equally among backend servers.
* This LB is simple to implement and does not introduce any overhead.
* Another benefit of this approach is that if a server is dead, LB will take it out of the rotation and will stop sending any traffic to it.
* A problem with Round Robin LB is that we don’t take the server load into consideration.
* If a server is overloaded or slow, the LB will not stop sending new requests to that server.
* To handle this, a more intelligent LB solution can be placed that periodically queries the backend server about its load and adjusts traffic based on that.

### Purging or DB cleanup

* Should entries stick around forever or should they be purged? If a user-specified expiration time is reached, what should happen to the link?
* If we chose to actively search for expired links to remove them, it would put a lot of pressure on our database.
* Instead, we can slowly remove expired links and do a lazy cleanup.
* Our service will make sure that only expired links will be deleted, although some expired links can live longer but will never be returned to users.
* A separate Cleanup service can run periodically to remove expired links from our storage and cache. This service should be very lightweight and can be scheduled to run only when the user traffic is expected to be low.
* We can have a default expiration time for each link (e.g., two years).
* After removing an expired link, we can put the key back in the key-DB to be reused.
* Should we remove links that haven’t been visited in some length of time, say six months? This could be tricky. Since storage is getting cheap, we can decide to keep links forever.

## 9. Security and Permissions

* Can users create private URLs or allow a particular set of users to access a URL?
* We can store the permission level (public/private) with each URL in the database.
* We can also create a separate table to store UserIDs that have permission to see a specific URL.
* If a user does not have permission and tries to access a URL, we can send an error (HTTP 401) back.
* Given that we are storing our data in a NoSQL wide-column database like Cassandra, the key for the table storing permissions would be the ‘Hash’ (or the KGS generated ‘key’).
* The columns will store the UserIDs of those users that have the permission to see the URL.
