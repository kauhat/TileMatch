<template>
  <div>bum</div>
</template>

<script>
import IPFS from "ipfs";
import OrbitDB from "orbit-db";

import { v4 as uuidv4 } from "uuid";

// async function main () {
//   // Create IPFS instance
//   const ipfsOptions = { repo : './ipfs', }
//   const ipfs = await IPFS.create(ipfsOptions)

//   // Create OrbitDB instance
//   const orbitdb = await OrbitDB.createInstance(ipfs)
//   }

// main()
export default {
  name: "HelloWorld",
  props: {
    msg: String,
  },
  data: function () {
    return {
      ipfs: null,
      ipfsdbReady: false,
      orbitdb: null,
      db: null,
      lastEntry: null,
      dbCache: null,
      gameID: null,
      gameData: null,
      gameFound: false,
      gameOver: false,
      hashCheck: null,
      size: {
        x: 4,
        y: 4,
      },
      playerN: null,
      turnN: 0,
      pairsFound: [],
      score: null,
    };
  },
  mounted: function () {
    const ipfsOptions = { repo: "./ipfs" };
    this.ipfs = new IPFS(ipfsOptions);
    this.ipfs.on("ready", async () => {
      this.orbitdb = new OrbitDB(this.ipfs);
      this.db = await this.orbitdb.eventlog("TileMatch");
      this.ipfsdbREADY = true;
      this.collectMessagesOnTimer();
    });
  },
  collectMessagesOnTimer: function () {
    const timerFunc = () => {
      const asyncCall = async () => {
        const lastEntry = await this.db
          .iterator({ reverse: true })
          .collect()
          .map((e) => e.payload.value);
        if (lastEntry !== this.lastEntry) {
          this.dbCache = await this.db
            .iterator({ reverse: true, limit: -1 })
            .collect()
            .map((e) => e.payload.value);
        }
      };
      asyncCall();
      setTimeout(() => {
        timerFunc();
      }, 1000);
    };
    timerFunc();
  },
  sendMessage: function (message) {
    if (this.ipfsdbREADY === true) {
      const asyncCall = async () => {
        return await this.db.add(message);
      };
      asyncCall();
    } else {
      console.log("Waiting ipfs db");
    }
  },
  findGame: function () {
    let alreadyAccepted = [];
    // let found = false;
    this.dbCache.forEach((message) => {
      // check if the entry is too old
      if (message.moveID === "request") {
        if (!alreadyAccepted.includes(message.gameID)) {
          this.gameID = message.gameID;
          this.gameData = message.gameData;
          const acceptMessage = {
            gameID: this.gameID,
            gameData: this.gameData,
            moveID: "accept",
          };
          console.log("Found game, sending accept message");
          console.log("GameID", this.gameID);
          this.hashCheck = this.sendMessage(acceptMessage);
          this.acceptGameCheck();
          return;
        }
      } else if (message.moveID === "accept") {
        alreadyAccepted.push(message.gameID);
      }
    });
    if (!this.gameFound) {
      this.gameData = this.generateGameData();
      this.gameID = this.uuidv4();
      const requestMessage = {
        gameID: this.gameID,
        gameData: this.gameData,
        moveID: "request",
      };
      this.hashCheck = this.sendMessage(requestMessage);
      this.awaitAccept();
    }
  },
  acceptGameCheck: function () {
    const timerFunc = () => {
      this.dbCache.forEach((message) => {
        if (message.gameID === this.gameID && message.moveID === "accept") {
          if (message.hash === this.hashCheck) {
            this.gameFound = true;
            this.playerN = 1;
            this.awaitTurn();
            return;
          } else {
            this.findGame();
            console.log("Someone else accepted first! /nFinding new game");
            return;
          }
        }
      });
      setTimeout(() => {
        timerFunc();
      }, 1000);
    };
  },
  generateGameData: function (
    // Default Size
    size = {
      x: 4,
      y: 4,
    }
  ) {
    // Function Start
    // Generate list of all cell coordinates
    let cells = [];
    for (let i = 0; i < size.x; i++) {
      for (let j = 0; j < size.y; j++) {
        cells.push([i, j]);
      }
    }
    // Randomise those into a list of pairs
    let pairs = [];
    let tempPair = [];
    let random = null;
    for (let i = 0; i < size.x * size.y * 0.5; i++) {
      random = Math.floor(Math.random() * cells.length);
      tempPair[0] = [cells[random]];
      cells = cells.slice(random, 1);
      random = Math.floor(Math.random() * cells.length);
      tempPair[1] = [cells[random]];
      cells = cells.slice(random, 1);
      pairs[i] = tempPair;
    }
    // Return gameData object
    return {
      size: size,
      pairs: pairs,
    };
  },
  awaitAccept: function () {
    const timerFunc = () => {
      this.dbCache.forEach((message) => {
        if (message.gameID === this.gameID && message.moveID === "accept") {
          this.gameFound = true;
          this.playerN = 0;
          this.takeTurn();
          return;
        }
      });
      setTimeout(() => {
        timerFunc();
      }, 1000);
    };
  },
  takeTurn: function () {
    const input = prompt("Please enter your guess in the form/nx1,y1;x2,y2");
    const turn = input.split(";").map((coord) => coord.split(","));

    this.turnCheck(turn, this.playerN);

    const turnMessage = {
      gameID: this.gameID,
      gameData: this.gameData,
      moveID: this.turnN,
      turn: turn,
    };
    this.hashCheck = this.sendMessage(turnMessage);

    this.turnN++;

    if (this.gameOver) {
      console.log("GameOver");
    } else {
      this.awaitTurn();
    }
  },
  awaitTurn: function () {
    const timerFunc = () => {
      this.dbCache.forEach((message) => {
        if (message.gameID === this.gameID && message.moveID === this.turnN) {
          this.turnCheck(message.turn, (this.playerN + 1) % 2);
          if (this.gameOver) {
            console.log("GameOver");
          } else {
            this.takeTurn();
          }
        }
      });
      setTimeout(() => {
        timerFunc();
      }, 1000);
    };
  },
  turnCheck: function (turn, playerN) {
    const turnCheck = this.gameData.pairs.indexOf(turn);
    if (turnCheck !== -1) {
      this.pairsFound[turnCheck] = this.playerN;
      this.gameData.pairs = this.gameData.pairs.slice(turnCheck, 1);
      if (playerN === this.playerN) {
        this.score++;
      }
      if (this.gameData.pairs.length === 0) {
        this.gameOver = true;
      }
    }
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
