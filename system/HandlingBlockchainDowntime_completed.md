# Handling blockchain downtime (Completed)

2021-10-06 was a hard day for Mysterium while running in testnet3. The polygon mumbai blockchain maintainers decided to do a release to validators, which in turn broke the whole mumbai testnet. Since we were running on mumbai at that time, none of our services worked. Nodes failed to start, the running nodes could not establish/maintain sessions as Hermes would encounter issues when checking the channel status for both providers and consumers on chain.

## Learning from mistakes of the past

To prevent such issues, a few fixes need to be made:

## The proposed fixes

### Node

Node failed to start due to the fact that they could not lookup hermes address(URL) via blockchain. This could be solved by adding a fallback to the hermes url for known hermeses directly in node.

However, this is unlikely to be a sufficient step to solve the startup issues as I believe there are many more further checks that happen on startup that rely on blockchain.

From memory, some further lookups that might be optional are:

1) On session start, provider checks the hermes fee via blockchain. This can easily be cached for reasonably long times if it already is not.
2) On session start, provider checks the hermes status via blockchain. This call should probably be remade in a way where a failure to call blockchain is not treated as critical.

All of them can likely be done via Hermes + Transactor instead of blockchain as long as those two services are alive. 

Ideally, one would add full node URLs to ones' hosts file and point them to localhost, try to start node and not rest until all the issues are solved.

### Hermes

Hermes has the capability to work without a blockchain as a checkpoint to a somewhat limited extent. Currently, Hermes only performs lookups on chain on a periodic basis. The results are then stored and used from the database instead of blockchain. However, should a consumer or provider run out of balance for the operation they are trying to execute a force-sync with BC is issued.

When the blockchain is offline, eventually, all the data in the database is marked as stale and no further exchanges can occur as any fetch of the data will try to hit blockchain, which, in the case of this study, was unavailable. However, having a longer caching period would have helped as the sessions for consumers with enough balance would still have been maintained.

Currently, to switch the flag you'd need to redeploy. Firstly, you need to figure out that this would have helped, which I only did today.

So the proposal is as such - add the ability to switch the caching period easily. One could even call it a `blockchain_down_use_db_exclusively` feature toggle. Such toggle would need to be enabled via the universe and hermeses should look it up dynamically to detect if it is changed. Such flag could set the cache expiry to something like a day, which would mean that sessions could happily run for long periods of time without blockchain.

This, however, should not become the normal running mode for hermes, as if the blockchain is alive and well a short cache period is required to ensure that massive cheating does not occur. 

To take things further, hermes could determine that such situation is occuring automatically and switch the flag on itself. However, this needs to be visible as such flag should be switched off once the blockchain is back online.

### Misc notes

Keep in mind that new consumers/providers will still have a hard time when the blockchain is offline and that's a topic for a separate discussion. This would also not solve issues for users who have small amounts of balance left before the blockchain downtime as they would be unable to topup and hermes would still refuse them.
