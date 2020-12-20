---
layout: default
---

# Designing a resilient delivery system

Last mile delivery services like UberEats, Doordash and SkipTheDishes have become a staple of our new normal. Technology like this would seem almost magical a few decades ago: where pressing a button on a handheld piece of glass summons a stranger with your order at your doorstep. Despite how magical this technology is, it sure comes with its annoyances. More often than not, I have noticed that components of the delivery system seem to break down with little to no recourse or graceful failures. Some of the recent annoyances / failures have been :

1. The dasher reached the restaurant but could not pick up my order since the doordash's restaurant portal was down.
   - Doordash did not notify me that the order can't be picked up.
   - Doordash did not notify the dasher that the order can't be picked up.
   - Doordash kept increasing my delivery estimate making me hungrier and [hangrier](https://www.urbandictionary.com/define.php?term=hangry)

2. Delivery apps often go down especially around peak lunch / dinner hours.

Developing a three sided market place like this is an especially difficult challenge. Being a regular user of these apps, I wanted to take on the mental exercise of designing what a more resilient tech architecture for apps like these might look like. At the most basic, these platforms can be split into three main pieces:

1. Customer Facing platform
   - allows a user to view menus / search and place orders
   - allows a user to track the status of their order

2. Restaurant facing platform
   -  allows a restaurant to add / update their menu.
   -  allows a restaurant to set themselves as open / closed
   -  allows a restaurant to receive orders and update their status
   These are my assumptions. I have never really used the restaurant facing side of these apps

3. Partner facing platform
   - allows a partner to accept a pickup
   - allows a partner to view the delivery address
   - allows a partner to update the order status
   These are my assumptions. I have never really used the partner facing side of these apps.

Lets take a stab at "reverse engineering" each of these platforms and see how we can make them more resilient or at-least try to understand the challenge around making it more resilient.


## Customer Facing Platform
Lets take a look at one of the customer facing applications and try to figure out all the ways that this application can fail during a server outage. Once we map out the several ways it can fail, we can try to reverse engineer a possibly more resilient system architecture. One of the easiest ways of simulating a completely un-response server is to simply put your phone on airplane mode and open the app.
So lets try it out.

The app loads fine and displays the restaurants that are available near my delivery address. This indicates that the client side application
is most likely caching some data locally on my device since the last time I opened it. Whats missing here is any indication that the app
cannot connect to its API server and that the information I am viewing might have changed.

<blockquote class="imgur-embed-pub" lang="en" data-id="a/ZjsiSmd" data-context="false" ><a href="//imgur.com/a/ZjsiSmd"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Using any of the quick filters leaves the grid un-responsive. Which means that the app most likely relies on an API call to do the filtering
on the grid. However, now the app clearly says that there is a connectivity issue.

<blockquote class="imgur-embed-pub" lang="en" data-id="a/mUIfFFQ" data-context="false" ><a href="//imgur.com/a/mUIfFFQ"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>


Using any of the quick cuisine preferences also leaves the grid un-response. This is very likely an API call to filter the grid as well.

<blockquote class="imgur-embed-pub" lang="en" data-id="a/rqJWlqW" data-context="false" ><a href="//imgur.com/a/rqJWlqW"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

I can understand the need for an extra API call to do filtering. There might be some metadata that is needed to filter that isn't available when the grid loads and needs to be fetched only when we try to filter. A possible optimization here could be :

1. Look for the most regularly used filters and cuisine selectors and cache the information you might need to filter it.

As an example, I tend to order Indian food quite a bit, it might make sense to cache a cuisine related attributes when the restaurant grid data is cached on my device. That way, it would at-least allow the cuisine filters to work. There is a fine balance to be struck here since
we don't want to cache everything. We can be smart and personalize caching in this situation. This would protect the perception of functionality in some cases. At the very least it still allows the user to browse somewhat if there's an outage.

On a similar note, attributes that are used to filter by ratings, dash-pass, $$ can also potentially be cached depending on a user's preference. These values don't tend to change very frequently and its okay for them to be eventually consistent. This would allow the experience to be somewhat seamless even during downtime.

An odd thing I noted was that if I moved over to the pickup tab, it shows me "pickup" eligible restaurants on a map but somehow isn't able to filter the grid using the "pickup" quick select option.  ¯\_(ツ)_/¯

<blockquote class="imgur-embed-pub" lang="en" data-id="a/LPzLAp4" data-context="false" ><a href="//imgur.com/a/LPzLAp4"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

On certain occasions, loading the offers page without an internet connection throws this error:

<blockquote class="imgur-embed-pub" lang="en" data-id="a/NRiaqS1" data-context="false" ><a href="//imgur.com/a/NRiaqS1"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

There is definitely room for a more graceful failure here.


The app also often tends to be slower in general during times of peak load around lunch and dinner hours. The add-to-cart button is where the first cracks of elevated load typically show up on doordash. When you click it, it doesn't add anything to cart. A clear outage. But the fact that the app seems slower in general tends to point to some kind of monolithic architecture where the all functionalities are being powered by a tightly coupled codebase so elevated load on one piece tends to affect the other. A possible way this can be optimized is obviously by splitting something like this up into micro-services that perform logically independent parts of the application. In order to study how a probable monolith can be split up into micro-services, lets look at the high level data access patterns for the app. I am not going to dig into minute details like filtering and personalization but mostly around how a bare bones app would access data. The most common patterns appear to be :

- GET information of all restaurants around a location. These get arranged into several carousels [This powers the home screen restaurant grid]
- GET search results for restaurants around a location. [This includes tracking recently viewed, top searches etc]

- GET information of a particular restaurant, its menu items, its prices, its reviews etc. [This powers the menu page]
- GET detailed information of a restaurant's menu items.

- POST an item to your cart.
- GET the current items in your cart.
- POST an order to a restaurant

- GET the status of your order.
- GET the status of your dasher.

I tried to group the data access patterns together logically in a way where we might attempt to club them together into micro-services. A bulk of the user's discovery experience on the app will likely be powered by the first two access patterns

- GET information of all restaurants around a location. These get arranged into several carousels
- GET search results for restaurants around a location.


The information that gets shown here is highly contextual. Its based on the user's delivery location.  Different users at the same location appear to get a different ordering of the same base set of restaurants, so there is likely some personalization logic in the background, but I am going to leave that out of the mix for now. If we switch over to a different delivery address in a different city, the screen loads for a bit and displays a different set of restaurants into the grid arranged as carousels. My estimate is that some backend logic is taking the user's delivery location as one of the primary inputs and its mapping it to a set of restaurants.

Given that the set of restaurants changes even if you move the delivery address by a few miles within the same city, its likely that candidate set is being restricted to some delivery radius. The delivery radius could be miles / kms. If they have a more sophisticated estimate of delivery times, it could even be ETA from the restaurant to your location. Let's keep things simple for now and assume that the backend server calculates a base set of restaurants within a certain radius from your delivery location. The algorithm also appears to personalize the order of this base set of restaurants using some user attributes. These are two separate and computationally expensive operations.

Given the facts and assumptions we have, I would guess that this experience is likely being powered by elastic search cluster(s) as the primary database. Its likely that each major metropolitan has its own index and each restaurant is represented as a document. The document likely has
several fields that can provide the information that needs to be displayed on the home page and for the restaurants to be grouped together into grids and carousels. Its likely that they might simply store lat / longitude information as fields in the document as well. A lot of this information on the screen doesn't change pretty often, its likely that partial updates happen on the indexes every so often to display changes in location, active status or change in names. I would guess that this ES cluster stores no data on the menu items of the restaurants itself or if it does, there are separate indexes for those.

<blockquote class="imgur-embed-pub" lang="en" data-id="a/IyrLR6j" data-context="false" ><a href="//imgur.com/a/IyrLR6j"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

## Scaling the API web service
When a request to display information on the homepage is received, the API web service likely augments it with additional preference metadata and runs a ES query to find restaurants within a radius of the requested location. The response to the query is likely post-processed and arranged in a way that it can be easily rendered on the device screen. Because of the way this information is processed, it could be pretty easy to keep this API server stateless. If it does need to get user preference information, it could possibly query another micro-service like a blackbox. This would allow for an API web service architecture that can scale horizontally with demand. As the number of concurrent requests come in, we could simply increase the number of API services to scale with the demand. There isn't much rocket science to this.

## Scaling the ES cluster
As far as the ES cluster goes, scaling an elastic search cluster has 3 main components that we need to consider that can affect performance.
1. Size of each index.
Every index is divided into shards. Each shard is assigned to a node. Each node has multiple replicas. So your ES index shards are distributed across several nodes. When a new search is received its transformed into a set of searches (one on each shard). Each shard returns the document that match the search query and then the lists are merged and sorted. The number of documents on a shard gives us an idea of how long a search on a shard takes. The number of shards on a node can give us an estimate on the memory required. So faster response time can possibly be achieved by splitting up your index into several smaller shards but you are trading that off with search concurrency. Several smaller shards would just mean more shards on the node and wouldn't mean less documents in total on the same node.

<img src="https://miro.medium.com/max/1400/1*U39NfbVwkht1kLex8scVIw.png" width="600" height="600">


2. Throughput : One of the aims of scaling is to be able to manage several search requests at the same time without significant degradation in response time. In Elastic Search, a search request can be received by any node. That node then co-ordinates the search on shards in other nodes. Searches are performed by threads on a node, so the number of concurrent searches are generally controlled by the number of threads you have running on a node. The number of available threads on a node is controlled by the `thread_pool.search` setting on the cluster. Increasing the number of threads can help with search concurrency.

3. Size of the cluster : Simple increasing the number of nodes can help speed up performance for sure provided there are enough shards to be assigned to every node.

In general, there isn't a silver bullet for this. This is more like turning 4 knobs that you control:
- shard size        : This affects the time to perform search on that shard. The smaller the shard the faster the search.
- replica factor    : Number of replicas available for each shard. This impacts how many shards a node has.
- threads on a node : This might help with concurrency. More threads = parallel searches on the node.
- cluster size      : more nodes can help with throughput. However this depends on the number of documents being handled by each node.

A good idea when high throughput is required, is to consider increasing the threads on each node, adding nodes and then increasing the replica factor so that each node at-least has one shard per available thread.

With these improvements and using other strategies like leveraging a CDN, we can probably ensure that the homepage carousels and the search function will work well and scale well with traffic. Using other caching strategies, graceful failures and device specific caching we can make this experience pretty seamless.

A similar kind of solution can be achieved for the following access patterns as well:
- GET information of a particular restaurant, its menu items, its prices, its reviews etc. [This powers the menu page]
- GET detailed information of a restaurant's menu items.

Elastic Search is probably not the best fit for this kind of information since its super tightly coupled to the restaurant and probably doesn't need to be searched. On most occasions, it needs to be arranged for fast bulk retrieval. If speed is what you are optimizing for, it could even make sense to store this data as Key Value Pairs on a redis. However for all intents and purposes a well optimized relational database will fit the bill. This data however might need more frequent updates especially if a restaurant changes its menu or marks things as sold out.It might make sense to arrange the relational databases in a master -  clone configuration where writes are performed on the master and the cloning database has indexes optimized for high read throughout for our data access patterns.

<img src="https://www.red-gate.com/simple-talk/wp-content/uploads/2019/04/a-screenshot-of-a-cell-phone-description-automati-1.png">
