# swap
solidity swap
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract AtomicSwap {
    struct Swap {
        address sender;
        address receiver;
        uint256 amount;
        address token;
        bytes32 hashLock;
        uint256 timelock;
        bool claimed;
    }

    mapping(bytes32 => Swap) public swaps;

    event SwapInitiated(address indexed sender, address indexed receiver, uint256 amount, address indexed token, bytes32 hashLock, uint256 timelock);
    event SwapClaimed(address indexed sender, address indexed receiver, uint256 amount, address indexed token, bytes32 preimage);

    function initiateSwap(address _receiver, uint256 _amount, address _token, bytes32 _hashLock, uint256 _timelock) external {
        require(_amount > 0, "Amount must be greater than zero");
        require(_timelock > block.timestamp, "Timelock must be in the future");

        IERC20 token = IERC20(_token);
        require(token.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

        bytes32 swapId = keccak256(abi.encodePacked(msg.sender, _receiver, _amount, _token, _hashLock, _timelock));
        swaps[swapId] = Swap(msg.sender, _receiver, _amount, _token, _hashLock, _timelock, false);

        emit SwapInitiated(msg.sender, _receiver, _amount, _token, _hashLock, _timelock);
    }

    function claimSwap(bytes32 _preimage) external {
        bytes32 hashLock = keccak256(abi.encodePacked(_preimage));
        bytes32 swapId = keccak256(abi.encodePacked(msg.sender, hashLock));
        Swap storage swap = swaps[swapId];

        require(swap.sender != address(0), "Swap not initiated");
        require(swap.receiver == msg.sender, "You are not the intended receiver");
        require(!swap.claimed, "Swap already claimed");
        require(block.timestamp >= swap.timelock, "Timelock not expired");
        require(hashLock == swap.hashLock, "Invalid preimage");

        IERC20 token = IERC20(swap.token);
        require(token.transfer(msg.sender, swap.amount), "Transfer failed");

        swap.claimed = true;

        emit SwapClaimed(swap.sender, swap.receiver, swap.amount, swap.token, _preimage);
    }
}
