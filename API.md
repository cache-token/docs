# CACHE B2B API Documentation

CACHE intends to make available API's for B2B customer's to build infrastructure experiences.
CACHE mainly allows for the allocation of Gold Bullion. The API's are split into this core functionality and further
supportive API's used for building B2C applications are also available under the CACHE Whitelabel category. 

## Base URL
We will open up the base url to B2B customer's. Interested developers can reach out to ```contact@cache.gold```
For a quick reference please see ```https://api-prod.gramchain.net/api-docs/``` do note that it may not contain all the api's listed here.

## Open Endpoints - Explorer API's and Authentication

Open endpoints require no Authentication.

* [Explore CACHE Holdings](https://api-prod.gramchain.net/api-docs/) : `POST  <baseurl>/api/public/GetTreeByGroup`
* [Login](https://api-prod.gramchain.net/identity/login) : `POST <baseurl>/identity/login?signin=`


## Endpoints that require Authentication

Permissioned endpoints require a valid Token to be included in the header of the
request. A Token can be acquired from the Login view above.

### Bar Allocation related

Endpoint allows for the deposit, reservation, redemption, proof of reserve verification(smart contract), fees accrued etc:

* [Deposit]() : `GET /api/Secured/GramChainInsertViaWeb`
* [Reserve a particular bar]() : ``
* [Remove the reservation a particular bar]() : ``
* [Verify the reservation status of bars by address]() : ``
* [Mint new tokens]() : ``
* [Redeem a bar]() : ``
* [Check fees accrued]() : ``
* [Pay accrued fees]() : ``

#### Proof of Reserve Verification
In order to verify onchain the proof of reserve before doing any application the following smart contract can be used - 
Proof Of Reserve Contract address ```0x5586bf404c7a22a4a4077401272ce5945f80189c```
```
// contracts/CLv1.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract CLv1 {
    IERC20 public WBTC;
    AggregatorV3Interface public PoR_CGT;

    function setup(IERC20 _CGT, AggregatorV3Interface _porCGT) public {
        require(
            address(CGT) == address(0x0),
            "setup() already executed"
        );
        CGT = _CGT;
        PoR_CGT = _porCGT;
    }

    function checkLatestRoundDataPoRCGT()
        public
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        )
    {
        (roundId, answer, startedAt, updatedAt, answeredInRound) = PoR_CGT
            .latestRoundData();
    }

    function verifyPoRCGT() external view returns (bool result) {
        (, int256 reserveProof, , , ) = checkLatestRoundDataPoRCGT();
        if (reserveProof >= int256(CGT.totalSupply())) {
            return result = true;
        }
    }
}
```
### CACHE Whitelabel

Endpoints for requesting and managing the whitelabel product

* [Request for whitelabel]() : `POST /`
* [Check status of request]() : `GET /`
* [Creat An Account]() : `POST /api/accounts/`
* [Show An Account and Balances]() : `GET /api/accounts/:pk/`
* [Update An Account]() : `PUT /api/accounts/:pk/`
* [Delete An Account]() : `DELETE /api/accounts/:pk/`
* [Update KYC for an Account]() : `PUT /api/accounts/:pk/kyc`
* [ADD External wallet for an Account]() : `PUT /api/accounts/:pk/wallet`

### Trading API's
* [Buy Spot]() : `POST /`
* [Sell Spot]() : `POST /`
* [Buy Limit]() : `POST /`
* [Sell Limit]() : `POST /`


### B2B Account & Gnosis Safe Multisig Management
These are admin api's and can only be created / modified/ deleted via an admin action

* [Create API Key]() : `POST /api/admin/`
* [Create wallet]() : `POST /api/admin/`
* [Delete wallet]() : `POST /api/admin/`
* [Create a new outgoing tx]() : `POST /api/admin/`


### Testnet faucet - 
Please send a request to contact@cache.gold, if you would like some test CGT.