pragma solidity ^0.4.26;

import "./SafeMath.sol";

contract Ownable {
    address public owner;

    constructor() public {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only the owner can call this");
        _;
    }

    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Invalid new owner address");
        owner = newOwner;
    }
}

contract ERC20Basic {
    uint256 public totalSupply;

    function balanceOf(address who) public view returns (uint256);
    function transfer(address to, uint256 value) public;
    event Transfer(address indexed from, address indexed to, uint256 value);
}

contract ERC20 is ERC20Basic {
    function allowance(address owner, address spender) public view returns (uint256);
    function transferFrom(address from, address to, uint256 value) public;
    function approve(address spender, uint256 value) public;
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

contract BasicToken is ERC20Basic, Ownable {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    uint256 public basisPointsRate = 0;
    uint256 public maximumFee = 0;

    modifier onlyPayloadSize(uint256 size) {
        require(!(msg.data.length < size + 4), "Payload size is too small");
        _;
    }

    function transfer(address to, uint256 value) public onlyPayloadSize(2 * 32) {
        require(to != address(0), "Invalid recipient address");
        uint256 fee = value.mul(basisPointsRate).div(10000);
        if (fee > maximumFee) {
            fee = maximumFee;
        }
        uint256 sendAmount = value.sub(fee);
        balances[msg.sender] = balances[msg.sender].sub(value);
        balances[to] = balances[to].add(sendAmount);
        if (fee > 0) {
            balances[owner] = balances[owner].add(fee);
            emit Transfer(msg.sender, owner, fee);
        }
        emit Transfer(msg.sender, to, sendAmount);
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}

contract StandardToken is BasicToken, ERC20 {
    mapping(address => mapping(address => uint256)) public allowed;

    uint256 public constant MAX_UINT = 2**256 - 1;

    function transferFrom(address from, address to, uint256 value) public onlyPayloadSize(3 * 32) {
        uint256 _allowance = allowed[from][msg.sender];
        require(to != address(0), "Invalid recipient address");

        uint256 fee = value.mul(basisPointsRate).div(10000);
        if (fee > maximumFee) {
            fee = maximumFee;
        }
        if (_allowance < MAX_UINT) {
            allowed[from][msg.sender] = _allowance.sub(value);
        }
        uint256 sendAmount = value.sub(fee);
        balances[from] = balances[from].sub(value);
        balances[to] = balances[to].add(sendAmount);
        if (fee > 0) {
            balances[owner] = balances[owner].add(fee);
            emit Transfer(from, owner, fee);
        }
        emit Transfer(from, to, sendAmount);
    }

    function approve(address spender, uint256 value) public onlyPayloadSize(2 * 32) {
        require(spender != address(0), "Invalid spender address");

        if (value != 0 && allowed[msg.sender][spender] != 0) {
            revert("Approval already exists");
        }

        allowed[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
    }

    function allowance(address _owner, address spender) public view returns (uint256 remaining) {
        return allowed[_owner][spender];
    }
}

contract Pausable is Ownable {
    event Pause();
    event Unpause();

    bool public paused = false;

    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }

    modifier whenPaused() {
        require(paused, "Contract is not paused");
        _;
    }

    function pause() public onlyOwner whenNotPaused {
        paused = true;
        emit Pause();
    }

    function unpause() public onlyOwner whenPaused {
        paused = false;
        emit Unpause();
    }
}

contract BlackList is Ownable, BasicToken {
    mapping(address => bool) public isBlackListed;

    function addBlackList(address _evilUser) public onlyOwner {
        isBlackListed[_evilUser] = true;
        emit AddedBlackList(_evilUser);
    }

    function removeBlackList(address _clearedUser) public onlyOwner {
        isBlackListed[_clearedUser] = false;
        emit RemovedBlackList(_clearedUser);
    }

    function destroyBlackFunds(address _blackListedUser) public onlyOwner {
        require(isBlackListed[_blackListedUser], "Address is not blacklisted");
        uint256 dirtyFunds = balances[_blackListedUser];
        balances[_blackListedUser] = 0;
        totalSupply = totalSupply.sub(dirtyFunds);
        emit DestroyedBlackFunds(_blackListedUser, dirtyFunds);
    }

    event DestroyedBlackFunds(address indexed _blackListedUser, uint256 _balance);
    event AddedBlackList(address indexed _user);
    event RemovedBlackList(address indexed _user);
}

contract UpgradedStandardToken is StandardToken {
    function transferByLegacy(address from, address to, uint256 value) public;
    function transferFromByLegacy(address sender, address from, address spender, uint256 value) public;
    function approveByLegacy(address from, address spender, uint256 value) public;
}

contract BCGameUSDToken is Pausable, StandardToken, BlackList {
    string public name;
    string public symbol;
    uint8 public decimals;
    address public upgradedAddress;
    bool public deprecated;

    constructor(uint256 _initialSupply, string memory _name, string memory _symbol, uint8 _decimals) public {
        totalSupply = _initialSupply;
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        balances[msg.sender] = _initialSupply;
        deprecated = false;
    }

    function transfer(address _to, uint256 _value) public whenNotPaused {
        require(!isBlackListed[msg.sender], "Sender is blacklisted");
        if (deprecated) {
            UpgradedStandardToken(upgradedAddress).transferByLegacy(msg.sender, _to, _value);
        } else {
            super.transfer(_to, _value);
        }
    }

    function transferFrom(address _from, address _to, uint256 _value) public whenNotPaused {
        require(!isBlackListed[_from], "Sender is blacklisted");
        if (deprecated) {
            UpgradedStandardToken(upgradedAddress).transferFromByLegacy(msg.sender, _from, _to, _value);
        } else {
            super.transferFrom(_from, _to, _value);
        }
    }

    function approve(address _spender, uint256 _value) public onlyPayloadSize(2 * 32) {
        require(_spender != address(0), "Invalid spender address");
        if (deprecated) {
            UpgradedStandardToken(upgradedAddress).approveByLegacy(msg.sender, _spender, _value);
        } else {
            super.approve(_spender, _value);
        }
    }

    function totalSupply() public view returns (uint256) {
        if (deprecated) {
            return UpgradedStandardToken(upgradedAddress).totalSupply();
        } else {
            return totalSupply;
        }
    }

    function deprecate(address _upgradedAddress) public onlyOwner {
        deprecated = true;
        upgradedAddress = _upgradedAddress;
        emit Deprecate(_upgradedAddress);
    }

    event Deprecate(address newAddress);
}


