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

### Encoding Actual URL

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

### Generating Keys Offline

* We can have a standalone Key Generation Service (KGS) that generates random six-letter strings beforehand and stores them in a database (let’s call it key-DB).
* Whenever we want to shorten a URL, we will just take one of the already-generated keys and use it.
* This approach will make things quite simple and fast.
* Not only are we not encoding the URL, but we won’t have to worry about duplications or collisions.
* KGS will make sure all the keys inserted into key-DB are unique.
* What would be the key-DB size? 6 (characters per key) * 68.7B (unique keys) = **412 GB**

## 8. Bottlenecks

