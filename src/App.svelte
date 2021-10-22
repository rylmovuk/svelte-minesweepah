<script lang="ts">
  // import { count, sample } from "./util";
  import { chunk, filter, sampleSize, some, sumBy, times } from "lodash-es";

  type CellType = "" | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | "X";
  class Cell {
    type: CellType;
    flag = false;
    uncovered = false;

    public get row(): number {
      return ~~(this.id / columns);
    }

    public get col(): number {
      return this.id % columns;
    }

    constructor(public readonly id: number) {}

    uncover() {
      this.uncovered = true;
      if (this.type === "")
        for (const neigh of this.neighbors())
          neigh.uncovered || neigh.uncover();
    }

    show() {
      if (this.flag) return "F";
      if (this.uncovered) return this.type;
      return "";
    }

    *neighbors() {
      yield* Cell.neighbors(this.row, this.col);
    }

    static *neighbors(r: number, c: number) {
      for (const dx of [-1, 0, 1]) {
        for (const dy of [-1, 0, 1]) {
          if (dx === 0 && dy === 0) continue;
          const col = c + dx;
          if (col < 0 || col >= columns) continue;
          const row = r + dy;
          if (row < 0 || row >= rows) continue;
          yield cells[row * columns + col];
        }
      }
    }
  }

  export let columns = 24;
  export let rows = 20;
  export let totalMines = 100;

  let cells: Cell[];

  function initializeGrid() {
    cells = times(rows * columns, (i) => new Cell(i));
    // place mines
    for (const cell of sampleSize(cells, totalMines)) cell.type = "X";
    // place empty cells and numbers around mines
    for (const cell of cells) {
      if (cell.type === "X") continue;
      const minesN = filter([...cell.neighbors()], { type: "X" }).length;
      cell.type = minesN === 0 ? "" : (minesN as CellType);
    }
  }

  // Initialize on page load
  initializeGrid();

  // How many cells are still covered
  $: coveredCount = sumBy(cells, (c) => +!c.uncovered);

  // Whether any cell with a mine has been uncovered
  $: tripped = some(cells, { type: "X", uncovered: true });

  // How many flags have been placed
  $: flagCount = sumBy(cells, "flag");

  function cellClickHandler(this: HTMLButtonElement, ev: MouseEvent) {
    const cell: Cell = cells[this.dataset.cellId];
    switch (ev.button) {
      case 0: // left
        if (cell.flag) return;
        cell.uncover();
        break;
      case 1: // middle
        // Must be a covered cell surrounded with the corresponding number of flags
        const neighs = [...cell.neighbors()];
        if (
          !cell.uncovered ||
          (sumBy(neighs, "flag") as CellType) !== cell.type
        )
          return;
        for (const neigh of neighs)
          neigh.flag || neigh.uncovered || neigh.uncover();
        break;
      case 2: // right
        if (cell.uncovered) return;
        cell.flag = !cell.flag;
        break;
      default:
        return;
    }
    // `break` (not `return`) from the switch === success
    // trigger reactivity update
    cells = cells;
  }
</script>

<main>
  <div class="game">
    <div class="info">
      <span>Board size: {columns}x{rows}</span>
      <span>Mines: {flagCount}/{totalMines}</span>
      <button on:click={initializeGrid}>Restart</button>
    </div>
    <div class="grid">
      {#each chunk(cells, columns) as row}
        <div class="row">
          {#each row as cell}
            <div
              data-cell-id={cell.id}
              class="cell"
              class:covered={!cell.uncovered}
              on:mouseup={cellClickHandler}
              on:contextmenu|preventDefault
            >
              {cell.show()}
            </div>
          {/each}
        </div>
      {/each}
    </div>
    {#if tripped}
      <div class="lose">You lost :'(</div>
    {:else if coveredCount === totalMines}
      <div class="win">You won!!!</div>
    {/if}
  </div>
</main>

<style>
  main {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100%;
  }
  .game {
    display: flex;
    align-items: center;
    justify-content: center;
    flex-flow: column nowrap;
  }
  .info {
    width: 100%;
    display: flex;
    align-items: baseline;
    justify-content: space-around;
  }
  .grid {
    display: flex;
    flex-flow: column nowrap;
    line-height: 0;
  }
  .row {
    display: flex;
    flex-flow: row nowrap;
    height: 25px;
  }
  .cell {
    width: 25px;
    height: inherit;
    background: #ddd;
    text-align: center;
    line-height: 25px;
    vertical-align: middle;
    font-family: monospace;
    font-size: 20px;
    padding: 0;
    cursor: default;
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }
  .win {
    color: purple;
    font-family: "Monotype Corsiva", cursive;
    font-size: 2.5em;
  }
  .lose {
    color: magenta;
    font-family: "Comic Sans", cursive;
    font-size: 2.5em;
  }
  .covered {
    background: #23ba28;
    cursor: pointer;
  }
  .covered:hover {
    background: #7ad64f;
  }

  .grid *:nth-child(odd) > .covered:nth-child(odd),
  .grid *:nth-child(even) > .covered:nth-child(even) {
    background: #168019;
  }
  .grid *:nth-child(odd) > .covered:nth-child(odd):hover,
  .grid *:nth-child(even) > .covered:nth-child(even):hover {
    background: #72b346;
  }
</style>
