# Horse-Racing
Horse Racing.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract BaseHorseRacing is ERC721, Ownable {
    uint256 public nextTokenId = 1;
    uint256 public constant MAX_HORSES_PER_PLAYER = 8;
    uint256 public constant RACE_ENTRY_FEE = 0.0005 ether;

    struct Horse {
        uint8 speed;
        uint8 stamina;
        uint8 spirit;
        uint256 racesWon;
    }

    mapping(uint256 => Horse) public horses;
    mapping(address => uint256) public playerWins;

    uint256 public prizePool;
    uint256 public currentRaceId;
    mapping(uint256 => address[]) public raceParticipants;
    mapping(uint256 => uint256[]) public raceHorseIds;

    event HorseMinted(address owner, uint256 tokenId);
    event RaceJoined(uint256 raceId, uint256 horseId);
    event RaceFinished(uint256 raceId, uint256 firstPlace, uint256 secondPlace, uint256 thirdPlace);
    event PrizeClaimed(address winner, uint256 amount);

    constructor() ERC721("Base Horse", "BHR") Ownable(msg.sender) {}

    function mintHorse() external {
        require(balanceOf(msg.sender) < MAX_HORSES_PER_PLAYER, "Max 8 horses per player");
        
        uint256 tokenId = nextTokenId++;
        _mint(msg.sender, tokenId);

        horses[tokenId] = Horse({
            speed: uint8(10 + (tokenId % 15)),
            stamina: uint8(8 + ((tokenId * 3) % 12)),
            spirit: uint8(9 + ((tokenId * 7) % 11)),
            racesWon: 0
        });

        emit HorseMinted(msg.sender, tokenId);
    }

    function joinRace(uint256 horseId) external payable {
        require(ownerOf(horseId) == msg.sender, "Not your horse");
        require(msg.value == RACE_ENTRY_FEE, "Exact 0.0005 ETH required");

        prizePool += msg.value;

        currentRaceId++;
        raceParticipants[currentRaceId].push(msg.sender);
        raceHorseIds[currentRaceId].push(horseId);

        emit RaceJoined(currentRaceId, horseId);
    }

    function finishRace(uint256 raceId) external {
        require(raceHorseIds[raceId].length >= 2, "Not enough participants");

        uint256[] memory horseList = raceHorseIds[raceId];
        address[] memory playerList = raceParticipants[raceId];

        uint256[] memory scores = new uint256[](horseList.length);
        uint256 first = 0;
        uint256 second = 0;
        uint256 third = 0;

        for (uint i = 0; i < horseList.length; i++) {
            Horse memory h = horses[horseList[i]];
            uint256 random = uint256(keccak256(abi.encodePacked(
                block.timestamp, raceId, i, msg.sender
            ))) % 50;

            scores[i] = uint256(h.speed) * 3 + uint256(h.stamina) * 2 + uint256(h.spirit) + random;

            if (scores[i] > scores[first]) {
                third = second;
                second = first;
                first = i;
            } else if (scores[i] > scores[second]) {
                third = second;
                second = i;
            } else if (scores[i] > scores[third]) {
                third = i;
            }
        }

        uint256 totalPrize = prizePool * 70 / 100;
        prizePool -= totalPrize;

        if (totalPrize > 0) {
            uint256 firstPrize = totalPrize * 60 / 100;
            uint256 secondPrize = totalPrize * 25 / 100;
            uint256 thirdPrize = totalPrize * 15 / 100;

            if (firstPrize > 0) {
                (bool success, ) = payable(playerList[first]).call{value: firstPrize}("");
                if (success) {
                    horses[horseList[first]].racesWon++;
                }
            }
            if (secondPrize > 0 && horseList.length > 1) {
                (bool success, ) = payable(playerList[second]).call{value: secondPrize}("");
            }
            if (thirdPrize > 0 && horseList.length > 2) {
                (bool success, ) = payable(playerList[third]).call{value: thirdPrize}("");
            }
        }

        emit RaceFinished(raceId, horseList[first], horseList[second], horseList[third]);

        delete raceParticipants[raceId];
        delete raceHorseIds[raceId];
    }

    function getHorse(uint256 tokenId) external view returns (Horse memory) {
        return horses[tokenId];
    }

    function withdraw() external onlyOwner {
        (bool success, ) = payable(owner()).call{value: address(this).balance}("");
        require(success, "Withdraw failed");
    }

    function getPrizePool() external view returns (uint256) {
        return prizePool;
    }
}
Warning: Unused local variable.
   --> Horse Racing.sol:109:18:
    |
109 |                 (bool success, ) = payable(playerList[second]).call{value: secondPrize}("");
    |                  ^^^^^^^^^^^^
