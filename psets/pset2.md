# Problem Set 2: Composing Concepts

## Exercise 1: Concepts for URL Shortening

1. **Contexts.** The NonceGeneration concept ensures that the short strings it generates will be unique and not result in conflicts. What are the contexts for, and what will a context end up being in the URL shortening app?

   <details>
   <summary><b>Answer</b></summary>
    Contexts allow the same words to be used in different contexts while ensuring that each string it generates will not result in a conflict. In the URL shortening app, context are the base URL's. This allows two base URL's, for example `baseUrlA.com` and `baseUrlB.com` to both have shortened links that use the string `firstlink`, since `baseUrlA.com/firstlink` and `baseUrlB.com/firstlink` point to different addresses on the internet.
   </details>

2. **Storing used strings.** Why must the NonceGeneration store sets of used strings? One simple way to implement the NonceGeneration is to maintain a counter for each context and increment it every time the generate action is called. In this case, how is the set of used strings in the specification related to the counter in the implementation? (In abstract data type lingo, this is asking you to describe an abstraction function.)

   <details>
   <summary><b>Answer</b></summary>

   In the NonceGeneration spec, for each Context $c$ we model a set $c.used$ of strings already returned. Thus, our postcondition is: `generate(c)` returns $s \notin c.used$ and then adds $s$ to $c.used$.

   However, in the counter implementation we don’t store that set; we keep a per-context counter $c.count$ (which is initially 0) and implement `generate(c)` as: let $k = c.count$, return `String(k)`, then set $c.count := k+1$.

   The abstraction function relating the counter implementation to the spec is $$\alpha(c) = \{\text{String}(i) \mid 0 \le i < c.count\}.$$

   With $c.count=k$, $c.used$ under $\alpha$ is $\{\text{String}(0),\dots,\text{String}(k-1)\}$. Thus `String(k)` is a new unique string, and after an increment the original mapped set, $c.used$, becomes $c.used \cup \{\text{String}(k)\}$, which satisfies the spec.
   </details>

3. **Words as nonces.** One option for nonce generation is to use common dictionary words (in the style of yellkey.com, for example) resulting in more easily remembered shortenings. What is one advantage and one disadvantage of this scheme, both from the perspective of the user? How would you modify the NonceGeneration concept to realize this idea?
   <details>
   <summary><b>Answer</b></summary>

   One advantage of using a dictionary of words for the user is that it improves distribution of your shortened link to other users. Strings that are common words are easier to remember and are easier to verbally communicate since we encounter words like "dog" or "cat" more often than hashes like UUID that are mathematically unique and quick to generate.

   One disadvantage for the user is that the dictionary of words that is used is finite(even if it could be very large in practice). If the user has a lot of short links they could in theory "run out" of words which could be a problem. Additionally, if the nonce was only used one word strings from the dictionary, it could potentially pose a privacy concern/security risk as simple words like "dog" or "cat" are easily guessable.

   My updated concept addresses the limitations and risks associated with a finite dictionary of words that is used to create one word nonce strings by combining the finite word dictionary into nonces that can be made up of upto k-words. This increase the number of links that can be generated to |words in dictionary|^k, increasing the difficulty of guessing links, while maintaining easy to remember links. My implementation changes to do this are as follows:

   **concept** NonceGeneration [Context]

   **purpose** generate memorable word-based nonces within a context

   **principle** each generate returns a word (or k-word phrase) not returned before for that context

   **state**

   a words set of Strings

   a set of Contexts with

   - a k Number (the number of words combined to make the nonce)
   - a used set of Strings

   **actions**
   generate(context: Context) : (nonce: String)

   - **require:** |context.used| < |words|^context.k
   - **effect:** choose a sequence of context.k words that does not exist in context.used; join them using a "-"; add the new nonce to context.used; return the nonce.

    </details>

## Exercise 2: Synchronization Questions

1. **Partial matching.** In the first sync (called generate), the Request.shortenUrl action in the when clause includes the shortUrlBase argument but not the targetUrl argument. In the second sync (called register) both appear. Why is this?
   <details>
   <summary><b>Answer</b></summary>

   Each sync only binds what its _then_ clause needs. In the _generate_ sync, the only value required to invoke `NonceGeneration.generate` is the context(_shortUrlBase_). Therefore the _when_ matches `Request.shortenUrl` and binds just _shortUrlBase_ since _targetUrl_ isn't used , and adding it would over-constrain the match for no benefit. In the _register_ sync, the _then_ calls `UrlShortening.register` with _shortUrlBase_ and _targetUrl_ , and the _nonce_ from `NonceGeneration.generate`. Therefore, the _when_ must bind both arguments and the nonce. Partial matching keeps each sync minimal and avoids accidental coupling.
   </details>

2. **Omitting names.** The convention that allows names to be omitted when argument or result names are the same as their variable names is convenient and allows for a more succinct specification. Why isn’t this convention used in every case?
   <details>
   <summary><b>Answer</b></summary>

   This convention isn't used in every case because we are only able to do it when an action's parameter/result name matches the local variable name. A good example of when this isn't the case is in the generate sync where we need to "rename" the variable _shortUrlBase_ to the _context_ (`NonceGeneration.generate (context: shortUrlBase)`). While succinct when possible, for readability and clarity in our specifications sometimes it is preferable to explicitly map `argName: varName` especially when wiring outputs to inputs with different names across actions.
   </details>

3. **Inclusion of request.** Why is the request action included in the first two syncs but not the third one?
   <details>
   <summary><b>Answer</b></summary>

   The _then_ action of the first two syncs are triggered by the parameterization in the user's request. (_generate_ needs `shortUrlBase` from `Request.shortenUrl` and _register_ requires `nonce` from `NonceGeneration.generate()` in addition to `shortUrlBase` and `targetUrl` from `Request.shortenUrl`). In comparision the _setExpiry_ sync only requires the `shortUrl` returned from the completion of`UrlShortening.register()` to have the necessary information to invoke its _then_ clause. Including`Request.shortenUrl` in this sync would over-constrain the match and risk triggering without a successful registration.
   </details>

4. **Fixed domain.** Suppose the application did not support alternative domain names, and always used a fixed one such as “bit.ly.” How would you change the synchronizations to implement this?
   <details>
   <summary><b>Answer</b></summary>

   With a fixed domain, we change only generate and register. In the _generate_ sync, the _when_ just matches `Request.shortenUrl()` and the then invokes `NonceGeneration.generate(context: "bit.ly")`. In the _register_ sync, the _when_ only needs to bind the pieces it will pass to the methods it invokes in the _then_; namely: `targetUrl` from `Request.shortenUrl(targetUrl)` and `nonce` from the completion of `NonceGeneration.generate(): (nonce)`. The _then_ clause then invokes `UrlShortening.register(shortUrlSuffix: nonce, shortUrlBase: "bit.ly", targetUrl)`. The _setExpiry_ sync remains is unchanged.
   </details>

5. **Adding a sync.** These synchronizations are not complete; in particular, they don’t do anything when a resource expires. Write a sync for this case, using appropriate actions from the ExpiringResource and URLShortening concepts.
   <details>
   <summary><b>Answer</b></summary>

   **sync** expire

   - **when:** `ExpiringResource.expireResource (): (resource: shortUrl)`
   - **then:** `UrlShortening.delete (shortUrl)`

   This sync is triggered once a resource actually expires. The _when_ clause binds the returned resource to the variable `shortUrl`. The _then_ clause then deletes that shortened Url.
   </details>

## Exercise 3: Extending the Design

1. Design a couple of additional concepts to realize this extension, and write them out in full (but including only the essential actions and state). It should not be necessary to make any changes to the existing concepts.
   <details>
     <summary><b>Answer</b></summary>
   <hr/>

   **concept** Analytics[ShortUrl]

   **state**

   - a set of ShortUrls with

     - a visits Number

   **actions**

   - register(shortUrl)

     - **requires:** shortUrl not already in the set of ShortUrls
     - **effect:** adds shortUrl to the ShortUrls set with `visits = 0`

   - remove(shortUrl)

     - **requires:** shortUrl exists in ShortUrls
     - **effect:** removes shortUrl from the ShortUrls set

   - increment(shortUrl)

     - **requires:** shortUrl exists in ShortUrls
     - **effect:** increments `ShortUrls[shortUrl].visits` by 1

   - get(shortUrl) : (count: Number)

     - **requires:** shortUrl exists in ShortUrls
     - **effect:** returns `ShortUrls[shortUrl].visits`

   <hr/>

   **concept** Ownership[ShortUrl,User]

   **state**

   - a set of ShortUrls with

     - an owner User

   **actions**

   - assign(shortUrl, owner: User)

     - **requires:** shortUrl not already in the set of ShortUrls
     - **effect:** adds shortUrl to the ShortUrls set with `owner = owner`

   - remove(shortUrl):

     - **requires:** shortUrl exists in ShortUrls
     - **effect:** removes shortUrl to the ShortUrls set

   - ownerOf(shortUrl): (user: User)

     - **requires:** shortUrl exists in ShortUrls
     - **effect:** returns `ShortUrls[shortUrl].owner`

   <hr/>

   </details>

2. Specify three essential synchronizations with your new concepts: one that happens when shortenings are created; one when shortenings are translated to targets; and one when a user examines analytics.
   <details>
   <summary><b>Answer</b></summary>
   <hr/>

   **sync** onCreate

   **when:**

   - Request.shortenUrl(user, targetUrl, shortUrlBase);
   - UrlShortening.register(): (shortUrl)

   **then:**

   - Ownership.assign(shortUrl, owner: user);
   - Analytics.register(shortUrl)

   <hr/>

   **sync** onView

   **when:**

   - AnalyticsRequest.view(user, shortUrl)
   - Ownership.ownerOf(shortUrl) == user

   **then:**

   - Analytics.get(shortUrl): (count)

   <hr/>

   **sync** onLookUp

   **when:**

   - UrlShortening.lookup(shortUrl): (targetUrl)

   **then:**

   - Analytics.increment(shortUrl)

   <hr/>

   </details>

3. As a way to assess the modularity of your solution, consider each of the following feature requests, to be included along with analytics. For each one, outline how it might be realized (eg, by changing or adding a concept or a sync), or argue that the feature would be undesirable and should not be included:

   - Allowing users to choose their own short URLs

      <details>
        <summary><b>Answer</b></summary>
        We can handle custom URLs via a sync triggered when a `Request.shortenUrl` is made with a `desiredSuffix` parameter. We register the URL as usual and invoke the Analytics and Ownership concept actions accordingly.

     **sync:** onCustomUrl

     **when:**

     - Request.shortenUrl(user, desiredSuffix, shortUrlBase, targetUrl)
     - UrlShortening.register(shortUrlSuffix: desiredSuffix, shortUrlBase, targetUrl): (shortUrl)

     **then:**

     - Ownership.assign(shortUrl, owner: user)
     - Analytics.register(shortUrl)

      </details>
     <hr/>

   - Using the “word as nonce” strategy to generate more memorable short URLs

      <details>
        <summary><b>Answer</b></summary>
        We can simply change the implementation of `NonceGeneration` to use words as nonces. This leaves all syncs unchanged.

     **concept** NonceGeneration \[Context]

     **state**

     a Words set of Strings

     a set of Contexts with

     - a k Number (the number of words combined to make the nonce)
     - a used set of Strings

     **actions**

     generate(context: Context) : (nonce: String)

     - **require:** |context.used| < |Words|^context.k
     - **effect:** choose a sequence of `context.k` words that does not exist in `context.used`; join them with “-”; add the new nonce to `context.used`; return the nonce.

      </details>
     <hr/>

   - Including the target URL in analytics, so that lookups of different short URLs can be grouped together when they refer to the same target URL

      <details>
        <summary><b>Answer</b></summary>
        We introduce a new concept `AnalyticsByTarget` that provides grouped analytics by `(user, targetUrl)`.

     **concept** AnalyticsByTarget

     **state**
     a set of UserTargetCounts with

     - a user User
     - a targetUrl String
     - a count Number

     **actions**

     createUserTarget(user, targetUrl)

     - **requires:** no `userTargetCount` exists with this `(user, targetUrl)`
     - **effect:** add a new `UserTargetCount` with `user = user`, `targetUrl = targetUrl`, and `count = 0`

     increment(user, targetUrl)

     - **requires:** a `userTargetCount` exists with this `(user, targetUrl)`
     - **effect:** increase `userTargetCount.count` by 1

     getByTarget(user: User, targetUrl) : (count: Number)

     - **requires:** a `userTargetCount` exists with this `(user, targetUrl)`
     - **effect:** return `userTargetCount.count`

     **Associated Syncs**

     **sync** onCreateGrouped

     **when:**

     - Request.shortenUrl(user, targetUrl, shortUrlBase)
     - UrlShortening.register(): (shortUrl)

     **then:**

     - AnalyticsByTarget.createUserTarget(user, targetUrl)

     **sync** onLookupGrouped

     **when:**

     - UrlShortening.lookup(shortUrl): (targetUrl)
     - Ownership.ownerOf(shortUrl): (user)

     **then:**

     - AnalyticsByTarget.increment(user, targetUrl)

     **sync** onViewGrouped

     **when:**

     - Request.viewAnalyticsByTarget(user, targetUrl)

     **then:**

     - AnalyticsByTarget.getByTarget(user, targetUrl): (count)

      </details>
     <hr/>

   - Generate short URLs that are not easily guessed;
      <details>
        <summary><b>Answer</b></summary>
        Short URLs are public identifiers, not secrets. Optimizing for “unguessability” hurts memorability and adds complexity without meaningful security. If we wanted to improve security it would be better to implement some type of access control or mitigate enumeration of possible keys via rate limiting techniques instead. That said, if we ever need more opacity and we are okay with sacrificing new memorability, we can swap in a higher-entropy generator behind NonceGeneration later without changing any syncs.
      </details>
     <hr/>

   - Supporting reporting of analytics to creators of short URLs who have not registered as user.
      <details>
        <summary><b>Answer</b></summary>
        Without authenticated identity, we can’t reliably establish ownership or authorization. Most cookie/IP/token schemes are spoofable and leak analytics beyond the intended owner of those analytics. This creates privacy and support risks with little upside to no upside. Enforcing a the policy that analytics are available only to registered owners, is clear and reduces unnecessary complexity, especially when becoming a registered user could be trivial extra cost for the user.
      </details>
     <hr/>
