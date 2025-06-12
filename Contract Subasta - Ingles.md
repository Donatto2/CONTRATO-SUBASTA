# Smart Auction Contract

This repository contains the source code and documentation for a smart auction contract developed in Solidity. This contract implements a transparent auction system with automatic refunds for non-winning bidders, incorporating features such as time extensions on late bids, minimum bid increments, and secure fund management.

## Table of Contents

- [Key Features](#key-features)
- [Contract Structure](#contract-structure)
  - [State Variables](#state-variables)
  - [Structs](#structs)
  - [Events](#events)
  - [Modifiers](#modifiers)
  - [Constructor](#constructor)
  - [Main Functions](#main-functions)
  - [View Functions](#view-functions)
- [Security Considerations](#security-considerations)
- [Deployment and Usage](#deployment-and-usage)
- [License](#license)

## Key Features

The auction contract implements the following key features:

*   **Time Extensions on Late Bids:** If a bid is placed near the end of the auction, the deadline is automatically extended to allow for fair competition.
*   **Minimum Bid Increments:** New bids must exceed the current highest bid by at least 5%.
*   **2% Commission:** A 2% commission is applied to winning bids and refunds to non-winning bidders.
*   **Secure Withdrawal Patterns:** Implements secure mechanisms for fund withdrawal.
*   **Automatic Refunds:** Non-winning bidders receive an automatic refund of their funds (minus commission).
*   **Excess Funds Withdrawal:** Bidders can withdraw the amount of their bid that exceeds the minimum required to maintain their position, if their bid is no longer the highest.

## Contract Structure

The `Auction.sol` contract is structured as follows:

### State Variables

State variables store the persistent information of the contract on the blockchain. The most relevant variables are detailed below:

```solidity
    address private owner;
    uint private deadline;
    uint private constant commissionPercent = 2; // 2% commission
    uint private constant timeExtension = 10 minutes;
    bool private ended;
    Bid[] private bidHistory;
    Bid private highestBid;
    mapping(address => uint) private accumulatedBids;
    mapping(address => uint[]) private userBidHistory;
    mapping(address => uint) public pendingWithdrawals;
```

*   `owner`: The address of the contract owner, who has administrative privileges.
*   `deadline`: Timestamp indicating when the auction bidding period ends.
*   `commissionPercent`: A constant defining the commission percentage taken on bids (2% in this case).
*   `timeExtension`: A constant defining the duration in seconds (10 minutes) added to the auction deadline if a last-minute bid is placed.
*   `ended`: A boolean indicator that is `true` once the auction has ended, preventing further state changes.
*   `bidHistory`: An array that stores the complete history of all valid bids received, in chronological order.
*   `highestBid`: A `Bid` struct representing the current highest valid bid in the auction.
*   `accumulatedBids`: A mapping that tracks the total accumulated bid amount for each bidder address.
*   `userBidHistory`: A mapping that stores the history of all bid amounts made by each user address.
*   `pendingWithdrawals`: A mapping that tracks refundable amounts (98% of bids) for each bidder address.

### Structs

Structs allow you to define custom data types and group related variables. In this contract, the following struct is used:

```solidity
    struct Bid {
        address bidder;
        uint amount;
    }
```

*   `Bid`: Represents a single bid in the auction system, containing:
    *   `bidder`: The address of the bidder who placed this bid.
    *   `amount`: The amount of ETH bid (in wei).

### Events

Events are a way to log actions on the blockchain, allowing external applications (such as user interfaces or monitoring services) to react to contract state changes. The events defined in this contract are:

```solidity
    event NewBid(address indexed bidder, uint amount);
    event ExcessFundsAvailable(address indexed bidder, uint amount);
    event AuctionEnded(address winner, uint winningAmount);
    event PartialRefund(address indexed bidder, uint amount);
    event DepositWithdrawn(address indexed user, uint refundAmount, uint fee);
    event EmergencyWithdraw(address indexed receiver, uint amount);
    event DepositRefunded(address indexed bidder, uint refundAmount, uint fee);
```

*   `NewBid`: Emitted when a new bid is placed. Contains the bidder's address and the bid amount.
*   `ExcessFundsAvailable`: Emitted when a bidder places a bid that creates excess funds, available for withdrawal.
*   `AuctionEnded`: Emitted when the auction concludes. Reports the winner and the winning bid amount.
*   `PartialRefund`: Emitted when a bidder withdraws excess funds from their bid.
*   `DepositWithdrawn`: Emitted when a user withdraws their deposits (refunds).
*   `EmergencyWithdraw`: Emitted when contract funds are withdrawn under emergency by the owner.
*   `DepositRefunded`: Emitted when non-winning bids are automatically refunded.

### Modifiers

Modifiers are reusable code snippets that validate conditions before executing a function. This improves readability and reduces code duplication. The modifiers used are:

```solidity
    modifier onlyBeforeEnd() {
        require(block.timestamp < deadline, "Auction has ended");
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
```

*   `onlyBeforeEnd()`: Restricts function execution only if the auction has not yet ended.
*   `onlyOwner()`: Restricts function execution only to the contract owner.

### Constructor

The constructor is a special function that runs only once when the contract is deployed on the blockchain. It is responsible for initializing the essential state variables for the auction's operation.

```solidity
    constructor(uint _durationSeconds) {
        owner = msg.sender;
        deadline = block.timestamp + _durationSeconds;
    }
```

The constructor takes one parameter:

*   `_durationSeconds`: The duration of the auction in seconds from the time of deployment.

It initializes the contract owner and calculates the auction deadline.

### Main Functions

These functions are the core of the auction's logic, allowing users to interact with the contract to place bids, finalize the auction, and manage their funds.

```solidity
    function placeBid() external payable onlyBeforeEnd {
        // ... (function code)
    }

    function endAuction() external onlyOwner {
        // ... (function code)
    }

    function claimRefund() external returns (bool success) {
        // ... (function code)
    }

    function withdrawExcess() external onlyBeforeEnd returns (bool success) {
        // ... (function code)
    }

    function emergencyWithdraw() external onlyOwner {
        // ... (function code)
    }
```

*   `placeBid()`: Allows any participant to send Ether to place a bid. It validates that the bid meets the minimum increment (5% more than the current highest bid) and extends the auction if placed within the last 10 minutes. Emits `NewBid` and `ExcessFundsAvailable` if applicable.
*   `endAuction()`: Allows the owner to finalize the auction once the deadline has passed. It marks the auction as ended, transfers the winning bid (minus 2% commission) to the owner, and automatically refunds 98% to non-winning bidders. Emits `AuctionEnded` and `DepositRefunded`.
*   `claimRefund()`: Allows bidders to withdraw their refundable deposits once the auction has ended. Emits `DepositWithdrawn`.
*   `withdrawExcess()`: Allows bidders to withdraw the amount of their bid that exceeds the minimum required to maintain their position, if their bid is no longer the highest. Emits `PartialRefund`.
*   `emergencyWithdraw()`: Allows the owner to withdraw all contract funds in an emergency situation, once the auction has ended. Emits `EmergencyWithdraw`.

### View Functions

These functions allow any user to query the current state of the auction without modifying its state on the blockchain (`view` functions).

```solidity
    function getHighestBid() external view returns (address bidder, uint amount);
    function getBidHistory() external view returns (Bid[] memory);
    function timeRemaining() external view returns (uint remainingTime);
    function bidsOf(address user) external view returns (uint[] memory userBids);
    function getDeadline() external view returns (uint auctionDeadline);
    function getOwner() external view returns (address contractOwner);
    function isEnded() external view returns (bool endedStatus);
    function totalBidOf(address user) external view returns (uint totalBidAmount);
```

*   `getHighestBid()`: Returns the address of the current highest bidder and the amount of that bid.
*   `getBidHistory()`: Returns the complete history of all valid bids recorded in the auction.
*   `timeRemaining()`: Returns the remaining time in seconds until the auction ends (0 if already ended).
*   `bidsOf(address user)`: Returns an array with the amounts of all bids made by a specific user.
*   `getDeadline()`: Returns the timestamp of the auction deadline.
*   `getOwner()`: Returns the address of the contract owner.
*   `isEnded()`: Returns `true` if the auction has ended and `false` otherwise.
*   `totalBidOf(address user)`: Returns the total accumulated amount of all bids made by a specific user.

## Security Considerations

The contract has been designed with the following security considerations in mind:

*   **Checks-Effects-Interactions Pattern:** Ether transfers are performed after internal contract states are updated to prevent reentrancy attacks.
*   **Access Control:** The use of the `onlyOwner` modifier ensures that only the owner can execute critical functions.
*   **Robust Validations:** `require` statements are used extensively to validate function inputs and state conditions, thus preventing unexpected behavior and errors.
*   **Error Handling:** Clear and specific error messages are provided to guide users on unmet conditions.
*   **`block.timestamp` Dependence:** Although `block.timestamp` is used for time-based logic, it is acknowledged that miners can slightly manipulate this value. For high-security auctions, decentralized time oracles could be considered.

## Deployment and Usage

To deploy and use this contract, you will need a Solidity development environment (such as Truffle, Hardhat, or Remix) and an Ethereum network (or a testnet like Sepolia, Goerli, etc.).

1.  **Compilation:** Compile the `Auction.sol` contract using the Solidity compiler (version `0.8.20` or higher).
2.  **Deployment:** Deploy the contract to your chosen network, providing the constructor parameter (`_durationSeconds`).
3.  **Interaction:** Once deployed, you can interact with the contract by calling its functions through a user interface (DApp) or directly from a development console.

**Interaction Example (Pseudocode):**

```javascript
// Assuming \'auction\' is an instance of the deployed contract

// Place a bid
auction.placeBid({ value: web3.utils.toWei(\'0.5\', \'ether\') });

// Get the highest bid
const [bidder, amount] = await auction.getHighestBid();
console.log(`Highest bidder: ${bidder}, Highest bid: ${web3.utils.fromWei(amount, \'ether\')} ETH`);

// End the auction (owner only)
auction.endAuction();

// Withdraw excess funds
auction.withdrawExcess();

// Claim refund (if not the winner)
auction.claimRefund();
```

## License

This project is licensed under the MIT License. See the `LICENSE` file for more details.


