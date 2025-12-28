<script>
  // Initialize with default state or load from localStorage
  function loadGameState() {
    const saved = localStorage.getItem('monopoly-banker-state');
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        return {
          players: parsed.players || [
            { id: Date.now(), name: 'Player 1', money: 0 },
            { id: Date.now() + 1, name: 'Player 2', money: 0 }
          ],
          transactions: parsed.transactions || [],
          nextPlayerId: parsed.nextPlayerId || Date.now() + 2
        };
      } catch (e) {
        console.error('Failed to load saved state:', e);
      }
    }
    return {
      players: [
        { id: Date.now(), name: 'Player 1', money: 0 },
        { id: Date.now() + 1, name: 'Player 2', money: 0 }
      ],
      transactions: [],
      nextPlayerId: Date.now() + 2
    };
  }

  const initialState = loadGameState();
  let players = $state(initialState.players);
  let transactions = $state(initialState.transactions);
  let nextPlayerId = $state(initialState.nextPlayerId);

  let selectedPlayer = $state(null);
  let amount = $state(0);
  let amountDisplay = $state('');
  let transferFrom = $state(null);
  let transferTo = $state(null);
  let transferAmount = $state(0);
  let transferAmountDisplay = $state('');
  let newGameAmount = $state(0);
  let newGameAmountDisplay = $state('');
  let menuOpen = $state(false);
  let showNewGameModal = $state(false);

  // Save state to localStorage
  function saveState() {
    const state = {
      players,
      transactions,
      nextPlayerId
    };
    localStorage.setItem('monopoly-banker-state', JSON.stringify(state));
  }

  // Add transaction to history
  function addTransaction(type, data) {
    const transaction = {
      id: Date.now(),
      type,
      timestamp: new Date().toISOString(),
      ...data
    };
    transactions = [transaction, ...transactions];
    saveState();
  }

  // Get player name by ID (for history display)
  function getPlayerName(playerId) {
    const player = players.find(p => p.id === playerId);
    return player ? player.name : 'Unknown';
  }

  // Prefer current player name; fall back to stored name if player was removed
  function displayName(playerId, fallbackName = 'Unknown') {
    const player = players.find(p => p.id === playerId);
    return player ? player.name : fallbackName || 'Unknown';
  }

  // Add new player
  function addPlayer() {
    const newPlayer = {
      id: nextPlayerId++,
      name: `Player ${players.length + 1}`,
      money: 0
    };
    players = [...players, newPlayer];
    addTransaction('player_added', { playerId: newPlayer.id, playerName: newPlayer.name });
    saveState();
  }

  // Remove player (only if more than 2 players)
  function removePlayer(playerId) {
    if (players.length <= 2) return;
    const player = players.find(p => p.id === playerId);
    if (player) {
      players = players.filter(p => p.id !== playerId);
      addTransaction('player_removed', { playerId: player.id, playerName: player.name });
      // Clear selections if removed player was selected
      if (selectedPlayer === playerId) selectedPlayer = null;
      if (transferFrom === playerId) transferFrom = null;
      if (transferTo === playerId) transferTo = null;
      saveState();
    }
  }

  function givePlayer2M(playerId) {
    const player = players.find(p => p.id === playerId);
    if (player) {
      player.money += 2000000;
      addTransaction('give_2m', { playerId: player.id, playerName: player.name, amount: 2000000 });
      saveState();
    }
  }

  function addMoney(playerId) {
    if (amount > 0) {
      const player = players.find(p => p.id === playerId);
      if (player) {
        player.money += amount;
        addTransaction('add_money', { playerId: player.id, playerName: player.name, amount: amount });
        amount = 0;
        amountDisplay = '';
        selectedPlayer = null;
        saveState();
      }
    }
  }

  function withdrawMoney(playerId) {
    if (amount > 0) {
      const player = players.find(p => p.id === playerId);
      if (player && player.money >= amount) {
        player.money -= amount;
        addTransaction('withdraw_money', { playerId: player.id, playerName: player.name, amount: amount });
        amount = 0;
        amountDisplay = '';
        selectedPlayer = null;
        saveState();
      }
    }
  }

  function transfer() {
    if (transferAmount > 0 && transferFrom && transferTo && transferFrom !== transferTo) {
      const fromPlayer = players.find(p => p.id === transferFrom);
      const toPlayer = players.find(p => p.id === transferTo);
      if (fromPlayer && toPlayer && fromPlayer.money >= transferAmount) {
        fromPlayer.money -= transferAmount;
        toPlayer.money += transferAmount;
        addTransaction('transfer', {
          fromPlayerId: fromPlayer.id,
          fromPlayerName: fromPlayer.name,
          toPlayerId: toPlayer.id,
          toPlayerName: toPlayer.name,
          amount: transferAmount
        });
        transferAmount = 0;
        transferAmountDisplay = '';
        transferFrom = null;
        transferTo = null;
        saveState();
      }
    }
  }

  function formatMoney(value) {
    return new Intl.NumberFormat('en-US').format(value);
  }

  // Parse formatted number string to integer
  function parseFormattedNumber(str) {
    // Remove all non-digit characters
    const cleaned = str.replace(/\D/g, '');
    return cleaned ? parseInt(cleaned, 10) : 0;
  }

  // Handle amount input change
  function handleAmountInput(e) {
    const value = e.target.value;
    amountDisplay = value;
    amount = parseFormattedNumber(value);
  }

  // Format amount on blur
  function handleAmountBlur() {
    if (amount > 0) {
      amountDisplay = formatMoney(amount);
    } else {
      amountDisplay = '';
    }
  }

  // Handle transfer amount input change
  function handleTransferAmountInput(e) {
    const value = e.target.value;
    transferAmountDisplay = value;
    transferAmount = parseFormattedNumber(value);
  }

  // Format transfer amount on blur
  function handleTransferAmountBlur() {
    if (transferAmount > 0) {
      transferAmountDisplay = formatMoney(transferAmount);
    } else {
      transferAmountDisplay = '';
    }
  }

  // Handle new game start amount input change
  function handleNewGameAmountInput(e) {
    const value = e.target.value;
    newGameAmountDisplay = value;
    newGameAmount = parseFormattedNumber(value);
  }

  // Format new game amount on blur
  function handleNewGameAmountBlur() {
    if (newGameAmount > 0) {
      newGameAmountDisplay = formatMoney(newGameAmount);
    } else {
      newGameAmountDisplay = '';
    }
  }

  // Start a fresh game: set all player balances to chosen amount and clear history
  function startNewGame() {
    if (newGameAmount <= 0) return;
    players = players.map(player => ({ ...player, money: newGameAmount }));
    transactions = [];
    amount = 0;
    amountDisplay = '';
    transferAmount = 0;
    transferAmountDisplay = '';
    selectedPlayer = null;
    transferFrom = null;
    transferTo = null;
    saveState();
    showNewGameModal = false;
    newGameAmount = 0;
    newGameAmountDisplay = '';
  }

  // Full reset: reset players to default two, clear history, reset ids
  function resetGame() {
    const baseId = Date.now();
    players = [
      { id: baseId, name: 'Player 1', money: 0 },
      { id: baseId + 1, name: 'Player 2', money: 0 }
    ];
    nextPlayerId = baseId + 2;
    transactions = [];
    amount = 0;
    amountDisplay = '';
    transferAmount = 0;
    transferAmountDisplay = '';
    newGameAmount = 0;
    newGameAmountDisplay = '';
    selectedPlayer = null;
    transferFrom = null;
    transferTo = null;
    menuOpen = false;
    showNewGameModal = false;
    saveState();
  }

  function toggleMenu() {
    menuOpen = !menuOpen;
  }

  function closeMenu() {
    menuOpen = false;
  }

  function openNewGameModal() {
    showNewGameModal = true;
    menuOpen = false;
    newGameAmount = 0;
    newGameAmountDisplay = '';
  }

  function closeNewGameModal() {
    showNewGameModal = false;
  }

  function handleBackdropKeydown(event) {
    if (event.key === 'Escape' || event.key === 'Enter' || event.key === ' ') {
      event.preventDefault();
      closeNewGameModal();
    }
  }

  // Format amount for display
  function getFormattedAmount(value) {
    if (value === 0 || value === '') return '';
    return formatMoney(value);
  }

  function formatTime(timestamp) {
    const date = new Date(timestamp);
    return date.toLocaleTimeString();
  }

  function getTransactionDescription(transaction) {
    switch (transaction.type) {
      case 'transfer':
        return `${displayName(transaction.fromPlayerId, transaction.fromPlayerName)} transferred $${formatMoney(transaction.amount)} to ${displayName(transaction.toPlayerId, transaction.toPlayerName)}`;
      case 'add_money':
        return `Added $${formatMoney(transaction.amount)} to ${displayName(transaction.playerId, transaction.playerName)}`;
      case 'withdraw_money':
        return `${displayName(transaction.playerId, transaction.playerName)} withdrew $${formatMoney(transaction.amount)}`;
      case 'give_2m':
        return `Gave $2M to ${displayName(transaction.playerId, transaction.playerName)} (passed GO)`;
      case 'give_2m_all':
        return `Gave $2M to ${displayName(transaction.playerId, transaction.playerName)} (all players)`;
      case 'player_added':
        return `Added player: ${transaction.playerName}`;
      case 'player_removed':
        return `Removed player: ${transaction.playerName}`;
      default:
        return 'Unknown transaction';
    }
  }

  // Save state when player names change
  $effect(() => {
    if (players.length > 0) {
      saveState();
    }
  });
</script>

<div class="container">
  <div class="header">
    <h1>🏦 Monopoly Banker</h1>
    <div class="menu-wrapper">
      <button class="icon-button" onclick={toggleMenu} aria-label="Menu">
        ⋮
      </button>
      {#if menuOpen}
        <div class="menu-dropdown">
          <button class="menu-item" onclick={() => { addPlayer(); closeMenu(); }}>
            + Add Player
          </button>
          <button class="menu-item" onclick={openNewGameModal}>
            Start New Game
          </button>
          <button class="menu-item danger" onclick={resetGame}>
            Reset Game (clear all)
          </button>
        </div>
      {/if}
    </div>
  </div>

  <div class="players-grid">
    {#each players as player (player.id)}
      <div class="player-card">
        <div class="player-header">
          <div class="player-name-row">
            <input
              type="text"
              bind:value={player.name}
              class="player-name"
            />
            {#if players.length > 2}
              <button
                class="btn-remove"
                onclick={() => removePlayer(player.id)}
                title="Remove player"
              >
                ×
              </button>
            {/if}
          </div>
          <div class="player-money">${formatMoney(player.money)}</div>
        </div>
        
        <div class="player-actions">
          <button
            class="btn btn-primary btn-small"
            onclick={() => givePlayer2M(player.id)}
          >
            Give $2M
          </button>
          <button
            class="btn btn-small"
            onclick={() => selectedPlayer = selectedPlayer === player.id ? null : player.id}
          >
            {selectedPlayer === player.id ? 'Cancel' : 'Manage'}
          </button>
        </div>

        {#if selectedPlayer === player.id}
          <div class="manage-panel">
            <input
              type="text"
              value={amountDisplay}
              oninput={handleAmountInput}
              onblur={handleAmountBlur}
              placeholder="Amount (e.g., 1000000)"
              class="input"
            />
            {#if amount > 0}
              <div class="amount-preview">${formatMoney(amount)}</div>
            {/if}
            <div class="button-group">
              <button
                class="btn btn-success btn-small"
                onclick={() => addMoney(player.id)}
              >
                Add
              </button>
              <button
                class="btn btn-danger btn-small"
                onclick={() => withdrawMoney(player.id)}
              >
                Withdraw
              </button>
            </div>
          </div>
        {/if}
      </div>
    {/each}
  </div>

  <div class="transfer-section">
    <h2>Transfer Money</h2>
    <div class="transfer-controls">
      <div class="transfer-field">
        <label for="transfer-from">From:</label>
        <select id="transfer-from" bind:value={transferFrom} class="select">
          <option value={null}>Select player</option>
          {#each players as player}
            <option value={player.id}>{player.name}</option>
          {/each}
        </select>
      </div>
      
      <div class="transfer-field">
        <label for="transfer-to">To:</label>
        <select id="transfer-to" bind:value={transferTo} class="select">
          <option value={null}>Select player</option>
          {#each players as player}
            <option value={player.id}>{player.name}</option>
          {/each}
        </select>
      </div>
      
      <div class="transfer-field">
        <label for="transfer-amount">Amount:</label>
        <input
          id="transfer-amount"
          type="text"
          value={transferAmountDisplay}
          oninput={handleTransferAmountInput}
          onblur={handleTransferAmountBlur}
          placeholder="Amount (e.g., 1000000)"
          class="input"
        />
        {#if transferAmount > 0}
          <div class="amount-preview">${formatMoney(transferAmount)}</div>
        {/if}
      </div>
      
      <button
        class="btn btn-primary"
        onclick={transfer}
        disabled={!transferFrom || !transferTo || transferAmount <= 0 || transferFrom === transferTo}
      >
        Transfer
      </button>
    </div>
  </div>

  <div class="history-section">
    <h2>Transaction History</h2>
    <div class="history-list">
      {#if transactions.length === 0}
        <div class="history-empty">No transactions yet</div>
      {:else}
        {#each transactions as transaction (transaction.id)}
          <div class="history-item">
            <div class="history-time">{formatTime(transaction.timestamp)}</div>
            <div class="history-description">{getTransactionDescription(transaction)}</div>
          </div>
        {/each}
      {/if}
    </div>
  </div>

  {#if showNewGameModal}
    <div
      class="modal-backdrop"
      role="button"
      tabindex="0"
      onclick={closeNewGameModal}
      onkeydown={handleBackdropKeydown}
      aria-label="Close new game dialog"
    ></div>
    <div class="modal">
      <h3>Start New Game</h3>
      <p>Set the starting balance for every player. This will clear the history.</p>
      <input
        type="text"
        value={newGameAmountDisplay}
        oninput={handleNewGameAmountInput}
        onblur={handleNewGameAmountBlur}
        placeholder="Start amount (e.g., 1500000)"
        class="input"
      />
      {#if newGameAmount > 0}
        <div class="amount-preview">Each player starts with ${formatMoney(newGameAmount)}</div>
      {/if}
      <div class="modal-actions">
        <button class="btn btn-small" onclick={closeNewGameModal}>Cancel</button>
        <button class="btn btn-primary btn-small" onclick={startNewGame} disabled={newGameAmount <= 0}>
          Start New Game
        </button>
      </div>
    </div>
  {/if}
</div>

<style>
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  :global(body) {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    padding: 20px;
    margin: 0;
  }

  .container {
    max-width: 1200px;
    margin: 0 auto;
  }

  h1 {
    color: white;
    margin: 0;
    font-size: 2.3rem;
    text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
  }

  h2 {
    color: white;
    margin-bottom: 20px;
    font-size: 1.5rem;
  }

  .header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 30px;
  }

  .btn {
    padding: 12px 24px;
    font-size: 1rem;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-weight: 600;
    transition: all 0.2s;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }

  .btn:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  }

  .btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .btn-primary {
    background: #4CAF50;
    color: white;
  }

  .btn-success {
    background: #2196F3;
    color: white;
  }

  .btn-danger {
    background: #f44336;
    color: white;
  }

  .icon-button {
    width: 42px;
    height: 42px;
    border-radius: 50%;
    border: none;
    background: white;
    color: #444;
    font-size: 22px;
    font-weight: 700;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
    cursor: pointer;
    transition: transform 0.2s, box-shadow 0.2s;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .icon-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
  }

  .menu-wrapper {
    position: relative;
  }

  .menu-dropdown {
    position: absolute;
    right: 0;
    top: 48px;
    background: white;
    border-radius: 10px;
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.18);
    min-width: 180px;
    padding: 6px 0;
    z-index: 10;
  }

  .menu-item {
    width: 100%;
    background: transparent;
    border: none;
    text-align: left;
    padding: 10px 14px;
    font-size: 0.95rem;
    cursor: pointer;
    transition: background 0.15s, padding-left 0.15s;
  }

  .menu-item:hover {
    background: #f3f6ff;
    padding-left: 18px;
  }

  .menu-item.danger {
    color: #d32f2f;
  }

  .menu-item.danger:hover {
    background: #fff2f2;
  }

  .btn-small {
    padding: 8px 16px;
    font-size: 0.9rem;
  }

  .players-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 20px;
    margin-bottom: 40px;
  }

  .player-card {
    background: white;
    border-radius: 12px;
    padding: 20px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    transition: transform 0.2s;
  }

  .player-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
  }

  .player-header {
    margin-bottom: 15px;
  }

  .player-name-row {
    display: flex;
    gap: 8px;
    align-items: center;
    margin-bottom: 10px;
  }

  .player-name {
    flex: 1;
    padding: 8px;
    font-size: 1.1rem;
    font-weight: 600;
    border: 2px solid #e0e0e0;
    border-radius: 6px;
  }

  .player-name:focus {
    outline: none;
    border-color: #667eea;
  }

  .btn-remove {
    width: 32px;
    height: 32px;
    border: none;
    background: #f44336;
    color: white;
    border-radius: 50%;
    cursor: pointer;
    font-size: 20px;
    font-weight: bold;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;
    flex-shrink: 0;
  }

  .btn-remove:hover {
    background: #d32f2f;
    transform: scale(1.1);
  }

  .player-money {
    font-size: 1.8rem;
    font-weight: bold;
    color: #4CAF50;
    text-align: center;
  }

  .player-actions {
    margin-bottom: 10px;
    display: flex;
    gap: 10px;
  }

  .player-actions .btn {
    flex: 1;
  }

  .manage-panel {
    margin-top: 15px;
    padding-top: 15px;
    border-top: 2px solid #e0e0e0;
  }

  .input {
    width: 100%;
    padding: 10px;
    font-size: 1rem;
    border: 2px solid #e0e0e0;
    border-radius: 6px;
    margin-bottom: 10px;
  }

  .input:focus {
    outline: none;
    border-color: #667eea;
  }

  .amount-preview {
    font-size: 0.9rem;
    color: #4CAF50;
    font-weight: 600;
    margin-top: 5px;
    margin-bottom: 10px;
    text-align: center;
  }

  .button-group {
    display: flex;
    gap: 10px;
  }

  .button-group .btn {
    flex: 1;
  }

  .transfer-section {
    background: white;
    border-radius: 12px;
    padding: 30px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    margin-bottom: 30px;
  }

  .transfer-controls {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 20px;
    align-items: end;
  }

  .transfer-field {
    display: flex;
    flex-direction: column;
  }

  .transfer-field label {
    margin-bottom: 8px;
    font-weight: 600;
    color: #333;
  }

  .select {
    padding: 10px;
    font-size: 1rem;
    border: 2px solid #e0e0e0;
    border-radius: 6px;
    background: white;
  }

  .select:focus {
    outline: none;
    border-color: #667eea;
  }

  .history-section {
    background: white;
    border-radius: 12px;
    padding: 30px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .history-list {
    max-height: 400px;
    overflow-y: auto;
    border: 2px solid #e0e0e0;
    border-radius: 8px;
    padding: 10px;
  }

  .history-empty {
    text-align: center;
    color: #999;
    padding: 20px;
  }

  .history-item {
    padding: 12px;
    border-bottom: 1px solid #e0e0e0;
    display: flex;
    gap: 15px;
    align-items: center;
  }

  .history-item:last-child {
    border-bottom: none;
  }

  .history-time {
    font-size: 0.85rem;
    color: #666;
    min-width: 80px;
  }

  .history-description {
    flex: 1;
    color: #333;
  }

  .modal-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.45);
    z-index: 20;
  }

  .modal {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: white;
    padding: 24px;
    border-radius: 12px;
    box-shadow: 0 12px 30px rgba(0, 0, 0, 0.22);
    width: min(420px, 90vw);
    z-index: 30;
  }

  .modal h3 {
    margin-bottom: 10px;
    color: #222;
  }

  .modal p {
    margin-bottom: 14px;
    color: #444;
    line-height: 1.4;
  }

  .modal-actions {
    display: flex;
    gap: 10px;
    justify-content: flex-end;
    margin-top: 12px;
  }
</style>
