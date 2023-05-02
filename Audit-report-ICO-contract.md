## Audit Results:

The code is very well written. No major issues were found, and the natspec comments were super helpful in understanding the code. The simplicity of the code made it easy to digest and audit.

# **[M-01]** - The loop in the constructor has no upper bound:
A loop without an upper bound may exceed the gas block limit and cause the deployment of the contract to fail.

```
   constructor(
        address _owner,
        address treasuryAddress,
        address[] memory _allowlist
    ) {
        spaceCoin = new SpaceCoin(_owner, address(this), treasuryAddress);
        owner = _owner;
        for (uint256 i; i < _allowlist.length; i++) {
            allowlist[_allowlist[i]] = true;
        }
    }
```

## Remediation Steps:
Try setting an upper bound on the loop to avoid this.

# **[L-01]** - The loop isn't checking input validation:
A user could pass 0 as an argument for the constructor, and since `i < allowlist.length`, this would cause the deployment to fail and waste gas. Consider putting in some input validation for this array.

## Remediation Steps:
Use input validation `require(array.length >= 0)` or use `i <= array.length`.

# **[G-01]** - i++ can be changed to ++i for gas savings and can also be put in an unchecked block:
Use `++i` instead of `i++`.

Also, `++i` can be unchecked:

```
    unchecked {
        ++i;
    }
```

# **[G-02]** - Cache array.length in loops for gas savings:
`allowlist.length` can be cached outside of the loop to save gas. This will avoid having to read it from memory every time the loop runs, significantly lowering gas costs.

## Remediation 

```
        uint length = _allowlist.length

        for (uint256 i; i < length; i++) {
            allowlist[_allowlist[i]] = true;
        }
```

# **[NC- 01]**:

```
   constructor(
        address _owner,
        address treasuryAddress,
        address[] memory _allowlist
    ) {
        spaceCoin = new SpaceCoin(_owner, address(this), treasuryAddress);
        owner = _owner;
        for (uint256 i; i < _allowlist.length; i++) {
            allowlist[_allowlist[i]] = true;
        }
    }
```
Technically, this loop can be removed from the constructor and put into its own function that is called when the contract is deployed. This will make it a bit easier to read and understand.