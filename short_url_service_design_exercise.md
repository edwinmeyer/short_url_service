###Short URL Service
####Design Exercise
Edwin Meyer

####Summary
This exercise outlines the design aspects of a URL shortening service similar to bitly.com, which generates and saves a short URL for an arbitrary input URL. The web resource can be accessed using this short URL instead of the (potentially long) actual URL.

The principal requirements for this service are to service a high volume of requests and to indefinitely store the URL-short URL associations. Only forward and reverse lookup services are required.

For convenience this service will subsequently be called "Shurl" (for "Short URL"), and "SHURL" will designate the shortened URL.

####Requirements
* SHURL Lookup -- Return the SHURL associated with a URL. Generate a new SHURL and store it for subsequent retrievals if non-extant.
* URL Lookup -- Return the URL associated with a SHURL. Return an error if non-extant.
* URL-SHURL has a 1-many relationship. It is permissible (though non-optimum) for multiple SHURLs to reference the same URL. However a given SHURL must always reference only a single URL.
* Low latency response to a high volume of queries.
* Indefinite (permanent) storage of URL-SHURL relationships. 
* Provide a REST-API interface as well as a fronting web interface.

####Design Summary
Shurl will be a horizontally scaled system which achieves high throughput through the use of a NoSQL database such as MongoDB, fronted by an in-memory cache such as Memcache.  A NoSQL rather than a SQL database is used because it is far more performant and Shurl lacks aspects that SQL best provides: transactions and very high record insertion reliability.

The hash key component of a SHURL is the basis for the horizontal scaling: sharding of the database and cache.  Each database shard stores a different hash key range, and the front-line application server handling the request will direct it to the proper cache/db server based upon the SHURL's hash key component. (The server must first generate the hash when a full URL is provided.) 

Shurl is comprised of several types of components:

* A round-robin or other DNS server that distributes incoming requests across the available application servers.
* Ruby application server -- Accepts REST API requests, decodes the hash key for a SHURL (or generates one for a URL), and dispatches lookup requests to the cache and database services hosted by the database server handling this hash key range. It also provides a web intermediary to the REST API and suport services.
* Database server -- Provides both cache and NoSql database services for the same hash key shard.

_Note: the hardware requirements for a system that can handle the request volumes disclosed by Bitly are surprisingly modest -- see below._

####Request Processing
There are only two types of requests processed by the basic Shurl system:  SHURL creation and SHURL-URL lookup.

#####SHURL Creation
Process a REST API request that provides a URL and responds with the associated SHURL, creating it if non-extant. Steps:

1. Generate the associated hash key.
2. Dispatch the URL and hash key to the database server that handles that hash key range:
   * The cache service is queried first. If the associated SHURL is found, jump to step 7.
   * Otherwise the database service is queried. If the SHURL is found, jump to step 6.
4. Unless the URL is found, send the hash key to the database service to obtain the next available disambiguator. (This will be Null if the key is non-extant.)
5. Request the database server to add a new record containing the URL, hash key, and disambiguator.
6. Combine the hash key and disambiguator to form the SHURL.
7. Send the URL-SHURL association to the cache service.
8. Return the new SHURL in the HTTP response. 

#####SHURL-URL Lookup 
Process a REST API request that provides a SHURL and responds with the associated URL. Steps:

1. Obtain the hash key as the first 8 characters of the SHURL.
2. Dispatch the SHURL to the cache service of the database server that handles that hash key range. The associated URL is returned if found.
3. Otherwise decompose the SHURL into the hash key and disambiguator (often Null) and request the database service to return the associated URL . 
4. Return a 404 Not Found response if the URL is not in the database.
5. Send the URL-SHURL association to the cache service for the shard containing the hash key.
6. Return the retrieved URL in the HTTP response. 

Notes:

* The database server uses the hash key provided by the app server and does not independently recalculate it.
* The web interface intermediary returns a 301 Moved Permanently response so that the URL's resource is displayed by the requesting browser.

####SHURL Hash Key and Format
A SHURL consists of the hash key and possible disambiguation character(s), e.g. *A4dGHe6YY*. All hash keys are the same length (8 chars.) A disambiguating string is appended in case of key collisions, almost always of length 1.
The hash key component of a SHURL is a basic design element -- It is used to dispatch an incoming request to the specific database server that handles the hash key range it belongs to.

####Database Structure
Aside from tables implementing support functionality, the database consists of a single table having three fields:

* URL
* hash key -- the 8-character alphanumeric string generated from the URL by the hash function.
* disambiguation -- an alphanumeric string that distinguishes the URL in case of hash key collisions. Null for the first URL-SHURL association and almost always no more than a single character.

(The SHURL consists of the hash key plus the disambiguation, if any)

Three indicies are required:
* On URL -- to retrieve the associated hash key & disambiguation.
* On hash key/disambiguation -- to retrieve the associated URL.
* On hash key alone -- to obtain the next available disambiguation (highest in sort order) for the key. Required when storing a new URL-SHURL association.

####Planned Query Volume and System Requirements
Per http://dev.bitly.com/data_apis.html (date unknown): 
> we see about 4 billion clicks a month, and millions of new links a day. 

Arbitrarily assuming that the new link volume is 5 percent that of link accesses (clicks), this works out to 131,416,838 clicks/day (1,521/sec.) and 6,570,842 new links/day (76/sec.)

Based upon this, Shurl is to plan for 2,000 SHURL references per second and 100 new SHURLs/sec.

A brief scan of several benchmarking documents

* Facebook -- http://bit.ly/1HDAZjC
* Mongodb blog post -- http://bit.ly/1LxdX0i
* Hyperdex -- http://bit.ly/1MjudDY) 

suggests that a NoSQL database such as MongoDb can achieve a throughput of 5,000+ operations/sec. and that Memcache can handle 50,000+ requests/sec.

This indicates that the hardware requirements to support a request volume similar to Bitly's are quite modest, possibly as little as two or three parallel application and database server instances each.

####Permanent Storage Requirement
Assuming that an average URL is 40 chars and a SHURL is 10 chars in length, with 50% storage overhead, a URL-SHURL pair occupies 75 characters of storage.
Permanent storage can be added incrementally as needed (with resharding as the number of storage servers increase.) At 100 new SHURLs per sec. (3.2 billion/year) and 75 bytes per stored SHURL, a quite modest 237 gigabytes of additional storage need be added each year. (This wasn't at all modest only a couple of decades ago.)

####Cache Requirement
An in-memory cache (implemented by Memcache or similar cache module) that discards least recently used requests will greatly improve responsiveness and reduce server count. The required amount of cache memory is calculated using the simplifying assumption that a newly created SHURL is intensively accessed for a fairly short period after creation (and publication), and only sporadically thereafter. Assume:

* the "active period" for a new SHURL is 14 days
* the creation rate is 100/sec. (= 120.96 million per 14 days)
* there are 100 immediate accesses per average new SHURL
* URL-SHURL storage requires 75 bytes

Based upon these assumptions, the total main memory cache storage requirement is 907 gigabytes. This is quite do-able, especially considering that the cache will be distributed among the sharded servers.

####SHURL Key Length Requirements
While more servers and storage can be incorporated as needed, it will prove awkward if the initial choice of a SHURL hash key length results in frequent name space collisions years later. Therefore, the initially chosen key length should make quite generous assumptions about the number of keys to be accommodated.

Arbitrarily assuming that "permanent" storage means accommodating 50 years worth of new SHURLs generated at a  rate of 100,000/sec. (considerably higher than the 100/sec. planned rate), this translates to accommodating 158 trillion (10**12) SHURLs. 

If the hash key is alphanumeric, each character can encode 26+26+10 = 62 possibilities, and a key of length n can encode 62**n unique SHURLs.  Choosing a key length n=8 provides 218 trillion SHURLs. While the namespace will be sparse (unused keys) and there will also be key collisions, a key of length 8 characters will be chosen for simplicity.

Note: While most SHURLs will be 8 characters in length, additional disambiguating characters can be appended in the case of name collisions. The necessity for more than one disambiguating character (> 62 collisions) should be extremely rare.

