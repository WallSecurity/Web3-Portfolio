# **Santas List Security Review**

## **Table of  Contents**

 - [Information about the review](#info)
 - [High Findings](#high)
 - [Low Findings](#low)

---

<!-- headings -->
<a id="info"></a>

## **Information about the review**
| Auditor | Start Date | End Date | Highs | Mediums | Lows |
|---|---|---|---|---|---|
| 0xWallSecurity | 01/15/2024 | 01/15/2024 | 2 | 0 | 1 |

---

<a id="high"></a>

## **High findings**

### [H-1] Missing access control allows blocking any address from receiving a present

### Status
| Date | Status | 
|---|---|
|  N/A | N/A |

### Description
The `SantasList::checkList` function is missing the `onlySanta` modifier, allowing anyone to set arbitrary addresses to *naughty*, which blocks them from receiving their christmas present.

### Proof of Concept
Create a new address `attacker` and run the following test in foundry:

<details><summary>Code</summary>

```solidity
function testCheckListAsNonSanta() public {
    vm.prank(attacker);
    santasList.checkList(user, SantasList.Status.NAUGHTY);
    assertEq(uint256(santasList.getNaughtyOrNiceOnce(user)), uint256(SantasList.Status.NAUGHTY));
}
```

</details>

### Mitigation
Add the `onlySanta` modifier to the `SantasList::checkList` function.

---

### [H-2] `santasList::buyPresent` burns the receivers token and sends the NFT to the sender.

### Status
| Date | Status | 
|---|---|
|  N/A | N/A |

### Description
`santasList::buyPresent(address presentReceiver)` uses the parameter `presentReceiver` to call `i_santaToken.burn(presentReceiver)`, which burns the token from the `presentReceiver` address as seen in the code below in `santaToken::burn`:

```solidity
function burn(address from) external {
  if (msg.sender != i_santasList) {
      revert SantaToken__NotSantasList();
  }
  _burn(from, 1e18);
}
```

Furthermore, the resulting NFT is minted to the *sender* address, which gifted the NFT by calling `santasList::_mindAndIncrement` to the `msg.sender`:

```solidity
function _mintAndIncrement() private {
    _safeMint(msg.sender, s_tokenCounter++);
}

function mint(address to) external {
  if (msg.sender != i_santasList) {
      revert SantaToken__NotSantasList();
  }
  _mint(to, 1e18);
}
```

Lastly, the `santasList::buyPresent` is callable by anyone, but should only be callable by *naughty* people. This present also only costs 1 token, as opposed to the 2 tokens stated in the contracts invariant, 

```solidity
// The cost of santa tokens for naughty people to buy presents
uint256 public constant PURCHASED_PRESENT_COST = 2e18;
```

### Mitigation
Add a new error `SantasList__NotNaughty()` and change the following code:

```diff
      function buyPresent(address presentReceiver) external {
+          if (s_theListCheckedOnce[msg.sender] != Status.NAUGHTY {
+              revert SantasList__NotNaughty();
+          }
-          i_santaToken.burn(presentReceiver);
+          i_santaToken.burn(msg.sender);
-          _mintAndIncrement();
+          _mintAndIncrement(presentReceiver);
      }
      function _mintAndIncrement(address receiver) private {
-          _safeMint(msg.sender, s_tokenCounter++);
+          _safeMint(receiver, s_tokenCounter++);
      }
```

---

<a id="low"></a>

## **Low findings**

### [L-1] NFT can be minted after christmas.

### Status
| Date | Status | 
|---|---|
|  MM/DD/YYYY | Status |

### Description
the `santasList::collectPresent` function only checks, if it is *too early* to claim the christmas present, but not if it is *too late*.

```solidity
function collectPresent() external {
  if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
      revert SantasList__NotChristmasYet();
}
```
