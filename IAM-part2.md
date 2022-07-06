# Identity and Access Management part 2

## Task 1: Fine-grained access

The simple role-based authorization approach that we have used in the previous lab may not always be enough due to its limited flexibility.
With that approach, where access is checked like that:

```
fn canEdit(grant) {
    return grant.hasRole('editor')
}
```

application is coupled with a concrete set of roles that manages access to a set of resources.

If you think of a scenario, where for a concrete user we would want to forbid access to some specific resource - there is no way of doing that without modification of the original code.

In order to support such cases, we may want to check access to a specific resource like that:

```
fn canEdit(grant, requestedResource) {
    return grant.hasAccess(requestedResource, 'edit')
}
```

This approach increases the level of flexibility and decreasing the coupling - more specific rules can be defined at the side of IAM system. If there are some unexpected business requirements regarding access control, we will not need to introduce additional roles that would lead to modification of the application’s code.

But surely it brings complexity - rules becomes more complicated. Also, amount of resources that we now should manage separately may be very large. All that information about permissions cannot be possibly included into the access token. Thus, we have two options: perform a live verification by invoking authorization server, or include only some subset of permissions inside access token.


## Task 1.1: Create corresponding clients and enable Authorization services

- app-trainer - It displays data taken from api-trainer. You have already configured it in the previous lab, but for this lab make sure that user will have readonly role for api-pokemon in token claims.

- api-trainer - Resource server that stores data about trainer’s pokemon decks.
    - POST /deck-create/:deckname
        - Creates a new empty trainer’s deck inside in-memory store with a given name.
        - Also creates necessary resources in Keycloak, if they do not exist.
        - Returns created deck.
    - GET /deck-list
        - Returns all user decks from in-memory store.
    - GET /deck/:username/:deckname
        - Returns concrete deck.
    - PATCH /deck/:username/:deckname
        - Updates pokemons for concrete deck.
        - Returns concrete deck.
        - Body is a JSON array of pokemon names with length <= 6.
    - DELETE /deck/:username/:deckname
        - Deletes concrete deck.

Authorization services are required only for the resource server.

Application operates with the following resources and scopes:

- resources:
    - deck_create
        - represents deck creation capabilities only
        - should be created manually
    - deck_list|${username}
        - represents list of decks of a particular user
        - created automatically
    - deck|${username}|${name}
        - represents concrete deck of a particular user
        - created automatically

- scopes (all scopes should be created manually):
    - deck_create:create
    - deck_list:read
    - deck:read
    - deck:write
    - deck:delete

-----------------------

### Implementation:

First create the "api-trainer" clinet.

![](https://i.imgur.com/M1VOsn9.png)


Especially I had to make this part on (to enable the authorization for this resource):

![](https://i.imgur.com/n5Ox8c1.png)

Next step is to create the scopes you told:

![](https://i.imgur.com/fTunw84.png)

Then creating the "deck_create" resource (the others in this list created automatically when users tried to add something):

![](https://i.imgur.com/TgqrfvY.png)

![](https://i.imgur.com/LZsAJqh.png)

------------------

## Task 1.2: Implement authorization rules

Prepare (you can skip it from the report):

- Launch Keycloak from previous lab
- Launch sample app from previous lab
- Launch sample service from current lab
    - installation instructions inside iam-lab-part2.zip
    - do not forget to change config in keycloak.json with config from Installation section
    - note that app uses in-memory store - on restart data is wiped (automatically created resource records in keycloak still persist)

Create necessary policies and permissions for rules:

- any user with role trainer within api-trainer can create deck
- only owner can read deck_list
- only owner can access deck (read, write, delete)

After that you should be able to add/change/delete decks in the app-trainer.

**Hint**
For debugging purposes, you can use Evaluation tool.

**Hint**
The “only owner” policy can be implemented as a JS type policy like that:
```
var permission = $evaluation.getPermission();
var identity = $evaluation.getContext().getIdentity();
var resource = permission.getResource();
if (resource) {
    if (resource.getOwner().equals(identity.getId())) {
        $evaluation.grant();
    }
}
```
---------

### Implementation:

From the keycloak documentation we know that:

> Policies define the conditions that must be satisfied before granting access to an object.


lets add some policies (I will show each of them one by one lately):

![](https://i.imgur.com/mYNjF2L.png)

And from the permission definition:

> A permission associates the object being protected and the policies that must be evaluated to decide whether access should be granted.


Lets add permissions:

![](https://i.imgur.com/ZULTpnP.png)

The detail of each policy and permission associated with what you asked:

- Trainer policy and permission


![](https://i.imgur.com/8T78uJO.png)

![](https://i.imgur.com/yJxGsWg.png)


- Only owner reading the deck-list:


![](https://i.imgur.com/GYOK8cT.png)

![](https://i.imgur.com/AEsHtqi.png)

- Only owner access deck (read, write, delete):

![](https://i.imgur.com/mr0XgtE.png)

![](https://i.imgur.com/oymsXCU.png)

Then the configuration was successful:


![](https://i.imgur.com/2HPGnf2.png)

![](https://i.imgur.com/e3o9xlf.png)

![](https://i.imgur.com/0teRUa7.png)



## Task 2: User-Managed Access

User-Managed Access allows users to be in control of their resources and to decide what to share and to whom.

Keycloak provides built-in account console
/realms/as_lab/account/#/resources

Use account console to demonstrate how it works:

- share one of the user decks in Account Console and from another user in app-trainer.

-----------

### Implementation:

First we need to enable this feature:


![](https://i.imgur.com/2XrbLug.png)

Then I create another user and give it "trainer" role.

![](https://i.imgur.com/FCz5ZY4.png)

Opening the user1's resources list:

![](https://i.imgur.com/jdfQxKt.png)

Sharing with user2:

![](https://i.imgur.com/uoIYnCO.png)


Then user2 can read and add : 

![](https://i.imgur.com/N3MW2Iq.png)

-----------
![](https://i.imgur.com/tIojoPR.png)

-----------

![](https://i.imgur.com/3DfatmM.png)

Also we can see this shared deck in the user console

![](https://i.imgur.com/KH4mHfY.png)

On the other hand, we can see the user2's newly created deck:

![](https://i.imgur.com/xzfU9MA.png)


Share it with user1:


![](https://i.imgur.com/GGTgP94.png)

![](https://i.imgur.com/lVhUkET.png)


![](https://i.imgur.com/6fKyqHY.png)


---------

## Task 3: Audit

As you should know, Access Control based on 3 things: Authentication, Authorization and Audit. You already got familiar with the first two.
Now, learn about Keycloak audit capabilities and briefly demonstrate some of them that may be handful in incident response.

---------

### Implementation:
 
 Keycloak includes:
 - Recording login events: It can record all the users' login actions for example incorrect password, successful login, ..
 - Recording admin events: All admin's actions that are performed in the Admin Console.



Enabling saving events:


![](https://i.imgur.com/IH3QKwU.png)

![](https://i.imgur.com/xUcBunh.png)


Admin events enabling:

![](https://i.imgur.com/NnShrYc.png)

![](https://i.imgur.com/y7dtVnB.png)

