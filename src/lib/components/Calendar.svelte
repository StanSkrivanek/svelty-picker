<script>
  import { createEventDispatcher } from 'svelte';
  import { fade } from 'svelte/transition';
  import { compute, MODE_MONTH, MODE_YEAR, MODE_DECADE, moveGrid, isLower, isGreater } from '../utils/dateUtils.js';
  import { scale } from '../utils/utils.js'

  /** @type {Date|null} */
  export let date = null;
  /** @type {Date|null} */
  export let startDate = null;
  /** @type {Date|null} */
  export let endDate = null;
  export let weekStart = 1;
  /** @type {i18nType} */
  export let i18n;
  export let enableTimeToggle = false;
  /**
   * @param {string} key
   * @param {boolean} shiftKey
   */
  export function handleGridNav(key, shiftKey) {
    if (!internalDate) {
      onClick(new Date);
      return;
    }
    /** @type {GridPosition} pos */
    let pos, diffSelection;
    switch (key) {
      case 'PageDown':
        shiftKey = true;
      case 'ArrowDown':
        if (shiftKey) return handleShiftNav(activeDate.getFullYear(), activeDate.getMonth() + 1, 1);
        diffSelection = dataset.selectionMark + 7;
        if (diffSelection >= dataset.nextFrom) {
          const tmpDate = new Date(activeDate.getFullYear(), activeDate.getUTCMonth() + 1, 28);
          let tmpData = compute(tmpDate, internalDate, currentView, i18n, weekStart);
          onChangeMonth(1);
          pos = tmpData.selectionMark !== null
            ? {
              y: Math.floor((tmpData.selectionMark + 7) / 7),
              x: (tmpData.selectionMark + 7) % 7
            }
            : {
              y: (diffSelection + 7) % 7 < tmpData.prevTo ? 1 : 0,
              x: (diffSelection + 7) % 7,
            }
          onClick(tmpData.grid[pos.y][pos.x]);
          return;
        }
        pos = moveGrid(dataset.selectionMark + 7, currentView);
        if (dataset.grid[pos.y][pos.x].getMonth() !== activeDate.getMonth()) {
          onChangeMonth(1);
        }
        onClick(dataset.grid[pos.y][pos.x]);
        break;
      case 'PageUp':
        shiftKey = true;
      case 'ArrowUp':
        if (shiftKey) return handleShiftNav(activeDate.getFullYear(), activeDate.getMonth() - 1, -1);
        diffSelection = dataset.selectionMark - 7;
        if (diffSelection <= dataset.prevTo) {
          // goto prev month
          const tmpDate = new Date(activeDate.getFullYear() + (activeDate.getMonth() > 0 ? 0 : - 1), activeDate.getMonth() > 0 ? activeDate.getMonth() -1 : 11, 1);
          const tmpData = compute(tmpDate, internalDate, currentView, i18n, weekStart);
          onChangeMonth(-1);
          pos = tmpData.selectionMark !== null
            ? {
              x: Math.floor((tmpData.selectionMark -7)  /  7),
              y: (tmpData.selectionMark - 7) % 7
            }
            : {
              x: 5,
              y: diffSelection
            }
          onClick(tmpData.grid[pos.x][pos.y]);
          return;
        }
        pos = moveGrid(dataset.selectionMark - 7, currentView);
        if (dataset.grid[pos.y][pos.x].getMonth() !== activeDate.getMonth()) {
          onChangeMonth(-1);
        }
        onClick(dataset.grid[pos.y][pos.x]);
        break;
      case 'ArrowLeft':
        if (shiftKey) return handleShiftNav(activeDate.getFullYear() - 1, activeDate.getMonth(), 1);
        pos = moveGrid(dataset.selectionMark - 1, currentView);
        if (dataset.grid[pos.y][pos.x].getMonth() !== activeDate.getMonth()) {
          onChangeMonth(-1);
        }
        onClick(dataset.grid[pos.y][pos.x]);
        break;
      case 'ArrowRight':
        if (shiftKey) return handleShiftNav(activeDate.getFullYear() + 1, activeDate.getMonth(), 1);
        pos = moveGrid(dataset.selectionMark + 1, currentView);
        if (dataset.grid[pos.y][pos.x].getMonth() !== activeDate.getMonth()) {
          onChangeMonth(1);
        }
        onClick(dataset.grid[pos.y][pos.x]);
        break;
    }
  }

  /**
   * @param {number} year
   * @param {number} month
   * @param {number} monthChange
   */
  function handleShiftNav(year, month, monthChange) {
    let tmpDate;
    let newDateDay = activeDate.getDate();
    // required when going back from 31 long month to 30, or 28/29 and to ensure month is changed, not day only
    do {
      tmpDate = new Date(year, month, newDateDay);
      newDateDay--;
    } while (tmpDate.getMonth() === activeDate.getMonth());
    const tmpData = compute(tmpDate, tmpDate, currentView, i18n, weekStart);
    const pickedDate = tmpData.grid[Math.floor(tmpData.selectionMark / 7)][tmpData.selectionMark % 7];
    if ((endDate && isGreater(pickedDate, endDate)) || (startDate && isLower(pickedDate, startDate))) return;
    onChangeMonth(monthChange);
    onClick(pickedDate);
  }

  let internalDate = date;
  let activeDate = date ? new Date(date.valueOf()) : new Date();

  $: computedStartDate = startDate
    ? new Date(startDate.getFullYear(), startDate.getMonth(), startDate.getDate(), 0,0,0,0)
    : null;

  const dispatch = createEventDispatcher();

  let currentView = MODE_MONTH;
  let viewDelta = -2;
  let viewChanged = false;
  let duration = 400;
  $: start = viewDelta < 1 ? 1.5 : 0.5;
  $: end = viewDelta < 1 ? 1 : 1.5;
  const TRANSFORM_CONST = 222;
  const TRANSFORM_DECADE_UNEVEN = 148;  // in decade view, transform constants values are changing
  let transform = TRANSFORM_CONST;  // month +/- constant
  /** @type {Function|null} */
  let onMonthTransitionTrigger = null;

  $: swapTransition = viewDelta === -2
    ? fade
    : (viewDelta !== null ? scale : () => {});

  $: {
    if (date !== internalDate) {
      internalDate = date;
      if (date) {
        activeDate = new Date(date.valueOf())
      };
      viewDelta = 1;
      viewChanged = true;
      currentView = MODE_MONTH;
    }
  }
  $: dataset = compute(activeDate, internalDate, currentView, i18n, weekStart);
  $: dayLabels = weekStart > -1
    ? i18n.daysMin.concat(i18n.daysMin).slice(weekStart, 7 + weekStart)
    : i18n.daysMin.slice(weekStart, 7 + weekStart)

  function isBetween(/** @type {number} */num) {
    return dataset.prevTo <= num && num < dataset.nextFrom;
  }

  function isDisabledDate(/** @type {Date} */ date) {
    switch (currentView) {
      case MODE_MONTH:
        if (computedStartDate && computedStartDate > date) return true;
        if (endDate && endDate <= date) return true;
        break;
      case MODE_YEAR:
        if (computedStartDate && computedStartDate.getFullYear() === date.getFullYear() && computedStartDate.getMonth() > date.getMonth()) return true;
        if (endDate && endDate.getFullYear() === date.getFullYear() && endDate.getMonth() < date.getMonth()) return true;
        break;
      case MODE_DECADE:
        if (computedStartDate && computedStartDate.getFullYear() > date.getFullYear()) return true;
        if (endDate && endDate.getFullYear() < date.getFullYear()) return true;
        break;
    }
    return false;
  }

  function onChangeMonth(/** @type {number} */ val) {

    const multiplier = currentView === MODE_DECADE
      ? 120
      : (currentView === MODE_YEAR
      ? 12
      : 1
    )
    activeDate.setMonth(activeDate.getMonth() + (val*multiplier));
    activeDate = activeDate;
    onMonthTransitionTrigger = null;
    transform = currentView === MODE_DECADE
      ? activeDate.getFullYear() % 20 >= 10 ? TRANSFORM_CONST : TRANSFORM_DECADE_UNEVEN
      : TRANSFORM_CONST;
  }

  function onTransformChangeMonth(/** @type {number} */ val) {
    if (currentView === MODE_MONTH) {
      return onChangeMonth(val);
    }
    onMonthTransitionTrigger = () => {
      onChangeMonth(val)
    };
    
    if (currentView === MODE_DECADE) {
      transform = transform === TRANSFORM_DECADE_UNEVEN
        ? (val === -1
          ? transform - TRANSFORM_CONST
          : TRANSFORM_CONST + TRANSFORM_DECADE_UNEVEN
        )
        : (val === -1
          ? transform - TRANSFORM_CONST // 0 / -74
          : transform + TRANSFORM_DECADE_UNEVEN // TRANSFORM_CONST + TRANSFORM_CONST
        )
      return;
    }
    transform = val === -1 ? transform - TRANSFORM_CONST : transform + TRANSFORM_CONST;
  }

  function onSwitchView() {
    viewDelta = -1
    viewChanged = true;
    currentView && currentView--;
    if (currentView === MODE_DECADE) {
      const isLongerMove = (Math.floor(activeDate.getFullYear() / 10) * 10) % 20 === 0;
      transform = isLongerMove ? TRANSFORM_DECADE_UNEVEN : TRANSFORM_CONST;
    }
  }


  // @ts-ignore
  function onClick(value) {
    viewDelta = 1;
    viewChanged = true;
    switch (currentView) {
      case 0:
        activeDate.setFullYear(value);
        activeDate = activeDate;
        break;
      case 1:
        activeDate.setMonth(i18n.monthsShort.indexOf(value));
        activeDate = activeDate;
        break;
      case 2:
        if (startDate && !isGreater(value, startDate)) return;
        if (endDate && !isLower(value, endDate)) return;
        const newInternalDate = new Date(value.getFullYear(), value.getMonth(), value.getDate());
        if (internalDate) {
          newInternalDate.setMinutes(internalDate.getMinutes());
          newInternalDate.setHours(internalDate.getHours());
        }
        internalDate = newInternalDate;
        dispatch('date', internalDate);
        break;
    }
    currentView < MODE_MONTH && currentView++;
    transform = TRANSFORM_CONST;  // reset transform
  }

  
  function onTransitionOut() {
    viewChanged = false;
  }
  

  function onTimeSwitch() {
    dispatch('switch', 'time');
  }

  /**
   * @param {number} currentView
   * @param {Date} activeDate
   */
  function showCaption(currentView, activeDate) {
    switch (currentView) {
      case MODE_DECADE:
        const from = [Math.floor(dataset.prevTo / 4), dataset.prevTo % 4] // y,x
        const to = [Math.floor(dataset.nextFrom / 4), (dataset.nextFrom % 4)] // y,x
        return `${dataset.years[from.shift()][from.shift()]} - ${dataset.years[to.shift()][to.shift()] - 1}`
      case MODE_YEAR:
        return activeDate.getFullYear();
      case MODE_MONTH:
        return i18n.months[activeDate.getMonth()] + ' ' + activeDate.getFullYear();
    }
  }

  $: tableCaption = i18n && showCaption(currentView, activeDate);

</script>

<div class="sdt-thead-nav">
  <button class="std-btn std-btn-header sdt-toggle-btn" on:click|preventDefault={onSwitchView}>{tableCaption}</button>
  <div class="sdt-nav-btns">
    {#if enableTimeToggle && internalDate}
    <button class="std-btn std-btn-header icon-btn sdt-time-icon" title={i18n.timeView} on:click|preventDefault={onTimeSwitch} >
      <svg class="sdt-svg" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" width="16" height="16"><path fill-rule="evenodd" d="M1.5 8a6.5 6.5 0 1113 0 6.5 6.5 0 01-13 0zM8 0a8 8 0 100 16A8 8 0 008 0zm.5 4.75a.75.75 0 00-1.5 0v3.5a.75.75 0 00.471.696l2.5 1a.75.75 0 00.557-1.392L8.5 7.742V4.75z"></path></svg>
    </button>
    {/if}
    <button class="std-btn std-btn-header icon-btn" on:click|preventDefault={() => onTransformChangeMonth(-1)}>
      <svg class="sdt-svg" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" width="24" height="24"><path d="M4.427 9.573l3.396-3.396a.25.25 0 01.354 0l3.396 3.396a.25.25 0 01-.177.427H4.604a.25.25 0 01-.177-.427z"></path></svg>
    </button>
    <button class="std-btn std-btn-header icon-btn" on:click|preventDefault={() => onTransformChangeMonth(1)}>
      <svg class="sdt-svg" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" width="24" height="24"><path d="M4.427 7.427l3.396 3.396a.25.25 0 00.354 0l3.396-3.396A.25.25 0 0011.396 7H4.604a.25.25 0 00-.177.427z"></path></svg>
    </button>
  </div>
</div>
<div class="sdt-calendar" class:is-grid={viewChanged}>
  {#if currentView === MODE_DECADE}
  <table class="sdt-table" style="max-height: 221px; height: 221px">
    <tbody in:swapTransition={{duration: duration, start: start, opacity: 1}} class="sdt-tbody-lg" out:swapTransition|local={{duration, end, start: 1}} on:outroend={onTransitionOut}
      style={`transform: translateY(-${transform}px); color: red`}
      class:animate-transition={onMonthTransitionTrigger ? true : false}
      on:transitionend={() => onMonthTransitionTrigger && onMonthTransitionTrigger()}
    >
      {#each dataset.years as row, i}
      <tr class="sdt-cal-td">
        {#each row as year, j(j)}
        <td class="sdt-cal-td" class:is-selected={i*4+j === dataset.selectionMark}>
          <button
            class="std-btn"
            class:not-current={!isBetween(i*4+j)}
            on:click|preventDefault={() => { onClick(year)}}
            disabled={isDisabledDate(new Date(year, activeDate.getMonth(), activeDate.getDate()))}
          >{year}</button>
        </td>
        {/each}
      </tr>
      {/each}
    </tbody>  
  </table>
  {/if}
  {#if currentView === MODE_YEAR}
  <table class="sdt-table">
    <tbody in:swapTransition={{duration, start, opacity: 1}} class="sdt-tbody-lg" out:swapTransition|local={{duration, end, start: 1 }} on:outroend={onTransitionOut} style={`transform: translateY(-${transform}px)`}
      class:animate-transition={onMonthTransitionTrigger ? true : false}
      on:transitionend={() => onMonthTransitionTrigger && onMonthTransitionTrigger()}
    >
      {#each dataset.months as row, i}
      <tr class="sdt-cal-td">
        {#each row as month, j(j)}
        <td class="sdt-cal-td" class:is-selected={i*4+j === dataset.selectionMark}>
          <button class="std-btn"
            class:not-current={!isBetween(i*4+j)}
            on:click|preventDefault={() => { onClick(month)}}
            disabled={isDisabledDate(new Date(activeDate.getFullYear(), i18n.monthsShort.indexOf(month), activeDate.getDate()))}
          >{month}</button>
        </td>
        {/each}
      </tr>
      {/each}
    </tbody>
  </table>
  {/if}
  {#if currentView === MODE_MONTH}
  <table class="sdt-table sdt-table-height">
    <tbody in:swapTransition={{duration, start: 0.5, opacity: 1}} out:swapTransition|local={{duration, start: Math.abs(viewDelta)}} on:outroend={onTransitionOut}>
      <tr class="sdt-cal-td">
      {#each dayLabels as header}
        <th class="sdt-cal-th">{header}</th>
      {/each}
      </tr>
      {#each dataset.grid as row, i }
      <tr>
        {#each row as currDate, j(j)}
        <td class="sdt-cal-td"
          class:sdt-today={i*7+j === dataset.todayMark}
          class:is-selected={i*7+j === dataset.selectionMark}
        >
          <button on:click|preventDefault={() => {onClick(currDate)}}
            class="std-btn  sdt-btn-day"
            class:not-current={!isBetween(i*7+j) }
            disabled={isDisabledDate(currDate)}
          >{currDate.getDate()}</button>
        </td>
        {/each}
      </tr>
      {/each}
    </tbody>
  </table>
  {/if}
</div>


<style>
.sdt-cal-td {
  padding: 0;
  font-size: 90%;
  text-align: center;
  background-color: var(--sdt-bg-main);
}
.sdt-cal-th {
  height: 24px;
}
.sdt-calendar {
  height: 221px;
  overflow: hidden;
}
.sdt-calendar.is-grid {
  display: grid;
}
.sdt-calendar.is-grid .sdt-table {
  grid-column: 1/2;
  grid-row: 1/2;
}
.sdt-table {
  width: 100%;
  border-collapse: collapse;
}
.sdt-table-height {
  height: 222px;
}
.animate-transition {
  will-change: transform;
  transition: transform .3s ease;
}
.not-current {
  opacity: 0.3;
}
.not-current:hover {
  opacity: 0.55;
}
.std-btn {
  margin: 0;
  border: 0;
  background: transparent;
  text-align: center;
  width: 100%;
  border-radius: 4px;
  cursor: pointer;
  padding: 0.3rem;
  box-sizing: border-box;
  color: var(--sdt-color);
}
.sdt-btn-day {
  max-height: 32px;
}
.std-btn[disabled] {
  cursor: not-allowed;
  opacity: 0.5;
  color: var(--sdt-disabled-date, #b22222);
}
.std-btn-header {
  width: auto;
  font-weight: bold;
  padding: 0.375rem 0.5rem;
}
.std-btn-header.icon-btn:first-of-type {
  padding-left: 0.375rem;
  padding-right: 0.375rem;
}
.std-btn-header.icon-btn {
  padding-left: 0.25rem;
  padding-right: 0.25rem;
}
.std-btn:hover {
  background-color: var(--sdt-btn-bg-hover);
}
.is-selected .std-btn {
  background-color: var(--sdt-primary);
  color: var(--sdt-color-selected, var(--sdt-bg-main));
  opacity: 0.9;
}
.std-btn-header:hover {
  background-color: var(--sdt-btn-header-bg-hover);
}
.sdt-time-icon {
  margin-right: -4px;
}
.sdt-time-icon svg {
  margin: 4px 0;
}
.sdt-tbody-lg {
  background-color: var(--sdt-bg-main);
}
.sdt-tbody-lg .std-btn {
  height: 74px;
}
.sdt-thead-nav {
  display: flex;
  margin-bottom: 0.25rem;
}
.sdt-nav-btns {
  white-space: nowrap;
}
.sdt-toggle-btn {
  width: 100%;
  text-align: left;
}
.sdt-today:before {
  box-sizing: border-box;
  position: absolute;
  content: '';
  margin-left: 4px;
  margin-top: 4px;
  border-left: 4px solid var(--sdt-shadow);
  border-top: 4px solid var(--sdt-shadow);
  border-bottom: 4px solid transparent;
  border-right: 4px solid transparent;
  border-radius: 2px;
  height: 4px;
  z-index: 2;
}
.sdt-svg {
  fill: var(--sdt-color);
}
.sdt-today:hover:before {
  border-left-color: var(--sdt-primary);
  border-top-color: var(--sdt-primary);
}
.is-selected.sdt-today:before {
  border-left-color: #eee;
  border-top-color: #eee;
}
</style>