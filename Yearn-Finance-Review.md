# YEARN FINANCE REVIEW


## INTRODUUCTION

Yearn finance is a multi chain yeild aggregator. It uses vaults and strategies to help users maximize yeild on their assets. It utilizes strategies such as staking, auto-compounding and other methods to help users maximize yield while reducing gas cost.

A vault collects a token (refered to as "want") which the user deposits, it then uses these tokens to generate yield for the depositors. It splits this want token and asign certain amount to the best performing strategies.

A vault can be made up of up to 20 strategies. Each strategy is alloted a certain amount of the want tokens deposited in the vault which it uses to generate yield. Users who deposit funds to a yearn vault recieve a Yearn vault token which acts as a reciept of their share in the vault.

At construction time, the address of the vault of which the strategy belongs will be passed in. The deployer will be set as the strategist, rewards and keeper.

The base strategy contract is a contract that strategies are to implement. It is not a strategy implementation contract and as such does not contain much function logic.

The vault contract is the user interfacing contract, the user cannot interact directly with the strategy contracts. Only authorized people can interact with the strategies contract directly to manage the positions the strategy is handling.

![Userflow of a yearn user](./User%20flow.png)

## IMPLEMENTED INTERFACES

* A strategy implements the StrategyAPI interface to allows the vault to interact with it seamlessly while still enabling it perform its function and maintaining some security standards. A strategy implements the interface below. 
```
interface StrategyAPI {
    function name() external view returns (string memory);
    function vault() external view returns (address);
    function want() external view returns (address);
    function apiVersion() external pure returns (string memory);
    function keeper() external view returns (address);
    function isActive() external view returns (bool);
    function delegatedAssets() external view returns (uint256);
    function estimatedTotalAssets() external view returns (uint256);
    function tendTrigger(uint256 callCost) external view returns (bool);
    function tend() external;
    function harvestTrigger(uint256 callCost) external view returns (bool);
    function harvest() external;
    event Harvested(uint256 profit, uint256 loss, uint256 debtPayment, uint256 debtOutstanding);
}
```

* The StrategyParams struct is what enables the vault keep track of the strategies under it. The StrategyParams struct consists of the following:
```
struct StrategyParams {
    uint256 performanceFee;
    uint256 activation;
    uint256 debtRatio;
    uint256 minDebtPerHarvest;
    uint256 maxDebtPerHarvest;
    uint256 lastReport;
    uint256 totalDebt;
    uint256 totalGain;
    uint256 totalLoss;
}
```


* A vault implements the VaultAPI interface which allows it to keep track of the strategies under it and their performance. A vault implements the interface below
```
interface VaultAPI is IERC20 {
    function name() external view returns (string calldata);
    function symbol() external view returns (string calldata);
    function decimals() external view returns (uint256);
    function apiVersion() external pure returns (string memory);
    function permit(
        address owner,
        address spender,
        uint256 amount,
        uint256 expiry,
        bytes calldata signature
    ) external returns (bool);
    // NOTE: Vyper produces multiple signatures for a given function with "default" args
    function deposit() external returns (uint256);
    function deposit(uint256 amount) external returns (uint256);
    function deposit(uint256 amount, address recipient) external returns (uint256);
    // NOTE: Vyper produces multiple signatures for a given function with "default" args
    function withdraw() external returns (uint256);
    function withdraw(uint256 maxShares) external returns (uint256);
    function withdraw(uint256 maxShares, address recipient) external returns (uint256);
    function token() external view returns (address);
    function strategies(address _strategy) external view returns (StrategyParams memory);
    function pricePerShare() external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function depositLimit() external view returns (uint256);
    function maxAvailableShares() external view returns (uint256);
    function creditAvailable() external view returns (uint256);
    function debtOutstanding() external view returns (uint256);
    function expectedReturn() external view returns (uint256);
    function report(
        uint256 _gain,
        uint256 _loss,
        uint256 _debtPayment
    ) external returns (uint256);
    function revokeStrategy() external;
    function governance() external view returns (address);
    function management() external view returns (address);
    function guardian() external view returns (address);
}
```


* The health check interface as seen below
```
interface HealthCheck {
    function check(
        uint256 profit,
        uint256 loss,
        uint256 debtPayment,
        uint256 debtOutstanding,
        uint256 totalDebt
    ) external view returns (bool);
}
```



## FUNCTIONS

The BaseStrategy contract is meant to be inherited by strategy contracts. Of the various functions in the base strategy contract, some of them need to be implemented by the inheriting contract while others have been implemented in the BaseStrategy contract.


### IMPLEMENTED FUNCTIONS

* **apiVersion**: Returns the version of the strategy and can be modified.
* **delegatedAssets**: Returns the delegated assets and can be overridden

* **AUTHORIZATION FUNCTIONS**: These are internal modifier function that allows only authorized people call a function. They should not be modified.
    * _onlyAuthorized
    * _onlyEmergencyAuthorized
    * _onlyStrategist
    * _onlyGovernance
    * _onlyRewarder
    * _onlyKeepers
    * _onlyVaultManagers

* **_initialize**: This function is used to initialize the strategy.

* **setRewards**: Used to change the address that recieves the rewards

* **setForceHarvestTriggerOnce**: This function is used to set a trigger to force harvest of yield.

* **setCreditThreshold**: Used to set the credit threshold of a strategy 

* **sweep**: The sweep function is used to rescue tokens that are not used by the strategy but might have been sent to the strategy by mistake. These tokens are sent to the governance address. This helps to prevent total token loss or burn from not being able to be retrieved. The vault share token and strategy want token cannot be swept.

* **setEmergencyExit**: This function sets a strategys emergency exit value to true and revokes the strategy from the vault if the strategy has a debt ratio(ie if money has been assigned to the strategy for yield generation)

* **withdraw**: This function is used to withdraw waant token from a strategy, it can only be called by the vault. This funds is liquidated from positions in action and the leftover reinvested at the next harvest or tend.

* **tend** : This function doesnt take profit when called but adjusts the position of the strategy for better profit

* **tendTrigger**: This function provides a signal to the keeper to call tend function using the estimated gas cost passed in by the keeper to know if calling tend is worth it.

* **harvest**: This function is used to harvest a strategy. The strategy reports its status to the vault when this function is called.

* **setMetadataURI**: This function is used to set the metadata URI of the strategy. The strategys metadata cotains information about the strategy.

* **isActive**: This function is used to know is a strategy is still active or retired.

* **setBaseFeeOracle**: This function is used to set the base fee oracle. This oracle is used to retrieve the base network fee in other to know if it is optimal to call the tend or harvest functions.

* **setStrategist**: This function is used to set the strategist address.

* **setKeeper**: This function is used to set the keeper address.

* **governance**: This function returns the vault governance address.

* **harvestTrigger**: This function signals the keeper to call the harvest function.

* **isBaseFeeAcceptable**: Used to check if the base network fee is okay for the harvest trigger or harvest function to be called.

* **migrate**: This function is used to migrate the want tokens in the strategy contract to the new strategy contract.

* **setMaxReportDelay**: Used to set the maximum blocks that should pass before harvest can be called.

* **setMinReportDelay**: Used to set the minimum blocks that should pass before harvest can be called.

* **setDoHealthCheck**: Used to signal if a health check should be carried out.

* **setHealthCheck**




### FUNCTIONS TO BE IMPLEMENTED

* **name**: Returns the name of the strategy

* **ethToWant**: This function is used to convert the tokens passed in as ETH to the decimal of the want token.

* **protectedTokens**: The protectedTokens function is used to return an array of token addresses that is  used by the strategy for yield generation. The vault share token and strategy want token should not be included as that has been taken care of already in the sweep function.

* **prepareMigration**: The prepareMigration function is used to prepare other tokens to be migrated from the current strategy address to the new strategy address.

* **liquidateAllPositions**: This function is particularly used in the case of an emergency exit. It frees up as much assets as it can for the vault

* **liquidatePosition**: This is used to make availiable a certain amount of token to the vault contract from the strategy.

* **estimatedTotalAssets**: This function returns the total assets this strategy is managing which is inclusive of capital and returns.

* **prepareReturn**: This is used to prepare a strategy for harvest in normal conditions. It should minimize loss and maximize profit.

* **adjustPosition**(to be implemented): This function is used to adjust a strategys position based on the amount that has been allocated to the strategy. Any free capital in the strategy can be reinvested.

other functions include restrictive functions like modifier functions and functions to get the associated vaults parameter in the strategy with ease.



## OBSERVATIONS

* In the Base strategy contract line 565:
```
// with USDC/ETH = 1800, this should give back 1800000000 (180 USDC)
```

should be

```
with ETH/USDC = 1800, this should give back 1800000000 (1800 USDC)
```

* Recieve function: A recieve function should be implemented in the base strategy contract. Although it is expected that the strategy contract implements it, it would be much safer if the base strategy contract implements it. This would allow the contract recieve ETH especially if it uses it in its strategy.

* Withdraw function: A withdraw function should be implemented in the base strategy contract. Although it is expected that the strategy contract implements it, it would be much safer if the base strategy contract implements it. This would prevent ETH sent to the contract by mistake to be rescued.




## SUGGESTIONS

* EPNS can be integrated to alert the keeper on when they should call the tend or harvest function.