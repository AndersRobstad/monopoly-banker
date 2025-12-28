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
  let showNewGameModal = $state(false);
  let showHistory = $state(false);

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
    showNewGameModal = false;
    saveState();
  }

  function openNewGameModal() {
    showNewGameModal = true;
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

  function toggleHistory() {
    showHistory = !showHistory;
  }

  // Format amount for display
  function getFormattedAmount(value) {
    if (value === 0 || value === '') return '';
    return formatMoney(value);
  }

  function formatTime(timestamp) {
    const date = new Date(timestamp);
    return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  }

  function formatDate(timestamp) {
    const date = new Date(timestamp);
    const today = new Date();
    const isToday = date.toDateString() === today.toDateString();
    
    if (isToday) {
      return formatTime(timestamp);
    }
    return date.toLocaleDateString([], { month: 'short', day: 'numeric' }) + ' ' + formatTime(timestamp);
  }

  function getTransactionDescription(transaction) {
    switch (transaction.type) {
      case 'transfer':
        return `${displayName(transaction.fromPlayerId, transaction.fromPlayerName)} → ${displayName(transaction.toPlayerId, transaction.toPlayerName)}`;
      case 'add_money':
        return `Added to ${displayName(transaction.playerId, transaction.playerName)}`;
      case 'withdraw_money':
        return `${displayName(transaction.playerId, transaction.playerName)} withdrew`;
      case 'give_2m':
        return `${displayName(transaction.playerId, transaction.playerName)} passed GO`;
      case 'give_2m_all':
        return `All players received $2M`;
      case 'player_added':
        return `${transaction.playerName} joined`;
      case 'player_removed':
        return `${transaction.playerName} left`;
      default:
        return 'Unknown';
    }
  }

  function getTransactionIcon(transaction) {
    switch (transaction.type) {
      case 'transfer':
        return '↔️';
      case 'add_money':
        return '➕';
      case 'withdraw_money':
        return '➖';
      case 'give_2m':
        return '🎯';
      case 'player_added':
        return '👤';
      case 'player_removed':
        return '👋';
      default:
        return '💰';
    }
  }

  // Save state when player names change
  $effect(() => {
    if (players.length > 0) {
      saveState();
    }
  });
</script>

<div class="app">
  <header class="header">
    <div class="header-content">
      <h1 class="logo">
        <span class="logo-icon">🏦</span>
        <span class="logo-text">Monopoly Banker</span>
      </h1>
      <div class="header-actions">
        <button class="header-btn" onclick={addPlayer} title="Add a new player">
          <span class="btn-icon">➕</span>
          <span class="btn-label">Add Player</span>
        </button>
        <button class="header-btn primary" onclick={openNewGameModal} title="Start a new game">
          <span class="btn-icon">🎮</span>
          <span class="btn-label">New Game</span>
        </button>
        <button class="header-btn danger" onclick={resetGame} title="Reset everything">
          <span class="btn-icon">🔄</span>
          <span class="btn-label">Reset</span>
        </button>
        <div class="divider"></div>
        <button class="history-toggle" onclick={toggleHistory}>
          <span class="icon">📊</span>
          <span>History</span>
          {#if transactions.length > 0}
            <span class="badge">{transactions.length}</span>
          {/if}
        </button>
      </div>
    </div>
  </header>

  <div class="main-content">
    <div class="primary-section">
      <div class="players-section">
        <h2 class="section-title">Players</h2>
        <div class="players-grid">
          {#each players as player (player.id)}
            <div class="player-card" class:expanded={selectedPlayer === player.id}>
              <div class="player-info">
                <div class="player-name-row">
                  <input
                    type="text"
                    bind:value={player.name}
                    class="player-name-input"
                    placeholder="Player name"
                  />
                  {#if players.length > 2}
                    <button
                      class="remove-btn"
                      onclick={() => removePlayer(player.id)}
                      title="Remove player"
                    >
                      ×
                    </button>
                  {/if}
                </div>
                <div class="player-balance">
                  <span class="currency">$</span>
                  <span class="amount">{formatMoney(player.money)}</span>
                </div>
              </div>

              <div class="player-quick-actions">
                <button
                  class="quick-action pass-go"
                  onclick={() => givePlayer2M(player.id)}
                  title="Passed GO - receive $2M"
                >
                  <span class="action-icon">🎯</span>
                  <span class="action-label">Pass GO</span>
                </button>
                <button
                  class="quick-action manage"
                  onclick={() => selectedPlayer = selectedPlayer === player.id ? null : player.id}
                >
                  <span class="action-icon">{selectedPlayer === player.id ? '✕' : '⚙️'}</span>
                  <span class="action-label">{selectedPlayer === player.id ? 'Close' : 'Manage'}</span>
                </button>
              </div>

              {#if selectedPlayer === player.id}
                <div class="manage-section">
                  <div class="manage-input-group">
                    <input
                      type="text"
                      value={amountDisplay}
                      oninput={handleAmountInput}
                      onblur={handleAmountBlur}
                      placeholder="Enter amount"
                      class="manage-input"
                    />
                    {#if amount > 0}
                      <div class="input-preview">${formatMoney(amount)}</div>
                    {/if}
                  </div>
                  <div class="manage-actions">
                    <button
                      class="manage-btn add"
                      onclick={() => addMoney(player.id)}
                      disabled={amount <= 0}
                    >
                      ➕ Add
                    </button>
                    <button
                      class="manage-btn withdraw"
                      onclick={() => withdrawMoney(player.id)}
                      disabled={amount <= 0 || player.money < amount}
                    >
                      ➖ Withdraw
                    </button>
                  </div>
                </div>
              {/if}
            </div>
          {/each}
        </div>
      </div>

      <div class="transfer-section">
        <h2 class="section-title">Transfer Money</h2>
        <div class="transfer-card">
          <div class="transfer-form">
            <div class="transfer-field">
              <label for="from-player">From</label>
              <select id="from-player" bind:value={transferFrom} class="transfer-select">
                <option value={null}>Select player</option>
                {#each players as player}
                  <option value={player.id}>{player.name}</option>
                {/each}
              </select>
            </div>

            <div class="transfer-arrow">→</div>

            <div class="transfer-field">
              <label for="to-player">To</label>
              <select id="to-player" bind:value={transferTo} class="transfer-select">
                <option value={null}>Select player</option>
                {#each players as player}
                  <option value={player.id}>{player.name}</option>
                {/each}
              </select>
            </div>

            <div class="transfer-field amount-field">
              <label for="transfer-amount">Amount</label>
              <input
                id="transfer-amount"
                type="text"
                value={transferAmountDisplay}
                oninput={handleTransferAmountInput}
                onblur={handleTransferAmountBlur}
                placeholder="Enter amount"
                class="transfer-input"
              />
              {#if transferAmount > 0}
                <div class="input-preview">${formatMoney(transferAmount)}</div>
              {/if}
            </div>

            <button
              class="transfer-btn"
              onclick={transfer}
              disabled={!transferFrom || !transferTo || transferAmount <= 0 || transferFrom === transferTo}
            >
              Transfer Money
            </button>
          </div>
        </div>
      </div>
    </div>

    {#if showHistory}
      <aside class="history-sidebar">
        <div class="history-header">
          <h2 class="section-title">History</h2>
          <button class="close-history" onclick={toggleHistory}>✕</button>
        </div>
        <div class="history-content">
          {#if transactions.length === 0}
            <div class="history-empty">
              <div class="empty-icon">📋</div>
              <p>No transactions yet</p>
            </div>
          {:else}
            <div class="history-list">
              {#each transactions as transaction (transaction.id)}
                <div class="history-item">
                  <div class="history-icon">{getTransactionIcon(transaction)}</div>
                  <div class="history-details">
                    <div class="history-description">{getTransactionDescription(transaction)}</div>
                    <div class="history-meta">
                      <span class="history-time">{formatDate(transaction.timestamp)}</span>
                      {#if transaction.amount}
                        <span class="history-amount">${formatMoney(transaction.amount)}</span>
                      {/if}
                    </div>
                  </div>
                </div>
              {/each}
            </div>
          {/if}
        </div>
      </aside>
    {/if}
  </div>

  {#if showNewGameModal}
    <div
      class="modal-backdrop"
      role="button"
      tabindex="0"
      onclick={closeNewGameModal}
      onkeydown={handleBackdropKeydown}
      aria-label="Close modal"
    ></div>
    <div class="modal">
      <div class="modal-header">
        <h3>Start New Game</h3>
        <button class="modal-close" onclick={closeNewGameModal}>✕</button>
      </div>
      <div class="modal-body">
        <p>Set the starting balance for all players. This will clear transaction history.</p>
        <div class="modal-input-group">
          <input
            type="text"
            value={newGameAmountDisplay}
            oninput={handleNewGameAmountInput}
            onblur={handleNewGameAmountBlur}
            placeholder="e.g., 1500000"
            class="modal-input"
          />
          {#if newGameAmount > 0}
            <div class="input-preview">Each player: ${formatMoney(newGameAmount)}</div>
          {/if}
        </div>
      </div>
      <div class="modal-footer">
        <button class="modal-btn cancel" onclick={closeNewGameModal}>Cancel</button>
        <button class="modal-btn confirm" onclick={startNewGame} disabled={newGameAmount <= 0}>
          Start Game
        </button>
      </div>
    </div>
  {/if}
</div>

<style>
  :global(*) {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  :global(body) {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
    min-height: 100vh;
    color: #e4e4e7;
    overflow-x: hidden;
  }

  .app {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
  }

  /* Header */
  .header {
    background: rgba(255, 255, 255, 0.03);
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    position: sticky;
    top: 0;
    z-index: 100;
  }

  .header-content {
    max-width: 1400px;
    margin: 0 auto;
    padding: 1.25rem 1.5rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .logo {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    font-size: 1.5rem;
    font-weight: 700;
    color: #fff;
  }

  .logo-icon {
    font-size: 1.8rem;
  }

  .header-actions {
    display: flex;
    align-items: center;
    gap: 0.625rem;
  }

  .header-btn {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 10px;
    color: #e4e4e7;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s;
  }

  .header-btn:hover {
    background: rgba(255, 255, 255, 0.12);
    transform: translateY(-1px);
  }

  .header-btn.primary {
    background: rgba(59, 130, 246, 0.15);
    border-color: rgba(59, 130, 246, 0.3);
    color: #93c5fd;
  }

  .header-btn.primary:hover {
    background: rgba(59, 130, 246, 0.25);
  }

  .header-btn.danger {
    background: rgba(248, 113, 113, 0.12);
    border-color: rgba(248, 113, 113, 0.25);
    color: #fca5a5;
  }

  .header-btn.danger:hover {
    background: rgba(248, 113, 113, 0.2);
  }

  .btn-icon {
    font-size: 1rem;
  }

  .btn-label {
    font-size: 0.875rem;
  }

  @media (max-width: 768px) {
    .header-btn .btn-label {
      display: none;
    }

    .header-btn {
      padding: 0.625rem;
      min-width: 40px;
      justify-content: center;
    }

    .btn-icon {
      font-size: 1.125rem;
    }

    .divider {
      display: none;
    }

    .history-toggle span:not(.icon):not(.badge) {
      display: none;
    }
  }

  .divider {
    width: 1px;
    height: 32px;
    background: rgba(255, 255, 255, 0.12);
    margin: 0 0.25rem;
  }

  .history-toggle {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 10px;
    color: #e4e4e7;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s;
    position: relative;
  }

  .history-toggle:hover {
    background: rgba(255, 255, 255, 0.12);
    transform: translateY(-1px);
  }

  .history-toggle .icon {
    font-size: 1.1rem;
  }

  .badge {
    background: #3b82f6;
    color: white;
    font-size: 0.7rem;
    font-weight: 600;
    padding: 0.125rem 0.4rem;
    border-radius: 10px;
    min-width: 1.25rem;
    text-align: center;
  }

  /* Main Content */
  .main-content {
    flex: 1;
    max-width: 1400px;
    width: 100%;
    margin: 0 auto;
    padding: 2rem 1.5rem;
    display: grid;
    grid-template-columns: 1fr;
    gap: 2rem;
  }

  /* Sections */
  .section-title {
    font-size: 1.25rem;
    font-weight: 600;
    color: #fff;
    margin-bottom: 1.25rem;
  }

  .players-section {
    margin-bottom: 2rem;
  }

  .players-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 1.25rem;
  }

  .player-card {
    background: rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 16px;
    padding: 1.5rem;
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .player-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 12px 28px rgba(0, 0, 0, 0.3);
    border-color: rgba(255, 255, 255, 0.15);
  }

  .player-card.expanded {
    background: rgba(59, 130, 246, 0.08);
    border-color: rgba(59, 130, 246, 0.3);
  }

  .player-info {
    margin-bottom: 1.25rem;
  }

  .player-name-row {
    display: flex;
    gap: 0.5rem;
    align-items: center;
    margin-bottom: 1rem;
  }

  .player-name-input {
    flex: 1;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 8px;
    padding: 0.625rem 0.875rem;
    font-size: 1rem;
    font-weight: 600;
    color: #fff;
    transition: all 0.2s;
  }

  .player-name-input:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.12);
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }

  .remove-btn {
    width: 32px;
    height: 32px;
    border: none;
    background: rgba(248, 113, 113, 0.15);
    color: #f87171;
    border-radius: 8px;
    font-size: 1.25rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s;
    flex-shrink: 0;
  }

  .remove-btn:hover {
    background: rgba(248, 113, 113, 0.25);
    transform: scale(1.05);
  }

  .player-balance {
    text-align: center;
    padding: 1rem;
    background: rgba(34, 197, 94, 0.08);
    border-radius: 12px;
  }

  .currency {
    font-size: 1.25rem;
    color: #86efac;
    font-weight: 500;
  }

  .amount {
    font-size: 2rem;
    font-weight: 700;
    color: #4ade80;
    margin-left: 0.25rem;
  }

  .player-quick-actions {
    display: flex;
    flex-direction: row;
    gap: 0.75rem;
    margin-top: 1rem;
  }

  .quick-action {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    padding: 0.75rem 1rem;
    background: rgba(255, 255, 255, 0.06);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 10px;
    color: #e4e4e7;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s;
  }

  .quick-action:hover {
    background: rgba(255, 255, 255, 0.1);
    transform: translateY(-2px);
  }

  .quick-action.pass-go {
    background: rgba(59, 130, 246, 0.12);
    border-color: rgba(59, 130, 246, 0.3);
  }

  .quick-action.pass-go:hover {
    background: rgba(59, 130, 246, 0.2);
  }

  .action-icon {
    font-size: 1.1rem;
  }

  .action-label {
    font-size: 0.85rem;
  }

  .manage-section {
    margin-top: 1.25rem;
    padding-top: 1.25rem;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
    animation: slideIn 0.2s ease;
  }

  @keyframes slideIn {
    from {
      opacity: 0;
      transform: translateY(-8px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .manage-input-group {
    margin-bottom: 1rem;
  }

  .manage-input {
    width: 100%;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 8px;
    padding: 0.75rem 1rem;
    font-size: 1rem;
    color: #fff;
    transition: all 0.2s;
  }

  .manage-input:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.12);
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }

  .input-preview {
    margin-top: 0.5rem;
    text-align: center;
    font-size: 0.9rem;
    color: #4ade80;
    font-weight: 600;
  }

  .manage-actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.75rem;
  }

  .manage-btn {
    padding: 0.75rem 1rem;
    border: none;
    border-radius: 8px;
    font-size: 0.9rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s;
  }

  .manage-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .manage-btn.add {
    background: rgba(34, 197, 94, 0.15);
    color: #4ade80;
    border: 1px solid rgba(34, 197, 94, 0.3);
  }

  .manage-btn.add:not(:disabled):hover {
    background: rgba(34, 197, 94, 0.25);
    transform: translateY(-2px);
  }

  .manage-btn.withdraw {
    background: rgba(248, 113, 113, 0.15);
    color: #f87171;
    border: 1px solid rgba(248, 113, 113, 0.3);
  }

  .manage-btn.withdraw:not(:disabled):hover {
    background: rgba(248, 113, 113, 0.25);
    transform: translateY(-2px);
  }

  /* Transfer Section */
  .transfer-card {
    background: rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 16px;
    padding: 2rem;
  }

  .transfer-form {
    display: grid;
    grid-template-columns: 1fr auto 1fr 1.5fr auto;
    gap: 1rem;
    align-items: end;
  }

  @media (max-width: 968px) {
    .transfer-form {
      grid-template-columns: 1fr;
      gap: 1.5rem;
    }

    .transfer-arrow {
      display: none;
    }
  }

  .transfer-field {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .transfer-field label {
    font-size: 0.875rem;
    font-weight: 600;
    color: #a1a1aa;
  }

  .transfer-select,
  .transfer-input {
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 8px;
    padding: 0.75rem 1rem;
    font-size: 1rem;
    color: #fff;
    transition: all 0.2s;
  }

  .transfer-select:focus,
  .transfer-input:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.12);
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }

  .transfer-arrow {
    font-size: 1.5rem;
    color: #a1a1aa;
    padding-bottom: 0.75rem;
  }

  .transfer-btn {
    padding: 0.75rem 1.5rem;
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    border: none;
    border-radius: 10px;
    color: white;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s;
    box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
  }

  .transfer-btn:not(:disabled):hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(59, 130, 246, 0.4);
  }

  .transfer-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  /* History Sidebar */
  .history-sidebar {
    position: fixed;
    right: 0;
    top: 0;
    height: 100vh;
    width: 380px;
    background: rgba(20, 20, 30, 0.98);
    backdrop-filter: blur(12px);
    border-left: 1px solid rgba(255, 255, 255, 0.1);
    box-shadow: -4px 0 24px rgba(0, 0, 0, 0.3);
    z-index: 110;
    display: flex;
    flex-direction: column;
    animation: slideInRight 0.3s ease;
  }

  @keyframes slideInRight {
    from {
      transform: translateX(100%);
    }
    to {
      transform: translateX(0);
    }
  }

  @media (max-width: 768px) {
    .history-sidebar {
      width: 100%;
    }
  }

  .history-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 1.5rem;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  }

  .close-history {
    width: 36px;
    height: 36px;
    border: none;
    background: rgba(255, 255, 255, 0.08);
    color: #e4e4e7;
    border-radius: 8px;
    font-size: 1.25rem;
    cursor: pointer;
    transition: all 0.2s;
  }

  .close-history:hover {
    background: rgba(255, 255, 255, 0.12);
  }

  .history-content {
    flex: 1;
    overflow-y: auto;
    padding: 1rem;
  }

  .history-empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 1rem;
    padding: 4rem 2rem;
    text-align: center;
    color: #71717a;
  }

  .empty-icon {
    font-size: 3rem;
    opacity: 0.5;
  }

  .history-list {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  .history-item {
    display: flex;
    gap: 0.875rem;
    padding: 1rem;
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid rgba(255, 255, 255, 0.08);
    border-radius: 10px;
    transition: all 0.2s;
  }

  .history-item:hover {
    background: rgba(255, 255, 255, 0.06);
    border-color: rgba(255, 255, 255, 0.12);
  }

  .history-icon {
    font-size: 1.5rem;
    flex-shrink: 0;
  }

  .history-details {
    flex: 1;
    min-width: 0;
  }

  .history-description {
    font-size: 0.95rem;
    color: #e4e4e7;
    margin-bottom: 0.375rem;
    line-height: 1.4;
  }

  .history-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
    font-size: 0.8rem;
  }

  .history-time {
    color: #71717a;
  }

  .history-amount {
    color: #4ade80;
    font-weight: 600;
  }

  /* Modal */
  .modal-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.6);
    backdrop-filter: blur(4px);
    z-index: 200;
    animation: fadeIn 0.2s ease;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }

  .modal {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(30, 30, 46, 0.98);
    backdrop-filter: blur(12px);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 16px;
    box-shadow: 0 24px 48px rgba(0, 0, 0, 0.5);
    width: min(460px, 92vw);
    z-index: 210;
    animation: modalSlideIn 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  @keyframes modalSlideIn {
    from {
      opacity: 0;
      transform: translate(-50%, -48%) scale(0.96);
    }
    to {
      opacity: 1;
      transform: translate(-50%, -50%) scale(1);
    }
  }

  .modal-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 1.5rem;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  }

  .modal-header h3 {
    font-size: 1.25rem;
    font-weight: 600;
    color: #fff;
  }

  .modal-close {
    width: 32px;
    height: 32px;
    border: none;
    background: rgba(255, 255, 255, 0.08);
    color: #e4e4e7;
    border-radius: 6px;
    font-size: 1.125rem;
    cursor: pointer;
    transition: all 0.2s;
  }

  .modal-close:hover {
    background: rgba(255, 255, 255, 0.12);
  }

  .modal-body {
    padding: 1.5rem;
  }

  .modal-body p {
    color: #a1a1aa;
    line-height: 1.6;
    margin-bottom: 1.25rem;
  }

  .modal-input-group {
    margin-bottom: 1rem;
  }

  .modal-input {
    width: 100%;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 8px;
    padding: 0.875rem 1rem;
    font-size: 1rem;
    color: #fff;
    transition: all 0.2s;
  }

  .modal-input:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.12);
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }

  .modal-footer {
    display: flex;
    gap: 0.75rem;
    padding: 1.5rem;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
  }

  .modal-btn {
    flex: 1;
    padding: 0.875rem 1.5rem;
    border: none;
    border-radius: 10px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s;
  }

  .modal-btn.cancel {
    background: rgba(255, 255, 255, 0.08);
    color: #e4e4e7;
  }

  .modal-btn.cancel:hover {
    background: rgba(255, 255, 255, 0.12);
  }

  .modal-btn.confirm {
    background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
    color: white;
    box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
  }

  .modal-btn.confirm:not(:disabled):hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(59, 130, 246, 0.4);
  }

  .modal-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
