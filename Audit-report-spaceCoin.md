## Audit Summary
Overall the code is simple and does what as is intended. The overriding of the `transfer` function is simple to understand and is throughly tested.

Code coverage is above standard.

## **[M-01]** - Using `ether` to denominate an ERC20 token 
Using Ether instead of using `10 ** 18` is a high severity issue. It can cause the contract to not actually mint the values set in the constructor.

This would cause the contract to be unusable and have to lead to a redeployment.

Line 65.

```
 constructor(
        address _owner,
        address icoAddress,
        address _treasuryAddress
    ) ERC20("SpaceCoin", "SPC") {
        owner = _owner;
        treasuryAddress = _treasuryAddress;

        _mint(icoAddress, 150_000 ether);
        _mint(_treasuryAddress, 350_000 ether);
    }
```


