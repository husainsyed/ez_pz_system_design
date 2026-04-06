
**Functionality (Functional reqs):**
(User should be able to):

1. requests ride: user id, user pickup location (long/lat), user end location (long/lat)
2. alert drivers: user pickup location
3. broadcast location: show rider/driver each others location (driver id, driver location, user id, user location)

**Non functional:**
1. rider can only request ride at a time


**Capacity:**
1. 200 million riders, 10 million drivers
2. Drivers always have to broadcast their locations (consistent pings -> horizontal scaling)

#### Requesting a ride:
1. Ride start and end address -> convert these to longitude and latitude w/ geocoding api
2. Google Maps API can compute ride distance & time -> leading to pricing estimates

3. When user requests a ride -> map that to a persistent db schema (not cache cause user closes the app/cache is lost, they are requesting a duplicated ride) --> remember one ride at a time.

**Rides Table:**
- ride id (long)
- passenger id (long)
- driver id (long)
- creation timestamp (datetime)
- start location (LatLong)
- end location (LatLong)
- ride status (string) {requested, claimed, active, complete, canceled}
	- always think of when you request a ride, whats happening behind the scenes
	- requested- requested a ride, claimed- driver verified, active- taking the ride, complete-ride completed, canceled- ride canceled

**Auxiliary (helper) table- active_rides**
- indexes to passenger id and ride id in the main **rides table**
- faster lookup (instead of looking up millions and billions of rows in the main rides table), low latency
- only small subset of the main table


**Matching to a driver:**
- broadcast the rider location to nearby drivers
- continue broadcasting until someone accepts
- upon acceptance, no other driver can accept the ride:
	![[Pasted image 20260219085400.png | 500]]
- We use geospatial indexing --> determines close pts within the lat/long radius
- Once server has a list of potential drivers, we can:
	- notify all of them, some of them, one of them, score based on location, if they're completing ride etc
	  
- Ways to send notification:
	- long polling: driver continuously sends request- we just need notifications lol, not continuous locations. good: stateless but continous.
	- web sockets (client->server, persistent, real-time UI updates): bidirectional, the driver app notifies whenever its closeby.
	- analogy:
	  call the restaurant every 30 secs to see if they got a table ready (long polling)
	  restaurant calls you once they have the table ready (web sockets)
	  
	- We use web sockets