<template>
    <div class="game-container">
      <div v-if="!connected" class="connection-panel">
        <h2>Nine Men's Morris</h2>
        <div class="form-group">
          <label for="roomId">Room ID:</label>
          <input v-model="roomId" type="text" id="roomId" placeholder="Enter room ID">
        </div>
        <button @click="createOrJoinGame" class="btn primary">Join Game</button>
        <p v-if="connectionMessage" class="status-message">{{ connectionMessage }}</p>
      </div>
  
      <div v-else class="game-board-container">
        <div class="game-info">
          <h2>Nine Men's Morris - Room: {{ roomId }}</h2>
          <p class="debug-info">You are playing as: {{ playerColor }}</p>
          <div class="player-info">
            <div class="player white" :class="{ active: gameState.currentPlayer === 'W' }">
              <div class="player-color"></div>
              <div class="player-details">
                <h3>White Player</h3>
                <p v-if="gameState.CurrentPhase === phases.PLACEMENT">
                  Tokens Remaining: {{ gameState.WhiteTokensRemaining }}
                </p>
                <p>Tokens on Board: {{ gameState.WhiteTokensOnBoard }}</p>
              </div>
            </div>
            <div class="player black" :class="{ active: gameState.currentPlayer === 'B' }">
              <div class="player-color"></div>
              <div class="player-details">
                <h3>Black Player</h3>
                <p v-if="gameState.CurrentPhase === phases.PLACEMENT">
                  Tokens Remaining: {{ gameState.BlackTokensRemaining }}
                </p>
                <p>Tokens on Board: {{ gameState.BlackTokensOnBoard }}</p>
              </div>
            </div>
          </div>
          <div class="game-status">
            <p class="phase">Phase: {{ gameState.CurrentPhase }}</p>
            <p class="turn">
              Turn: {{ gameState.currentPlayer === 'W' ? 'White' : 'Black' }}
              <span v-if="gameState.LastMoveFormedMill"> - Remove an opponent's token</span>
            </p>
            <p v-if="isPlayersTurn" class="instruction">
              <span v-if="gameState.CurrentPhase === phases.PLACEMENT">Place a token by clicking on an empty position</span>
              <span v-else-if="gameState.LastMoveFormedMill">Remove an opponent's token</span>
              <span v-else-if="selectedPosition !== null">Select a destination to move this token</span>
              <span v-else>Select one of your tokens to move</span>
            </p>
            <p v-else class="waiting">Waiting for opponent's move...</p>
            <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
          </div>
          <button @click="leaveGame" class="btn secondary">Leave Game</button>
        </div>
  
        <div class="board">
          <svg viewBox="0 0 700 700" xmlns="http://www.w3.org/2000/svg">
            <!-- Outer square -->
            <rect x="50" y="50" width="600" height="600" fill="none" stroke="#8B4513" stroke-width="4" />
            
            <!-- Middle square -->
            <rect x="150" y="150" width="400" height="400" fill="none" stroke="#8B4513" stroke-width="4" />
            
            <!-- Inner square -->
            <rect x="250" y="250" width="200" height="200" fill="none" stroke="#8B4513" stroke-width="4" />
            
            <!-- Connecting lines -->
            <line x1="350" y1="50" x2="350" y2="250" stroke="#8B4513" stroke-width="4" />
            <line x1="350" y1="450" x2="350" y2="650" stroke="#8B4513" stroke-width="4" />
            <line x1="50" y1="350" x2="250" y2="350" stroke="#8B4513" stroke-width="4" />
            <line x1="450" y1="350" x2="650" y2="350" stroke="#8B4513" stroke-width="4" />
  
            <!-- Board positions -->
            <template v-for="(pos, index) in positions" :key="index">
              <!-- Larger clickable area -->
              <circle
                :cx="pos.x"
                :cy="pos.y"
                r="30"
                :fill="isPositionClickable(index) ? 'rgba(255, 255, 255, 0.1)' : 'transparent'"
                class="clickable-area"
                :class="{ 'clickable': isPositionClickable(index) }"
                @click="handlePositionClick(index)"
              />
              
              <!-- Visual position indicator -->
              <circle
                :cx="pos.x"
                :cy="pos.y"
                r="20"
                fill="#D2B48C"
                stroke="#8B4513"
                stroke-width="2"
                class="position"
                :class="{
                  'highlight-source': selectedPosition === index,
                  'valid-move': isValidMove(index),
                  'highlight': shouldHighlight(index)
                }"
              />
  
              <!-- Token display -->
              <circle
                v-if="gameState.Board && gameState.Board[index] !== ' '"
                :cx="pos.x"
                :cy="pos.y"
                r="15"
                :fill="gameState.Board[index] === 'W' ? '#FFFFFF' : '#000000'"
                stroke="#333"
                stroke-width="1"
              />
            </template>
          </svg>
        </div>
      </div>
  
      <!-- Game over modal -->
      <div v-if="gameState.IsGameOver" class="modal">
        <div class="modal-content">
          <h2>Game Over</h2>
          <p>{{ gameState.WinMessage }}</p>
          <button @click="leaveGame" class="btn primary">Back to Lobby</button>
        </div>
      </div>
    </div>
  </template>
  
  <script>
  import * as signalR from "@microsoft/signalr";
  
  export default {
    name: "GameBoard",
    data() {
      return {
        connection: null,
        connected: false,
        roomId: "",
        playerColor: null,
        connectionMessage: "",
        errorMessage: "",
        // Add phase constants
        phases: {
          PLACEMENT: "Placement",
          MOVEMENT: "Movement",
          FLYING: "Flying"
        },
        gameState: {
          Board: Array(24).fill(" "),
          currentPlayer: "W",
          WhiteTokensRemaining: 9,
          BlackTokensRemaining: 9,
          WhiteTokensOnBoard: 0,
          BlackTokensOnBoard: 0,
          CurrentPhase: "Placement",
          LastMoveFormedMill: false,
          IsGameOver: false,
          Winner: null,
          WinMessage: ""
        },
        selectedPosition: null,
        // Define board positions coordinates
        positions: [
          // Top row
          { x: 50, y: 50 },     // 0
          { x: 350, y: 50 },    // 1
          { x: 650, y: 50 },    // 2
          
          // Second row
          { x: 150, y: 150 },   // 3
          { x: 350, y: 150 },   // 4
          { x: 550, y: 150 },   // 5
          
          // Third row
          { x: 250, y: 250 },   // 6
          { x: 350, y: 250 },   // 7
          { x: 450, y: 250 },   // 8
          
          // Middle left
          { x: 50, y: 350 },    // 9
          { x: 150, y: 350 },   // 10
          { x: 250, y: 350 },   // 11
          
          // Middle right
          { x: 450, y: 350 },   // 12
          { x: 550, y: 350 },   // 13
          { x: 650, y: 350 },   // 14
          
          // Third row from bottom
          { x: 250, y: 450 },   // 15
          { x: 350, y: 450 },   // 16
          { x: 450, y: 450 },   // 17
          
          // Second row from bottom
          { x: 150, y: 550 },   // 18
          { x: 350, y: 550 },   // 19
          { x: 550, y: 550 },   // 20
          
          // Bottom row
          { x: 50, y: 650 },    // 21
          { x: 350, y: 650 },   // 22
          { x: 650, y: 650 }    // 23
        ],
        adjacencyMap: {
          0: [1, 9],
          1: [0, 2, 4],
          2: [1, 14],
          3: [4, 10],
          4: [1, 3, 5, 7],
          5: [4, 13],
          6: [7, 11],
          7: [4, 6, 8],
          8: [7, 12],
          9: [0, 10, 21],
          10: [3, 9, 11, 18],
          11: [6, 10, 15],
          12: [8, 13, 17],
          13: [5, 12, 14, 20],
          14: [2, 13, 23],
          15: [11, 16],
          16: [15, 17, 19],
          17: [12, 16],
          18: [10, 19],
          19: [16, 18, 20, 22],
          20: [13, 19],
          21: [9, 22],
          22: [19, 21, 23],
          23: [14, 22]
        }
      };
    },
    computed: {
      isPlayersTurn() {
        return (this.playerColor === "White" && this.gameState.currentPlayer === "W") || 
               (this.playerColor === "Black" && this.gameState.currentPlayer === "B");
      }
    },
    methods: {
      // Update phase handling in event handlers
      handleGameState(gameState) {
        // Map the backend's phase to our frontend phase constants
        let phase = this.phases.PLACEMENT;
        if (gameState.currentPhase === "Movement") {
          phase = this.phases.MOVEMENT;
        } else if (gameState.currentPhase === "Flying") {
          phase = this.phases.FLYING;
        }

        console.log("Mapping phase from backend:", gameState.currentPhase, "to:", phase);

        return {
          Board: gameState.board || Array(24).fill(" "),
          currentPlayer: gameState.currentPlayer || "W",
          WhiteTokensRemaining: Math.max(0, gameState.whiteTokensRemaining ?? 9),
          BlackTokensRemaining: Math.max(0, gameState.blackTokensRemaining ?? 9),
          WhiteTokensOnBoard: gameState.whiteTokensOnBoard ?? 0,
          BlackTokensOnBoard: gameState.blackTokensOnBoard ?? 0,
          CurrentPhase: phase,
          LastMoveFormedMill: gameState.lastMoveFormedMill ?? false,
          IsGameOver: gameState.isGameOver ?? false,
          Winner: gameState.winner ?? null,
          WinMessage: gameState.winMessage ?? ""
        };
      },

      setupSignalRConnection() {
        this.connection = new signalR.HubConnectionBuilder()
          .withUrl("https://mill-game-signalr-sob.runasp.net/gamehub")
          .withAutomaticReconnect()
          .build();

        // Handle connection events
        this.connection.onclose(() => {
          this.connected = false;
          this.connectionMessage = "Connection closed. Refresh to reconnect.";
        });

        // Set up message handlers
        this.connection.on("JoinedAsWhite", (message) => {
          this.playerColor = "White";
          this.connectionMessage = message;
        });

        this.connection.on("JoinedAsBlack", () => {
          this.playerColor = "Black";
          this.connectionMessage = "You joined as Black player";
        });

        this.connection.on("JoinedAsSpectator", (gameState) => {
          this.playerColor = "Spectator";
          this.gameState = gameState;
          this.connectionMessage = "";
        });

        this.connection.on("GameStarted", (gameState) => {
          console.log("Received gameState:", gameState);
          this.gameState = this.handleGameState(gameState);
          this.connected = true;
          this.connectionMessage = "";
          console.log("Game started with current player:", this.gameState.currentPlayer);
        });

        this.connection.on("TurnChanged", (gameState) => {
          console.log("Turn changed, raw gameState:", gameState);
          this.gameState = this.handleGameState(gameState);

          // Check if we should transition to movement phase
          if (this.gameState.WhiteTokensRemaining === 0 && 
              this.gameState.BlackTokensRemaining === 0 && 
              this.gameState.CurrentPhase === this.phases.PLACEMENT) {
            this.gameState.CurrentPhase = this.phases.MOVEMENT;
            console.log("Transitioning to movement phase");
          }

          console.log("Mapped gameState:", this.gameState);
          console.log("Current player after turn change:", this.gameState.currentPlayer);
        });

        this.connection.on("TokenPlaced", (game, position) => {
          console.log("Token placed at position", position);
          console.log("Raw game state:", game);
          this.gameState = this.handleGameState(game);
          console.log("Mapped gameState after token placement:", this.gameState);
          this.updateBoard();
        });

        this.connection.on("TokenMoved", (game, sourcePos, destPos) => {
          console.log("Token moved from", sourcePos, "to", destPos);
          console.log("Raw game state:", game);
          this.gameState = this.handleGameState(game);
          console.log("Mapped gameState after token movement:", this.gameState);
          this.selectedPosition = null;
          this.updateBoard();
        });

        this.connection.on("GameOver", (winner) => {
          console.log("Game Over! Winner:", winner);
          this.handleGameOver(winner);
        });

        this.connection.on("Error", (message) => {
          console.error("Received error from server:", message);
          this.errorMessage = message;
          this.selectedPosition = null; // Clear selection on error
          setTimeout(() => { this.errorMessage = ""; }, 3000);
        });

        this.connection.on("PhaseChanged", (newPhase) => {
          console.log("Phase changed from backend:", newPhase);
          // Map the backend phase to our frontend phase constants
          if (newPhase === "Movement") {
            this.gameState.CurrentPhase = this.phases.MOVEMENT;
          } else if (newPhase === "Flying") {
            this.gameState.CurrentPhase = this.phases.FLYING;
          } else {
            this.gameState.CurrentPhase = this.phases.PLACEMENT;
          }
          console.log("Phase updated to:", this.gameState.CurrentPhase);
        });

        this.connection.on("TokenRemoved", (gameState, position) => {
  console.log("Token removed, checking win conditions");
  console.log("Received game state:", gameState);
  
  // Update game state
  this.gameState = this.handleGameState(gameState);
  
  // Game over state should now be correctly set from the server
  console.log("Updated game state after token removal:", this.gameState);
});

        return this.connection.start();
      },

      async createOrJoinGame() {
        if (!this.roomId.trim()) {
          this.errorMessage = "Please enter a room ID";
          return;
        }

        try {
          if (!this.connection) {
            await this.setupSignalRConnection();
          }
          await this.connection.invoke("CreateOrJoinGame", this.roomId);
        } catch (err) {
          console.error(err);
          this.errorMessage = "Failed to join game.";
        }
      },

      async leaveGame() {
        try {
          await this.connection.invoke("LeaveGame", this.roomId);
          this.connected = false;
          this.playerColor = null;
          this.gameState = {
            Board: Array(24).fill(" "),
            currentPlayer: "W",
            WhiteTokensRemaining: 9,
            BlackTokensRemaining: 9,
            WhiteTokensOnBoard: 0,
            BlackTokensOnBoard: 0,
            CurrentPhase: "Placement",
            LastMoveFormedMill: false,
            IsGameOver: false,
            Winner: null,
            WinMessage: ""
          };
          this.selectedPosition = null;
        } catch (err) {
          console.error(err);
        }
      },

      handlePositionClick(index) {
        console.log("Clicked position:", index);
        console.log("Current gameState:", this.gameState);

        if (!this.gameState) {
          console.error("gameState is not initialized");
          return;
        }

        // Safely access the Board property
        const board = this.gameState.Board;
        console.log("Board array:", board);
        
        if (!board || !Array.isArray(board)) {
          console.error("gameState.Board is not initialized properly", this.gameState);
          return;
        }

        if (index < 0 || index >= board.length) {
          console.error("Invalid index:", index);
          return;
        }

        console.log("Is player's turn?", this.isPlayersTurn);
        console.log("Current phase:", this.gameState.CurrentPhase);  

        if (!this.isPlayersTurn) {
          console.log("Not player's turn, ignoring click");
          return;
        }

        this.errorMessage = "";

        if (this.gameState.IsGameOver) {
          console.log("Game is over, ignoring clicks");
          return;
        }

        if (this.gameState.LastMoveFormedMill) {
          console.log("Mill formed, removing opponent's token");
          this.removeToken(index);
          return;
        }

        if (this.gameState.CurrentPhase === this.phases.PLACEMENT) {
          console.log("Placing token at position", index);
          this.placeToken(index);
          return;
        }

        // Check if the current player can make any moves
        if (!this.canPlayerMove()) {
          console.log("Current player has no valid moves, passing turn");
          this.connection.invoke("PassTurn", this.roomId).catch(err => {
            console.error("Error passing turn:", err);
            this.errorMessage = "Failed to pass turn.";
          });
          return;
        }

        // Movement phase logic
        const currentPlayerToken = this.gameState.currentPlayer;
        
        // If no token is selected
        if (this.selectedPosition === null) {
          // Check if clicked on own token
          if (board[index] === currentPlayerToken) {
            this.selectedPosition = index;
            console.log("Selected token at position:", index);
          } else {
            console.log("Must select your own token first");
            this.errorMessage = "Select one of your tokens to move";
          }
          return;
        }
        
        // If a token is already selected
        if (board[index] === currentPlayerToken) {
          // If clicking on another own token, update selection
          this.selectedPosition = index;
          console.log("Changed selection to token at position:", index);
          return;
        }
        
        // Attempt to move to empty position
        if (board[index] === " " && this.isValidMove(index)) {
          console.log("Moving token from", this.selectedPosition, "to", index);
          this.moveToken(this.selectedPosition, index);
          this.selectedPosition = null;
        } else {
          console.log("Invalid move target");
          this.errorMessage = "Invalid move";
        }
      },

      async placeToken(position) {
        console.log("Attempting to place token at:", position);
        try {
          await this.connection.invoke("PlaceToken", this.roomId, position);
          console.log("Token placement invoked successfully.");
        } catch (err) {
          console.error("Error placing token:", err);
          this.errorMessage = err;  // Backend error message
        }
      },

      async moveToken(sourcePos, destPos) {
        console.log("Moving token - Source:", sourcePos, "Destination:", destPos);
        console.log("Current phase:", this.gameState.CurrentPhase);
        console.log("Source position content:", this.gameState.Board[sourcePos]);
        console.log("Destination position content:", this.gameState.Board[destPos]);
        console.log("Valid adjacent positions for source:", this.adjacencyMap[sourcePos]);
        
        if (this.gameState.CurrentPhase === this.phases.PLACEMENT) {
          console.error("Cannot move tokens during placement phase");
          this.errorMessage = "Cannot move tokens during placement phase";
          return;
        }
        
        try {
          await this.connection.invoke("MoveToken", this.roomId, sourcePos, destPos);
          console.log("Move token request sent successfully");
        } catch (err) {
          console.error("Error moving token:", err);
          this.errorMessage = "Failed to move token.";
          this.selectedPosition = null;
        }
      },

      async removeToken(position) {
        try {
          await this.connection.invoke("RemoveToken", this.roomId, position);
          console.log("Token removal request sent successfully");
        } catch (err) {
          console.error("Error removing token:", err);
          this.errorMessage = "Failed to remove token.";
        }
      },

      isValidMove(position) {
        if (this.selectedPosition === null || !this.gameState.Board) {
          console.log("No position selected or board not initialized");
          return false;
        }
        
        if (this.gameState.Board[position] !== " ") {
          console.log("Target position is occupied:", position);
          return false;
        }

        // Don't allow moves during placement phase
        if (this.gameState.CurrentPhase === this.phases.PLACEMENT) {
          console.log("Cannot move during placement phase");
          return false;
        }
        
        const playerTokenCount = this.gameState.currentPlayer === "W" 
          ? this.gameState.WhiteTokensOnBoard 
          : this.gameState.BlackTokensOnBoard;
          
        console.log("Player token count:", playerTokenCount);
        
        // Allow flying moves if in flying phase or player has 3 tokens
        if (this.gameState.CurrentPhase === this.phases.FLYING || playerTokenCount <= 3) {
          console.log("Flying phase or 3 tokens - all moves valid");
          return true;
        }
        
        const validMoves = this.adjacencyMap[this.selectedPosition];
        console.log("Valid moves from position", this.selectedPosition, ":", validMoves);
        const isAdjacent = validMoves.includes(position);
        console.log("Is target position adjacent?", isAdjacent);
        
        return isAdjacent;
      },

      shouldHighlight(position) {
        if (!this.gameState.Board) return false;

        // Only show highlights if it's the player's turn
        if (!this.isPlayersTurn) return false;

        if (this.gameState.LastMoveFormedMill) {
          const opponentColor = this.gameState.currentPlayer === "W" ? "B" : "W";
          return this.gameState.Board[position] === opponentColor;
        }

        if (this.gameState.CurrentPhase === this.phases.PLACEMENT) {
          // During placement phase, highlight empty positions
          return this.gameState.Board[position] === " ";
        }

        if (this.selectedPosition === null) {
          // During movement phase, highlight current player's tokens that can be selected
          return this.gameState.Board[position] === this.gameState.currentPlayer;
        }

        // When a token is selected, highlight valid move destinations
        if (this.isValidMove(position)) {
          return true;
        }

        return false;
      },

      // Add method to determine if a position is clickable
      isPositionClickable(position) {
        if (!this.isPlayersTurn) return false;

        if (this.gameState.LastMoveFormedMill) {
          // During mill, only opponent's tokens are clickable
          const opponentColor = this.gameState.currentPlayer === "W" ? "B" : "W";
          return this.gameState.Board[position] === opponentColor;
        }

        if (this.gameState.CurrentPhase === this.phases.PLACEMENT) {
          // During placement, only empty positions are clickable
          return this.gameState.Board[position] === " ";
        }

        if (this.selectedPosition === null) {
          // When no token selected, only current player's tokens are clickable
          return this.gameState.Board[position] === this.gameState.currentPlayer;
        }

        // When a token is selected, only valid move destinations are clickable
        return this.isValidMove(position);
      },

      // Update the handleGameOver method to properly set game over state
      handleGameOver(winner) {
        console.log("Game Over! Winner:", winner);
        
        // Add a descriptive message based on the win condition
        const loser = winner === "White" ? "Black" : "White";
        const loserTokens = this.gameState[`${loser}TokensOnBoard`];
        const winMessage = `${winner} wins! ${loser} has only ${loserTokens} tokens remaining.`;
        
        // Update game state for game over
        this.gameState = {
          ...this.gameState,
          IsGameOver: true,
          Winner: winner,
          WinMessage: winMessage,
          currentPlayer: winner === "White" ? "W" : "B"
        };
        
        this.selectedPosition = null;
        this.errorMessage = "";  // Clear any existing error messages
        
        console.log("Game over state set:", this.gameState);
      },

      // Update the template phase check
      isPlacementPhase() {
        return this.gameState.CurrentPhase === this.phases.PLACEMENT;
      },

      canPlayerMove() {
        if (!this.gameState.Board || this.gameState.CurrentPhase === this.phases.PLACEMENT) {
          return true;
        }

        const currentPlayer = this.gameState.currentPlayer;
        const playerTokenCount = currentPlayer === "W" 
          ? this.gameState.WhiteTokensOnBoard 
          : this.gameState.BlackTokensOnBoard;

        // Check each position on the board
        for (let pos = 0; pos < this.gameState.Board.length; pos++) {
          // If we find a token belonging to the current player
          if (this.gameState.Board[pos] === currentPlayer) {
            // In flying phase or when player has 3 tokens, they can move anywhere
            if (this.gameState.CurrentPhase === this.phases.FLYING || playerTokenCount <= 3) {
              // Check if there's at least one empty spot
              if (this.gameState.Board.includes(" ")) {
                return true;
              }
            } else {
              // Check adjacent positions
              const adjacentPositions = this.adjacencyMap[pos];
              // If any adjacent position is empty, the player can move
              if (adjacentPositions.some(adjPos => this.gameState.Board[adjPos] === " ")) {
                return true;
              }
            }
          }
        }
        
        // If we get here, no valid moves were found
        return false;
      }
    }
  };
</script>

<style scoped>
.game-container {
  font-family: Arial, sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.connection-panel {
  max-width: 400px;
  margin: 100px auto;
  padding: 30px;
  background: #f8f8f8;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  text-align: center;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: bold;
}

.form-group input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.3s;
}

.btn.primary {
  background: #4CAF50;
  color: white;
}

.btn.primary:hover {
  background: #3e8e41;
}

.btn.secondary {
  background: #f44336;
  color: white;
}

.btn.secondary:hover {
  background: #d32f2f;
}

.status-message {
  margin-top: 20px;
  color: #666;
}

.error-message {
  color: red;
  margin-top: 10px;
}

.game-board-container {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.game-info {
  background: #f8f8f8;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.player-info {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
}

.player {
  display: flex;
  align-items: center;
  padding: 10px;
  border-radius: 4px;
  opacity: 0.7;
  transition: all 0.3s;
}

.player.active {
  opacity: 1;
  background: rgba(74, 144, 226, 0.1);
  box-shadow: 0 0 5px rgba(74, 144, 226, 0.3);
}

.player-color {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  margin-right: 10px;
}

.white .player-color {
  background: white;
  border: 1px solid #ccc;
}

.black .player-color {
  background: black;
}

.game-status {
  background: white;
  padding: 15px;
  border-radius: 4px;
  margin-bottom: 20px;
}

.phase, .turn {
  font-weight: bold;
  margin-bottom: 8px;
}

.instruction {
  color: #4CAF50;
}

.waiting {
  color: #FF9800;
}

.board {
  background: #E8D0AA;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
}

.position {
  pointer-events: none;
  transition: all 0.2s;
}

.clickable-area {
  cursor: default;
  transition: all 0.2s;
}

.clickable-area.clickable {
  cursor: pointer;
}

.clickable-area.clickable:hover {
  fill: rgba(255, 255, 255, 0.2);
}

.clickable-area.clickable:hover + .position {
  filter: brightness(1.1);
}

.highlight-source {
  stroke: #4CAF50;
  stroke-width: 4;
}

.valid-move {
  stroke: #2196F3;
  stroke-width: 4;
  animation: pulse 1.5s infinite;
}

.highlight {
  stroke: #FF9800;
  stroke-width: 3;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { opacity: 0.7; }
  50% { opacity: 1; }
  100% { opacity: 0.7; }
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  padding: 30px;
  border-radius: 8px;
  text-align: center;
  max-width: 400px;
}

.modal-content h2 {
  margin-top: 0;
}

.modal-content button {
  margin-top: 20px;
}
</style>
