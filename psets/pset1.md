# Problem Set 1

## Exercise 1: Reading a Concept

1. **Invariants.** What are two invariants of the state? (Hint: one is about aggregation/counts of items, and one relates requests and purchases). Say which one is more important and why; identify the action whose design is most affected by it, and say how it preserves it.
   <details>
   <summary><b>Answer</b></summary>

   First, a lifecycle/visibility invariant: at all times a registry is either active or inactive; create always starts it inactive; only open can switch inactive→active and only close can switch active→inactive; and no purchases are permitted unless the registry is active. Second, a counts/aggregation invariant: for any request in a registry, its remaining count is never negative and the total quantity purchased for that item never exceeds the total quantity requested (equivalently, under addItem and purchase we maintain remaining = requested_so_far − purchased_so_far).

   Between the two, the lifecycle invariant is more important because it protects the recipient’s intent about when buying is allowed; violating it leads to the worst user-visible failures (people buying when the registry shouldn’t be open), whereas the counts invariant is already strongly guarded by purchase. The action most affected by the lifecycle invariant is purchase: its precondition that the registry “exists and is active” gates buying strictly to the open period and, since it doesn’t modify the active flag, it preserves the visibility invariant.

</details>

2. **Fixing an action.** Can you identify an action that potentially breaks this important invariant, and say how this might happen? How might this problem be fixed?
   <details>
   <summary><b>Answer</b></summary>
    The lifecycle invariant can be broken because open and close have no actor and no ownership check, so any user could toggle a registry. To preserve “status reflects the recipient’s intent,” add a user parameter to both actions and require user = registry.owner; otherwise the action has no effect. This confines activation and deactivation to the owner and keeps the active flag aligned with the recipient’s wishes.

</details>

3. **Inferring behavior.** The operational principle describes the typical scenario in which the registry is opened and eventually closed. But a concept specification often allows other scenarios. By reading the specs of the concept actions, say whether a registry can be opened and closed repeatedly. What is a reason to allow this?
   <details>
      <summary><b>Answer</b></summary>
   A registry can be opened and closed repeatedly: open requires it be inactive and close requires it be active, so the actions can toggle indefinitely. This flexibility is useful—for example, the owner might temporarily close the registry to pause purchases while editing items or to correct an accidental toggle—so allowing repeated open/close seems is reasonable.

</details>

4. **Registry deletion.** There is no action to delete a registry. Would this matter in practice?
   <details>
   <summary><b>Answer</b></summary>
   Deletion isn’t necessary in practice. Closing removes public visibility, and most actions already require an open registry. Keeping a closed registry preserves purchase receipts for thank-yous, returns, and audits. Given that storage probably is cheap for this type of data, and there are minimal privacy concerns with registry purchase history, archival likely beats hard-delete here.

</details>

5. **Queries.** What are two common queries likely to be executed against the concept state? (Hint: one is executed by a registry owner, and one by a giver of a gift.)
   <details>
   <summary><b>Answer</b></summary>

   **For givers:** given a registry and an item, return how many units remain to be purchased for that item (and treat the item as not purchasable if the remaining count is zero).

   **For the owner:** given a registry, show its current state—active/inactive, owner, and for each item the requested total, purchased total, and remaining.

</details>

6. **Hiding purchases.** A common feature of gift registries is to allow the recipient to choose not to see purchases so that an element of surprise is retained. How would you augment the concept specification to support this?
   <details>
   <summary><b>Answer</b></summary>

   To support surprise for the recipient, each registry has a hidePurchases flag (default false) and two owner-only actions: hidePurchases and showPurchases. Hiding requires that the registry exists, the caller is the owner, and purchases are currently visible; its effect is to set hidePurchases to true. Showing requires the registry exists, the caller is the owner, and purchases are currently hidden; its effect is to set hidePurchases to false. When purchases are hidden, owner-facing queries suppress any information that would reveal what has been bought (purchaser names, purchased totals, remaining counts), showing only the originally requested items and quantities (or neutral placeholders), while giver-facing queries remain unchanged so they can still avoid duplicate purchases.

</details>

7. **Generic types.** The User and Item types are specified as generic parameters. The Item type might be populated by SKU codes, for example. Explain why this is preferable to representing items with their names, descriptions, prices, etc.
   <details>
   <summary><b>Answer</b></summary>

   Treating Item as a generic parameter (instantiated with stable IDs like SKUs) keeps the registry decoupled from any particular product catalog: the registry stores references, not mutable copies of names/descriptions/prices. That avoids drift and duplication (prices/names change elsewhere), preserves representation-independence (works with many stores/catalog schemas), and makes queries/joins clean—lookup by ID pulls the latest display info without corrupting the registry’s core state.

</details>

## Exercise 2: Extending a Familiar Concept

1. Complete the definition of the concept state.  <details>
   <summary><b>Answer</b></summary>

   To complete the concept state I would define it as follows:

   a set of Users with

   - a unique username String
   - a passwordHash String

</details>

2. Write a requires/effects specification for each of the two actions. (Hints: The register action creates and returns a new user. The authenticate action is primarily a guard, and doesn’t mutate the state.)  <details>
   <summary><b>Answer</b></summary>

   register(username, password): (user: User)

   - **requires:** no user with this username exists.

   - **effects:** create and return a new User with username and passwordHash = H(password).

   authenticate(username, password): (user: User)

   - **requires:** a user with this username exists and H(password) equals the stored passwordHash.

   - **effects**: return that user; the state is unchanged.

</details>

3. What essential invariant must hold on the state? How is it preserved?
   <details>
   <summary><b>Answer</b></summary>

   Username uniqueness is the essential invariant that must be preserved. This means that at any time, no two users share the same username. This ensures a given username always identifies the same User after registration, so any User who is authenticated is treated each time as the same User each time. This is preserved because register requires that the username not already exist before creating a new User, and authenticate does not mutate state.

</details>

4.  One widely used extension of this concept requires that registration be confirmed by email. Extend the concept to include this functionality. (Hints: you should add (1) an extra result variable to the register action that returns a secret token that (via a sync) will be emailed to the user; (2) a new confirm action that takes a username and a secret token and completes the registration; (3) whatever additional state is needed to support this behavior.).
    <details>
    <summary><b>Answer</b></summary>

    **Updated State Representation:**

    a set of Users with

    - a username String

    - a passwordHash String

    - an email String

    a Pending set of Users with

    - a token String

    - a tokenExpires DateTime

    a Confirmed set of Users

    **Updated Action Definitions:**

    register(username: String, password: String, email: String): (user: User, token: String)

    - **requires:** the username does not exist.

    - **effects:** create a new User with the given fields, add the user to Pending with a fresh secret token, and return (user, token) so the app can email it.

    confirm(username: String, token: String)

    - **requires:** a Pending user with this username and matching token (and, if present, that now ≤ tokenExpires).

    - **effects:** move the user from Pending to Confirmed and clear the token info.

    authenticate(username: String, password: String): (user: User)

    - **requires:** a Confirmed user with this username exists and H(password) = passwordHash.

    - **effects:** return that user; no state changes.

</details>

## Exercise 3: Comparing Concepts

Passwords authenticate human sign-in and, once accepted, give the user full account powers. PATs, on the other hand are per-tool bearer tokens for APIs/CLI; you can create many PATs, each constrained by scopes and (for org resources) explicit grants. This ensures access to resources is least-privileged and any security compromises are isolated to that token.

<details>

<summary><b>PersonalAccessTokens Concept</b></summary>

**concept** PersonalAccessToken [User, Org, Scope]

**purpose** provide API/CLI access without using a password

**principle**

- a user generates tokens;
- the user can add or remove scopes on a token;
- an org admin grants or revokes a token for that org;
- a client authenticates by sending username and token;
- access to an org’s resources also requires that the org has granted the token, and protected operations require appropriate scopes;
- the user may delete the token to revoke it everywhere.

**state**

a set of Tokens with

- an owner User
- a value String
- a scopes set of Scope

a set of Orgs with

- an admins set of Users
- a grantedTokens set of Tokens

**actions**

generate (by: User): (token: Token, value: String)

- **requires** by exists

- **effects** create token with owner = by, value a fresh secret, scopes empty; return (token, value)

addScope (by: User, token: Token, scope: Scope)

- **requires** token exists and by = token.owner

- **effects** add scope to token.scopes

removeScope (by: User, token: Token, scope: Scope)

- **requires** token exists and by = token.owner and scope in token.scopes

- **effects** remove scope from token.scopes

grantAccess (admin: User, org: Org, token: Token)

- **requires** org exists and admin in org.admins and token exists

- **effects** add token to org.grantedTokens

revokeAccess (admin: User, org: Org, token: Token)

- **requires** org exists and admin in org.admins and token exists

- **effects** remove token from org.grantedTokens

revoke (by: User, token: Token)

- **requires** token exists and by = token.owner

- **effects** remove token from Tokens and from every org.grantedTokens

authenticate (username: String, tokenValue: String): (credential: Token)

- **requires** a token t with t.owner.username = username and t.value = tokenValue exists

- **effects** return t

</details>

## Exercise 4: Defining Similar Concepts

### Concept 1: URL Shortener

<details>

<summary><b>URLShortener Concept</b></summary>

**concept** URLShortener [User]

**purpose** map long URLs to short stable suffixes

**principle**

- a user submits a long URL and may propose a suffix;
- if a requested suffix is unused it is claimed; if it is already used, creation does not occur;
- if no suffix is requested, the system generates a fresh unused suffix;
- anyone who visits the short URL is redirected to the target;
- the owner may update or deactivate the link.

**state**

a set of ShortLinks with

- an owner User
- a suffix String
- a targetUrl String
- an active Flag

**actions**

shortenUrl (by: User, targetUrl: String, optional requestedSuffix: String): (link: ShortLink)

- **requires:** by exists and if requestedSuffix is provided, no ShortLink exists with suffix = requestedSuffix

- **effects:** creates a ShortLink link with owner = by,
  suffix = requestedSuffix if requestedSuffix is provided else a fresh unused suffix,
  targetUrl = targetUrl, active = true; return link

updateTargetUrl (by: User, link: ShortLink, newUrl: String)

- **requires:** link exists and by = link.owner

- **effects:** set link.targetUrl = newUrl

deactivateLink (by: User, link: ShortLink)

- **requires:** link exists and by = link.owner and link.active = true

- **effects:** set link.active = false

activateLink (by: User, link: ShortLink)

- **requires:** link exists and by = link.owner and link.active = false

- **effects:** set link.active = true

redirectFromSuffix (suffix: String): (targetUrl: String)

- **requires:** a ShortLink link with link.suffix = suffix and link.active = true to exist

- **effects:** return link.targetUrl

</details>
<details>
<summary><b>Subtleties</b></summary>

- suffixes are globally unique
- if a requested suffix is taken, shortenUrl’s precondition fails and no link is created.
- fresh(random) suffixes are generated only when none is requested.
</details>

### Concept 2: Time-Based One-Time Password (TOTP)

<details>

<summary><b>TOTP Concept</b></summary>

**concept** TOTP [User]

**purpose** add a short-lived code step to strengthen sign-in

**principle**

- a user signs in by entering the current code from an enrolled authenticator
- the server accepts only if the code matches for the current time window
- the user may add or remove authenticators at any time.

**state**

a set of Authenticators with

    - an owner User
    - a sharedSecret String

**actions**

enrollAuthenticator (by: User): (auth: Authenticator, secret: String)

- **requires:** by exists

- **effects:** create an authenticator with owner = by and sharedSecret a freshly generated secret;
  return (auth, sharedSecret) for provisioning

deleteAuthenticator (by: User, auth: Authenticator)

- **requires:** auth exists and by = auth.owner

- **effects:** remove auth from Authenticators

verifyCode (user: User, code: Number): ()

- **requires:** there exists an authenticator for user whose TOTP(code, sharedSecret) matches for the current time window

- **effects:** no state change

</details>
<details>
<summary><b>Subtleties</b></summary>

- TOTP cuts password-reuse/credential-stuffing, but not real-time phishing. It’s a second factor and should be paired with password auth to be effective.
- verification is by time window which means that a valid code can be reused within that window.
</details>

### Concept 3:

<details>

<summary><b>BillableHours Concept</b></summary>

**concept** BillableHours [User, Project]

**purpose** record work sessions for client billing

**principle**

- an employee starts a session by choosing a project and entering a short description;
- later they end the session to record its duration;
- if a session is forgotten, it can be ended afterward with an explicit end time;
- sessions can be corrected before invoicing.

**state**

a set of Sessions with

- an owner User
- a Project
- a description String
- a startedAt DateTime
- an optional endedAt DateTime

**actions**

startSession (by: User, project: Project, description: String): (s: Session)

- **requires** by exists and there is no open session for by

- **effects:** create s with owner = by, Project = project, description = description, startedAt = now, endedAt missing; return s

endSession (by: User, s: Session)

- **requires:** s exists and by = s.owner and s.endedAt is missing

- **effects:** set s.endedAt = now

setSessionEnd (by: User, s: Session, newEnd: DateTime)

- **requires:** s exists and by = s.owner and newEnd > s.startedAt
  and (if s.endedAt is missing then newEnd ≤ now)
  and the interval [s.startedAt, newEnd] does not overlap any other session for by

- **effects:** set s.endedAt = newEnd

setSessionStart (by: User, s: Session, newStart: DateTime)

- **requires:** s exists and by = s.owner
  and (if s.endedAt exists then newStart < s.endedAt else newStart < now)
  and the interval [newStart, (s.endedAt if exists else now)] does not overlap any other session for by

- **effects:** set s.startedAt = newStart

updateSessionProject (by: User, s: Session, newProject: Project)

- **requires:** s exists and by = s.owner

- **effects:** set s.Project = newProject

updateSessionDescription (by: User, s: Session, newDescription: String)

- **requires:** s exists and by = s.owner

- **effects:** set s.description = newDescription

</details>
<details>
<summary><b>Subtleties</b></summary>

- one invariant: at most one open session per user, and no overlapping intervals; if a session has an end, then startedAt < endedAt.
- treat intervals as half-open [start, end), so back-to-back is fine (end == next start is allowed).
- timestamps are in UTC; now is server time; setSessionEnd can use any past time ≤ now, never a future time.
</details>
