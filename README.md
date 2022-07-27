**Audius Governance Takeover Staking & Delegation Recovery**

The approach the Audius team would like to take to turning the Staking and Delegation systems of the Audius protocol back online is targeted and straightforward.

This requires prior understanding of the approach we took to address the original governance takeover. Postmortem [here](https://blog.audius.co/article/audius-governance-takeover-post-mortem-7-23-22).

**Current State**

All Audius Governance contracts have been reset to original state + 

1. The Staking contract logic is currently set to the BlockingContract

2. The DelegateManagerV2 contract logic is currently set to the BlockingContract

3. There exists two delegations on the system that delegate 10T $AUDIO from one address to itself. Analysis \[[script](https://gist.github.com/raymondjacobson/697441861b47a5849d617c40082b1717)]These are the two addresses in question.

   1. **0xbdbb5945f252bc3466a319cdcc3ee8056bf2e569**
   2. **0xa62c3ced6906b188a4d4a3c981b79f2aabf2107f**

4. The Audius team believes the scope of damage done to the system is either only static variables and the delegation to these wallets. The static variables changed are:

   1. Staking

      1. governanceAddress, stakingToken

   2. Delegation

      1. governanceAddress, serviceProviderFactoryAddress, undelegateLockupDuration, audiusToken

**Steps**

Diff: <https://github.com/AudiusProject/eth-contracts-takeover-fix/pull/3>

1. Roll out the Staking contract with one change using the same upgradeTo & initialize process to set the correct variables

   1. Remove the line \`stakingToken.safeTransfer(\_transferAccount, \_amount);\` in \_unstakeFor. This change is so that we can undelegate & unstake the attacker without erroring out.
   2. Invoke initialize and reset goveranceAddress and stakingToken to their correct values
   3. Disable all functions with require(false) except for ‘undelegateStakeFor’ 

2. Roll out a modified DelegateManagerV2 that has two additional features

   1. A \_requiresProxyAdmin check on **every** method that limits invocation to the controlled proxyAdmin.

   2. An additional method to prune (\_pruneAttacker) the state from those addresses. The prune state method will accomplish

      1. Ridding the system of the erroneous delegations
      2. Emitting the standard undelegation events so that our subgraphs & downstream processing recovers state correctly

This has one caveat in that the Checkpoint History will contain the delegation indefinitely, but we believe that will have no impact on the broader protocol function.

3. Invoke the initialize method on the DelegateManagerV2

   1. Calls \_pruneAttacker method on the two wallets in question via the proxyAdmin
   2. Reset governanceAddress, serviceProviderFactoryAddress, undelegateLockupDuration, audiusToken to their correct values

4. Redeploy the DelegateManagerV2 logic contract without the \_requiresNotAttacker and \_pruneAttacker methods. Storage is kept the same.

5. Redeploy the Staking contract without the line omission.

6. Verify

**Testing**

Flow was tested end to end on a ganache fork at head
Attacker state appears to be reset using our scripts from above.

**Implications**

- Subgraph indexing (downstream) – Event emission should reconcile state, but we should re-examine.

  
  
  


**FULL ORIGINAL POST-MORTEM TIMELINE**

22:54:37 UTC - [First attempt by attacker to call initialize on Audius contracts](https://ethtx.info/mainnet/0x3bbb15f9852c389e8d77399fe88b49b042d0f22aad4a33c979fbabc60a34b24f/), This transaction delegates 10T $AUDIO internally to the staking contract (no circulating supply change or token impact). This transaction fails because no votes are cast on the proposal.

23:10:12 UTC - A second transaction is executed that delegates another 10T $AUDIO.** **Circulating supply is unaffected again, but the proposal does pass because the 10T $AUDIO is used to cast erroneous votes. <https://etherscan.io/tx/0xfefd829e246002a8fd061eede7501bccb6e244a9aacea0ebceaecef5d877a984>

23:11:36 UTC - Suspicious token transfer occurs on-chain as a result (<https://etherscan.io/tx/0x4227bca8ed4b8915c7eec0e14ad3748a88c4371d4176e716e8007249b9980dc9>)

23:35:00 UTC - Audius project team receives report of transfer, response team assembled

00:00:00 UTC - [Samczsun](https://twitter.com/samczsun) joins the response team to help pick apart what happened

00:25:00 UTC - Tweet ([Community: lost guardian](https://twitter.com/napgener/status/1551000568029642753))

00:26:00 UTC - Statement reporting exploit ([Audius: acknowledgement](https://twitter.com/AudiusProject/status/1551000725169180672))

00:30:00 UTC - Response team finds root cause, realizes that exploit is still actively exploitable

00:38:00 UTC - Response team begins development on a contract upgrade to mitigate exploit

01:19:00 UTC - Response team temporarily disables [Protocol Dashboard](https://dashboard.audius.co)

01:29:00 UTC - Exploit mitigation contract completed

01:50:00 UTC - Exploit mitigation fully tested in local environment

01:57:37 UTC - Deployed initial fix ([etherscan](https://etherscan.io/tx/0x13347615a94b2e3ad385277f5145102f50fe112f274b0e5300c6d8ce507eeb80), [blockscout](https://blockscout.com/eth/mainnet/tx/0x13347615a94b2e3ad385277f5145102f50fe112f274b0e5300c6d8ce507eeb80), [tenderly](https://dashboard.tenderly.co/tx/mainnet/0x13347615a94b2e3ad385277f5145102f50fe112f274b0e5300c6d8ce507eeb80)) to patch exploit, freezing currently deployed contracts (including token) as a side effect.

02:09:00 UTC - Tweet ([Audius: acknowledgement fix](https://twitter.com/AudiusProject/status/1551026771838914560))

03:33:00 UTC - Post Mortem started to collect information

**04:42:16 UTC**** Upgraded Contract Patch Deployed to TrustedNotifierManager: (0x6f08105c8CEef2BC5653640fcdbBE1e7bb519D39): **<https://etherscan.io/tx/0x4addea66274252d42fdef43798d5c926e1186ee7778dc7d0bf3d1e79ed8d711e>

05:47:00 UTC - Talks to upgrade registry contract are underway.

06:46:00 UTC - Confirmed that governance was not upgraded by attacker (storage is intact)

**07:06:21 UTC Upgraded Contract Patch Deployed to Governance: (0x4DEcA517D6817B6510798b7328F2314d3003AbAC)**

<https://etherscan.io/tx/0x5aafd3675b47110d12ae9306e1d9a9a5c5a442df4084de1680866a90d35f4def>

**08:26:51 UTC Upgraded Contract Patch Deployed to Token: (0x18aAA7115705e8be94bfFEBDE57Af9BFc265B998)**

<https://etherscan.io/tx/0x1eaeb997ce7baed0a906a1f3194835441821f89e129c718c7374f336350202cc>

**08:26:51 UTC Token is unfrozen**

09:05:00 UTC - Tweet ([Audius: issue resolved](https://twitter.com/AudiusProject/status/1551131385515020288))

**15:36:00 UTC ****Upgraded Contract Patch Deployed to ServiceTypeManager: (0x9EfB0f4F38aFbb4b0984D00C126E97E21b8417C5)******<https://etherscan.io/tx/0x1eaeb997ce7baed0a906a1f3194835441821f89e129c718c7374f336350202cc>

**15:49:55 UTC Upgraded Contract Patch Deployed to ****ServiceProviderFactory: (0xD17A9bc90c582249e211a4f4b16721e7f65156c8)**<https://etherscan.io/tx/0xa5eb014e99e6b9c14f49a424f00514ae019015fcc7b7fcda638ad74d6535a614>

**17:33:24 UTC Upgraded Contract Patch Deployed to ****ClaimsManager: (0x44617F9dCEd9787C3B06a05B35B4C779a2AA1334)**<https://etherscan.io/tx/0xc1e4cda438478f08f3ffb2e0a7a6c6287787914e9e380860e6ba4f92d7518983>

**17:52:05 UTC Upgraded Contract Patch Deployed to EthRewardsManager: (0x5aa6B99A2B461bA8E97207740f0A689C5C39C3b0)**

<https://etherscan.io/tx/0xaa9194a05eb9a092b8364169799d05161ea61e7f3830feed72d694ac51a91c57>

**17:33:24 UTC Upgraded Contract Patch Deployed to ****WormholeClient: (0x6E7a1F7339bbB62b23D44797b63e4258d283E095)**

**17:33:24 UTC Upgraded Contract Patch Deployed to ****Registry: (0xd976d3b4f4e22a238c1A736b6612D22f17b6f64C)**

<https://etherscan.io/tx/0x0ecb955e18ac8c27ee44b72708a95cf40a238397e2c68306e6763d4b19c8c7c7>

Remaining Remediation Steps \[As of writing 7/24/22 22:26 UTC]

**xx:yy:zz UTC ****Upgraded Contract Patch Deployed to**** Staking: (0xe6D97B2099F142513be7A2a068bE040656Ae4591)**

**xx:yy:zz UTC ****Upgraded Contract Patch Deployed to ****DelegateManager: (0x4d7968ebfD390D5E7926Cb3587C39eFf2F9FB225)**

xx:yy:zz UTC - Re-enable Protocol Dashboard
