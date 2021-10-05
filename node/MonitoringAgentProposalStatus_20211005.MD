# Monitoring Agent - Proposal Status

## Motivation

Somewhere at Mysterium Network lives a Monitoring Agent (MA). Its duty is to do scheduled checks on all node runner proposals in order to verify proposal connectability.
Currently, this is represented by a bool flag called **monitoring_failed** with 2 possible values: true or false. It turns out - this is not ideal.  

Never before registered proposals will be marked as **monitoring_failed: false** which means that it is not reachable. MA can’t instantly check all brand new proposals (supposedly because it would be expensive to scale it) so instead MA checks each proposal individually and this takes time. At the time of writing - it takes approximately 4 hours for MA to check (~3160) nodes.  

Depending on your luck (as a node runner) you might want to jump into node running business and right off the bat spend4 hours figuring out why your node status reads as failed (marked monitoring_failed: false). Thus bugging Mysterium Network support staff.  

Adding insult to injury, many API consumers down the line (desktop app, mobile app etc..) may filter out such proposals for better user experience which may also make it harder to debug or test the actual availability of your proposals and perhaps hinder your ability to provide an exit node service.  

Finally, MA seemingly doesn't mark the proposal as **monitoring_failed: false** on the first failed attempt. It only does that after it was not able to connect to the specific node within 24h window.  

Bottom line is - bool representation of a complex state does not convey the meaning and adds confusion.

## Solution

In my opinion a much better approach is to have human-readable states. MA knows what is actually going on and so should the proposal API consumers.  
First we would introduce a new field in the contract. Let’s call that field: **monitoring_state**

### States (only)

This solution would only deal with proposal monitoring states.

#### How It Works

Introduce a new field into Quality Oracle (QO) contract for _/sessions_ endpoint called: **monitoring_state**. This field will contain a more human-readable representation of the current **monitoring_failed** field.

|**monitoring_state**| Description | Can transition to |
| :--- | :---: | ---:|
| INITIAL_CHECK_PENDING| Proposal is brand new, waiting for first check by MA | UNHEALTHY, HEALTHY|
| UNHEALTHY| Proposal has connection issues (for last 24h proposal has been less than 50% reachable by MA) | HEALTHY, UNSTABLE|
| HEALTHY| Proposal is healthy (for last 24h it has been >75% reachable by MA) | UNHEALTHY, UNSTABLE|
| UNSTABLE| Proposal is unstable (for last 24h it has been reachable 50%-75% of the time in MA checks) | HEALTHY, UNHEALTHY|

This state machine is not final but rather serves as an idea of how much more we would be able to communicate by moving to documented states.

### States (with description)

This solution builds on top of _"States (only)"_ and adds detailed description to response.

#### How It Works

In addition, while the “States (only)” approach gives a lot of information it could give it some more. Instead of having just **monitoring_state** and documented Description we could have the description of the state in the contract. So if we add a complimentary field to a contract that I will call **monitoring_state_description** we then would be able to pass it to the consumers as a human-readable verbose explanation.

|**monitoring_state**| **monitoring_state_description** | Can transition to |
| :--- | :---: | ---:|
| INITIAL_CHECK_PENDING| Proposal is brand new, waiting for first check by MA | UNHEALTHY, HEALTHY|
| UNHEALTHY| Proposal has connection issues (for last 24h proposal has been less than 50% reachable by MA) | HEALTHY, UNSTABLE|
| HEALTHY| Proposal is healthy (for last 24h it has been >75% reachable by MA) | UNHEALTHY, UNSTABLE|
| UNSTABLE| Proposal is unstable (for last 24h it has been reachable 50%-75% of the time in MA checks) | HEALTHY, UNHEALTHY|

Potential drawback of “_States (with description)_” might be additional data. While including extra text averaging at say 100 Bytes with 3160 proposals results in ~316Kb (will use the Kilo of 1000) for 1M proposals that would be ~100Mb.

## Advantages
- Allows scaling the state machine if we wish to cover an additional case.
- Human-readable
- Self documenting
- More accurate
- Easier to debug (self debug on the node runner side)
- Support would get fewer distractions in the long run
- Solves problem with proposal list polluted by unchecked proposals, which were not visited by MA yet.

## Disadvantages
- Additional data
- Risk of complicated state machine

## Closing notes

Solutions proposed here are not to be treated as final. I may not have yet thought this through in an hour. Rather please find it as grounds for discussions, improvements and healthy critique.