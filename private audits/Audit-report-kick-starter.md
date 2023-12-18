## **[M-01]** - Mint function doesn't operate as intended and NFTs can get stuck in the contract

- Currently the mint function is peforming some arthimetic to try and figure out what the users NFT mint count should be based on how many NFTs they've already minted.

`uint amountToMint = (userContributions[msg.sender] / 1 ether) - mintedByContributor[msg.sender];`

However if a user withdraws before they mint their NFTs, their `userContributions[msg.sender]` will be 0 and the user will not be able to mint any NFTs.

The tech specs say that even if a user withdraws they should be allowed to mint NFTs, meaning this implementation does not meet the technical specifications.

Line `87`.

```
    function mint() public {
        uint amountToMint = (userContributions[msg.sender] / 1 ether) - mintedByContributor[msg.sender];

        mintedByContributor[msg.sender] += amountToMint;
        for (uint i = 0; i < amountToMint; i++) {
            _safeMint(msg.sender, tokenId++);
        }
```

## Remediation Steps

1. Try to have a different mapping that keeps track of users deposits/mints. That way if a user withdraws and sets `userContribution[msg.sender]` to 0 you can still keep track of how many NFTs a user is owed seperately.

Line `87`.

```
    mapping(address => uint) deposits;

    function mint() public {
        uint amountToMint = deposits[msg.sender] - mintedByContributor[msg.sender];

        mintedByContributor[msg.sender] += amountToMint;
        for (uint i = 0; i < amountToMint; i++) {
            _safeMint(msg.sender, tokenId++);
        }
```


## **[G-01]** - Variables can be set to their default values in order to save gas. `bool` default value is `false` and `uint` default value is `0`.

```
    
    bool public isCompleted = false;
    bool public isCancelled = false;
    uint public tokenId = 0;

```

## **[G-02]** - Storing the projects address in array may not be necesary and is costly to the user when calling ProjectFactory.

Since you are already emitting and event that has the project address in it, it my be easier to get the value from your events for your tests. This way you can get rid of the array, getter function, and pushing to an array everytime you call the factory to create a new contract.

Line `5`.

```
contract ProjectFactory {
    event ProjectCreated(address newProject, address creator, uint goal, string name, string symbol);
    Project[] public projects;

    function getProjectsLength() public view returns (uint256) {
        return projects.length;
    }

    function create(uint goal, string memory name, string memory symbol) external {
        Project project = new Project(goal, name, symbol, msg.sender);
        projects.push(project);
        emit ProjectCreated(address(project), msg.sender, goal, name, symbol); 
    }
}
```
## **[G-03]** - Using `++i` is cheaper than using `i++`.

Line `87`.

```
    function mint() public {
        uint amountToMint = userContributions[msg.sender] - mintedByContributor[msg.sender];

        mintedByContributor[msg.sender] += amountToMint;
        for (uint i = 0; i < amountToMint; ++i) {
            _safeMint(msg.sender, tokenId++);
        }
```

## **[G-04]** - Setting `uint i = 0` is uneccesary and can save gas.

Writing just `uint i` will set it equal to the default value and save gas during deployment.

Line `87`.


```
    function mint() public {
        uint amountToMint = userContributions[msg.sender] - mintedByContributor[msg.sender];

        mintedByContributor[msg.sender] += amountToMint;
        for (uint i = 0; i < amountToMint; ++i) {
            _safeMint(msg.sender, tokenId++);
        }
```