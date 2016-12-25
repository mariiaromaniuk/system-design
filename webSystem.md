# Web systems
<!-- MarkdownTOC -->

- TinyURL
	- Scenario
		- Features
		- Design goals
		- Estimation
	- Service
		- shortURL insert\( longURL \)
			- Encode
				- Traditional hash function
				- Base10 / Base62
			- Implementation
		- longURL lookup\( shortURL \)
			- Implementation
	- Storage
		- SQL
			- Schema design
		- NoSQL
	- Scale

<!-- /MarkdownTOC -->



# TinyURL 
## Scenario 
### Features 
* Shortening: Take a url and return a much shorter url. 
	- Ex: http://www.interviewbit.com/courses/programming/topics/time-complexity/ => http://goo.gl/GUKA8w/
	- Gotcha: What if two people try to shorten the same URL?
* Redirection: Take a short url and redirect to the original url. 
	- Ex: http://goo.gl/GUKA8w => http://www.interviewbit.com/courses/programming/topics/time-complexity/
* Custom url: Allow the users to pick custom shortened url. 
	- Ex: http://www.interviewbit.com/courses/programming/topics/time-complexity/ => http://goo.gl/ib-time 
* Analytics: Usage statistics for site owner. 
	- Ex: How many people clicked the shortened url in the last day? 

### Design goals 
* Latency
	- Our system is similar to DNS resolution, higher latency on URL shortener is as good as a failure to resolve.
* Consistency vs Availability
	-  Both are extremenly important. However, CAP theorem dictates that we choose one. Do we want a system that always answers correctly but is not available sometimes? Or else, do we want a system which is always available but can sometime say that a URL does not exists even if it does? This tradeoff is a product decision around what we are trying to optimize. Let's say, we go with consistency here.
* URL as short as possible
	- URL shortener by definition needs to be as short as possible. Shorter the shortened URL, better it compares to competition.

### Estimation 


## Service 
```java
class TinyURL
{
	map<longURL, shortURL> longToShortMap;
	map<shortURL, longURL> shortToLongMap;
	
	shortURL insert( longURL )
	{
		if longToShortMap not containsKey longURL
			generate shortURL;
			put<longURL, shortURL> into longToShortMap;
			put<shortURL, longURL> into shortToLongMap;
		return longToShortMap.get(longURL);
	}

	longURL lookup( shortURL )
	{
		return shortToLongMap.get( shortURL );
	}
}
```

### shortURL insert( longURL )
#### Encode 
##### Traditional hash function
* Types
	- Crypto hash function: MD5 and SHA-1
		+ Secure but slow
	- Fast hash function: Murmur and Jenkins
		+ Performance
		+ Have 32-, 64-, and 128-bit variants available

* Pros
	- No need to write additional hash function, easy to implement
	- Are randomly distributed
	- Support URL clean

* Cons

| Problem                             | Possible solution                                                                                                                                              | 
|-------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------| 
| Not short enough (At least 4 bytes) | Only retrieve first 4-5 digit of the hash result                                                                                                               | 
| Collision cannot be avoided         | Use long_url + timestamp as hash argument, if conflict happens, try again (timestamp changes) -> multiple attempts, highly possible conflicts when data is big | 
| Slow                                |                                                                                                                                                                | 

##### Base10 / Base62
* Base is important

| Encoding           | Base10     | Base62      | 
|--------------------|------------|-------------| 
| Year               | 36,500,000 | 36,500,000  | 
| Usable characters  | [0-9]      | [0-9a-zA-Z] | 
| Encoding length    | 8          | 5           | 

* Pros:
	- Shorter URL
	- No collision
	- Simple computation

* Cons:
	- No support for URL clean

#### Implementation 
```java
    public String longToShort( String url ) 
    {
        if ( url2id.containsKey( url ) ) 
        {
            return "http://tiny.url/" + idToShortKey( url2id.get( url ) );
        }
        GLOBAL_ID++;
        url2id.put( url, GLOBAL_ID );
        id2url.put( GLOBAL_ID, url );
        return "http://tiny.url/" + idToShortKey( GLOBAL_ID );
    }

    private String idToShortKey( int id )
    {    	
        String chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
        String short_url = "";
        while ( id > 0 ) 
        {
            short_url = chars.charAt( id % 62 ) + short_url;
            id = id / 62;
        }
        while ( short_url.length() < 6 ) 
        {
            short_url = "0" + short_url;
        }
        return short_url;
    }
```

### longURL lookup( shortURL ) 
#### Implementation 
```java
    public String shortToLong( String url ) 
    {
        String short_key = getShortKey( url );
        int id = shortKeytoID( short_key );
        return id2url.get( id );
    }

    private String getShortKey( String url ) 
    {
        return url.substring( "http://tiny.url/".length() );
    }

    private int shortKeytoID( String short_key ) 
    {
        int id = 0;
        for ( int i = 0; i < short_key.length(); ++i ) 
        {
            id = id * 62 + toBase62( short_key.charAt( i ) );
        }
        return id;
    }
```

## Storage 
### SQL 
#### Schema design 
* Two maps
	- longURL -> shortURL
	- shortURL -> longURL
* In order to store less data ( Given shortURL, its corresponding sequential ID can be calculated )
	- longURL -> Sequential ID
	- Sequential ID -> longURL
* Create index on longURL column, only needs to store one table
	- Sequential ID -> longURL

### NoSQL 

## Scale 