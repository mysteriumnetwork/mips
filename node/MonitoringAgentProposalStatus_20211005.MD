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

### States (with description)

#### How It Works

Introduce a new fields into Quality Oracle (QO) contract for _/sessions_ endpoint called: **monitoring_state** and **monitoring_state_description**. 
These fields will contain a more human-readable representation of the current **monitoring_failed** field.

|**monitoring_state**| **monitoring_state_description** | Can transition to | Show in Proposal List? |
| :---: | :---: | :---:| :---: |
| PENDING| Proposal is brand new, waiting for first check by MA | UNHEALTHY, HEALTHY| NO |
| UNHEALTHY| [Failed to send data, Failed to connect] | HEALTHY, UNSTABLE| NO |
| HEALTHY| "Anything about how AWESOME this node runner is" | UNHEALTHY, UNSTABLE| YES |
| UNSTABLE| [Last 2-3 attempts failed in a row, High RTT] | HEALTHY, UNHEALTHY| YES |

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