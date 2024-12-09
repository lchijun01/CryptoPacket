### **Crypto Packet Mini App**

This comprehensive guide includes the complete code for the **Crypto Packet Mini App** with a smart contract, frontend, and backend. The app allows users to:

1. **Verify World ID** to create a wallet.
2. **Top-Up WLD** into their wallet.
3. **Create Packets** with QR codes or shareable links for recipients to claim WLD.
4. **Claim Packets** using QR codes or manual input.
5. View **Wallet Balance** and **Transaction History**.

---

### **1. Smart Contract**
**File:** `contracts/CryptoPacket.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CryptoPacket {
    struct Packet {
        address creator;
        uint256 totalAmount;
        uint256 remainingAmount;
        uint256 recipients;
        bool isRandom;
        mapping(address => bool) claimed;
    }

    mapping(address => uint256) public walletBalances;
    mapping(address => bool) public isWorldIDVerified;
    mapping(uint256 => Packet) public packets;
    uint256 public packetCounter;

    event PacketCreated(uint256 packetId, address creator, uint256 amount, uint256 recipients, bool isRandom);
    event PacketClaimed(uint256 packetId, address claimer, uint256 amount);
    event WalletToppedUp(address user, uint256 amount);

    modifier onlyVerifiedWorldID() {
        require(isWorldIDVerified[msg.sender], "World ID verification required");
        _;
    }

    function verifyWorldID(address user) external {
        require(!isWorldIDVerified[user], "World ID already verified");
        isWorldIDVerified[user] = true;
    }

    function topUpWallet() external payable {
        require(msg.value > 0, "Must send WLD to top up wallet");
        walletBalances[msg.sender] += msg.value;
        emit WalletToppedUp(msg.sender, msg.value);
    }

    function createPacket(uint256 amount, uint256 recipients, bool isRandom) external onlyVerifiedWorldID {
        require(walletBalances[msg.sender] >= amount, "Insufficient balance");
        require(amount >= 1 ether, "Minimum packet value is 1 WLD");
        require(recipients > 0 && recipients <= 100, "Recipients must be between 1 and 100");

        walletBalances[msg.sender] -= amount;

        Packet storage packet = packets[++packetCounter];
        packet.creator = msg.sender;
        packet.totalAmount = amount;
        packet.remainingAmount = amount;
        packet.recipients = recipients;
        packet.isRandom = isRandom;

        emit PacketCreated(packetCounter, msg.sender, amount, recipients, isRandom);
    }

    function claimPacket(uint256 packetId) external onlyVerifiedWorldID {
        Packet storage packet = packets[packetId];
        require(packet.remainingAmount > 0, "Packet already fully claimed");
        require(!packet.claimed[msg.sender], "You have already claimed from this packet");

        uint256 claimAmount;
        if (packet.isRandom) {
            claimAmount = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender))) % packet.remainingAmount;
            if (claimAmount == 0) claimAmount = 1 ether; // Minimum 1 WLD for random
        } else {
            claimAmount = packet.totalAmount / packet.recipients;
        }

        require(claimAmount <= packet.remainingAmount, "Claim amount exceeds remaining balance");
        packet.remainingAmount -= claimAmount;
        packet.claimed[msg.sender] = true;

        walletBalances[msg.sender] += claimAmount;

        emit PacketClaimed(packetId, msg.sender, claimAmount);
    }

    function getWalletBalance(address user) external view returns (uint256) {
        return walletBalances[user];
    }

    function getPacketDetails(uint256 packetId) external view returns (address, uint256, uint256, uint256, bool) {
        Packet storage packet = packets[packetId];
        return (
            packet.creator,
            packet.totalAmount,
            packet.remainingAmount,
            packet.recipients,
            packet.isRandom
        );
    }
}
```

---

### **2. Backend**
**File:** `server/index.js`
```javascript
import express from "express";
import bodyParser from "body-parser";
import fetch from "node-fetch";

const app = express();
app.use(bodyParser.json());

app.post("/api/verify", async (req, res) => {
  const { payload, action, signal } = req.body;

  try {
    const verifyResponse = await fetch("https://id.worldcoin.org/verify", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        action,
        signal,
        proof: payload.proof,
        nullifier_hash: payload.nullifier_hash,
        merkle_root: payload.merkle_root,
      }),
    });

    const verifyData = await verifyResponse.json();
    if (verifyData.success) {
      res.status(200).json({ status: 200, message: "Verification successful" });
    } else {
      res.status(400).json({ status: 400, message: "Verification failed" });
    }
  } catch (error) {
    console.error("Error:", error);
    res.status(500).json({ status: 500, message: "Internal server error" });
  }
});

app.listen(3000, () => console.log("Server running on port 3000"));
```

---

### **3. Frontend**
#### **Pages**

**`src/components/CreatePacket.js`:** Allows the user to create a packet.

```javascript
import React, { useState } from "react";
import { MiniKit, VerifyCommandInput, VerificationLevel } from "@worldcoin/minikit-js";
import { createPacketContract } from "../utils/minikit";

const CreatePacket = () => {
  const [amount, setAmount] = useState(0);
  const [recipients, setRecipients] = useState(0);
  const [isRandom, setIsRandom] = useState(false);

  const handleVerifyAndCreate = async () => {
    const verifyPayload = {
      action: "create-red-packet",
      signal: recipients.toString(),
      verification_level: VerificationLevel.Orb,
    };

    try {
      if (!MiniKit.isInstalled()) {
        alert("World App is not installed. Please install it to proceed.");
        return;
      }

      const { finalPayload } = await MiniKit.commandsAsync.verify(verifyPayload);
      if (finalPayload.status === "error") {
        alert("Verification failed. Please try again.");
        return;
      }

      await createPacketContract(amount, recipients, isRandom);
      alert("Packet created successfully!");
    } catch (error) {
      console.error("Error:", error);
      alert("An error occurred.");
    }
  };

  return (
    <div>
      <h1>Create Red Packet</h1>
      <input type="number" placeholder="Total Amount (ETH)" value={amount} onChange={(e) => setAmount(e.target.value)} />
      <input type="number" placeholder="Number of Recipients" value={recipients} onChange={(e) => setRecipients(e.target.value)} />
      <label>
        <input type="checkbox" checked={isRandom} onChange={(e) => setIsRandom(e.target.checked)} /> Random Distribution
      </label>
      <button onClick={handleVerifyAndCreate}>Verify & Create Packet</button>
    </div>
  );
};

export default CreatePacket;
```

**`src/components/ClaimPacket.js`:** Allows the user to claim a packet.

```javascript
import React, { useState } from "react";
import { claimPacketContract } from "../utils/minikit";

const ClaimPacket = () => {
  const [packetId, setPacketId] = useState("");

  const handleClaim = async () => {
    try {
      await claimPacketContract(packetId);
      alert("Claim successful!");
    } catch (error) {
      console.error("Error:", error);
      alert("Claim failed.");
    }
  };

  return (
    <div>
      <h1>Claim Packet</h1>
      <input
        type="text"
        placeholder="Enter Packet ID"
        value={packetId}
        onChange={(e) => setPacketId(e.target.value)}
      />
      <button onClick={handleClaim}>Claim Packet</button>
    </div>
  );
};

export default ClaimPacket;
```

**`src/components/WalletHistory.js`:** Displays the user’s wallet balance and transaction history.

```javascript
import React, { useEffect, useState } from "react";
import { getWalletBalance, getWalletHistory } from "../utils/minikit";

const WalletHistory = () => {
  const [balance, setBalance] = useState(0);
  const [history, setHistory] = useState([]);

  useEffect(() => {
    const fetchData = async () => {
      const walletBalance = await getWalletBalance();
      const walletHistory = await getWalletHistory();
      setBalance(walletBalance);
      setHistory(walletHistory);
    };

    fetchData();
  }, []);

  return (
    <div>
      <h1>Wallet History</h1>
      <p>Total Balance: {balance} WLD</p>
      <ul>
        {history.map((item, index) => (
          <li key={index}>{item}</li>
        ))}
      </ul>
    </div>
  );
};

export default WalletHistory;
```

**`src/utils/minikit.js`:** Handles smart contract interactions.

```
