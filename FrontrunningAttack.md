# Front-running Attack (Simplified Explanation)

Front-running happens when someone takes advantage of seeing a pending transaction in the blockchain's public mempool and submits their own transaction with a higher gas fee to get it executed first. This allows them to profit or gain an advantage.

---

### Example: Front-running Issue in Code

Here’s a vulnerable smart contract example where someone can front-run:

#### Vulnerable Code
```solidity
// Vulnerable smart contract
pragma solidity ^0.8.0;

contract Vulnerable {
    address public owner;
    uint public bidAmount;

    constructor() {
        owner = msg.sender;
        bidAmount = 0;
    }

    // Function to place a bid
    function placeBid() external payable {
        require(msg.value > bidAmount, "Bid is too low");

        // Refund the previous bidder
        if (bidAmount > 0) {
            payable(owner).transfer(bidAmount);
        }

        // Update the bid
        owner = msg.sender;
        bidAmount = msg.value;
    }
}
```

---

#### Problem:
1. **Front-running Scenario:** 
   - User A submits a bid of 1 ETH.
   - A malicious User B sees this transaction in the mempool and submits a higher bid (e.g., 2 ETH) with a higher gas fee.
   - User B’s transaction gets mined first, making User A’s bid ineffective.

---

### Protected Code Example

To protect against front-running, use **commit-reveal schemes** where users commit their bids first, then reveal them later. This prevents attackers from seeing bid values.

#### Protected Code
```solidity
// Secure smart contract using commit-reveal
pragma solidity ^0.8.0;

contract Protected {
    struct Bid {
        bytes32 hashedBid;
        uint256 deposit;
    }

    mapping(address => Bid) public bids;
    address public highestBidder;
    uint public highestBid;

    // Step 1: Commit phase
    function commitBid(bytes32 _hashedBid) external payable {
        require(msg.value > 0, "Deposit is required");

        bids[msg.sender] = Bid({
            hashedBid: _hashedBid,
            deposit: msg.value
        });
    }

    // Step 2: Reveal phase
    function revealBid(uint256 _bid, string memory _secret) external {
        Bid storage userBid = bids[msg.sender];

        require(userBid.deposit > 0, "No bid found");
        require(
            userBid.hashedBid == keccak256(abi.encodePacked(_bid, _secret)),
            "Invalid reveal"
        );

        // Check if the bid is the highest
        if (_bid > highestBid) {
            highestBid = _bid;
            highestBidder = msg.sender;
        }
    }
}
```

---

### How This Solves the Problem:
1. In the **commit phase**, users submit a hashed bid (a combination of their bid and a secret value).
2. In the **reveal phase**, users reveal their actual bid and secret.
3. Since the actual bid is hidden during the commit phase, attackers cannot front-run.

---

This approach ensures fairness and prevents attackers from exploiting pending transactions.





Here’s an explanation of the **four questions from Solodit’s checklist** for understanding **front-running vulnerabilities** in Solidity with examples. I’ll also include an example for each to help you as an auditor.

---

### **1. Are there measures in place to prevent frontrunning vulnerabilities in get-or-create patterns?**

#### **Description**
The **get-or-create pattern** checks if a resource exists and creates it if it doesn’t. This is prone to frontrunning because:
- An attacker can see the "create" transaction and submit their own with higher gas, taking ownership of the resource before the legitimate user.

#### **Example**
**Vulnerable Code:**
```solidity
mapping(string => address) public users;

function getOrCreateUser(string memory username) external {
    require(users[username] == address(0), "Already taken");
    users[username] = msg.sender; // Assign user
}
```

**Attack Scenario:**
- A user tries to register "Alice."
- An attacker frontruns and registers "Alice" before the user.

#### **Fix:**
Implement a **reservation mechanism**:
```solidity
mapping(string => address) public reservations;
mapping(string => address) public users;

function reserveUsername(string memory username) external {
    require(reservations[username] == address(0), "Already reserved");
    reservations[username] = msg.sender;
}

function finalizeUsername(string memory username) external {
    require(reservations[username] == msg.sender, "You did not reserve this");
    users[username] = msg.sender;
    delete reservations[username]; // Clear reservation
}
```

**Auditor Tip:**
Check if sensitive actions like resource creation are protected against frontrunning. Look for **reservations** or **ownership checks**.

---

### **2. Are two-transaction actions designed to be safe from frontrunning?**

#### **Description**
Some actions require **two steps (two transactions)**, such as depositing tokens and withdrawing rewards. If these steps aren’t linked securely, attackers can frontrun between the two steps.

#### **Example**
**Vulnerable Code:**
```solidity
uint256 public poolBalance;

function deposit(uint256 amount) external {
    poolBalance += amount; // User deposits funds
}

function withdraw(uint256 amount) external {
    require(poolBalance >= amount, "Insufficient balance");
    poolBalance -= amount;
    payable(msg.sender).transfer(amount);
}
```

**Attack Scenario:**
- A user deposits funds.
- An attacker sees the transaction and frontruns to withdraw the funds before the deposit is finalized.

#### **Fix:**
Use a **user-specific balance** instead of a shared pool:
```solidity
mapping(address => uint256) public balances;

function deposit() external payable {
    balances[msg.sender] += msg.value; // Track user balance
}

function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    balances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);
}
```

**Auditor Tip:**
Ensure the two steps are **tied to the user** or implement **locks** to prevent other users from interfering.

---

### **3. Can users maliciously cause others' transactions to revert by preempting with dust?**

#### **Description**
Attackers can send **small, irrelevant (dust)** transactions to manipulate the state, causing legitimate transactions to fail. This happens when:
- Contracts don’t handle small (non-material) values correctly.
- The attacker uses dust to increase gas costs or force reverts.

#### **Example**
**Vulnerable Code:**
```solidity
mapping(address => uint256) public balances;

function deposit() external payable {
    balances[msg.sender] += msg.value; // Add deposited amount
}

function withdraw() external {
    uint256 amount = balances[msg.sender];
    require(amount > 0, "No balance");
    balances[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

**Attack Scenario:**
- An attacker sends a dust deposit (e.g., 1 wei).
- The victim tries to withdraw, but gas costs to transfer small amounts are higher, causing the transaction to fail.

#### **Fix:**
Add a **minimum threshold** to ignore dust:
```solidity
uint256 public constant MIN_DEPOSIT = 0.01 ether;

function deposit() external payable {
    require(msg.value >= MIN_DEPOSIT, "Deposit too small");
    balances[msg.sender] += msg.value;
}
```

**Auditor Tip:**
Look for **minimum thresholds** in deposits or actions. Ensure dust can’t disrupt legitimate transactions.

---

### **4. Does the protocol need a commit-reveal scheme?**

#### **Description**
Without a **commit-reveal scheme**, sensitive data like votes or bids are visible in the mempool. Attackers can:
- See your vote or bid before it is mined.
- Copy or manipulate it to gain an advantage.

#### **Example**
**Vulnerable Code:**
```solidity
function bid(uint256 amount) external {
    // Store the bid directly
    bids[msg.sender] = amount;
}
```

**Attack Scenario:**
- A user bids 10 tokens in an auction.
- An attacker frontruns with a bid of 11 tokens to outbid the user.

#### **Fix:**
Use a **commit-reveal scheme**:
1. **Commit Phase:** Users submit a hash of their bid + secret.
2. **Reveal Phase:** Users reveal the actual bid and secret.

```solidity
mapping(address => bytes32) public commitments;

function commitBid(bytes32 hash) external {
    commitments[msg.sender] = hash; // Save the hash
}

function revealBid(uint256 amount, bytes32 secret) external {
    require(commitments[msg.sender] == keccak256(abi.encodePacked(amount, secret)), "Invalid reveal");
    // Process the bid
}
```

**Auditor Tip:**
If sensitive actions (e.g., voting, bidding) are in the contract, ensure a **commit-reveal** mechanism is used to protect user intent.
