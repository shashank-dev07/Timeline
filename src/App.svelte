<script>
  import { onMount } from 'svelte';

  // Arrays for months, quarters, and years
  const months = [
    'January','February','March','April',
    'May','June','July','August',
    'September','October','November','December'
  ];
  const quarters = ['Q1','Q2','Q3','Q4'];
  const years = Array.from(
    { length: 2030 - 2020 + 1 },
    (_, i) => (2020 + i).toString()
  );

  

  // Timeline visibility state
  let isTimelineVisible = false;
  let activeOption = null; // Track which option is clicked
  let isRangeCardVisible = false;
  let startRangeInput;
  // Helper: current date indexes
  const currentDate = new Date();
  const currentMonth = months[currentDate.getMonth()];
  const currentYear = currentDate.getFullYear();
  const currentQuarter = `Q${Math.floor(currentDate.getMonth() / 3) + 1}`;
let isScrolling = false;
let scrollInterval;
let scrollSpeed = 20;
let isDropdownOpen = false;

  // Build timelines
  function generateMixedTimeline(startYear, endYear) {
    const tl = [];
    for (let y = startYear; y <= endYear; y++) {
      for (let m = 0; m < 12; m++) {
        tl.push({ type: 'month', value: months[m], year: y });
      }
      tl.push({ type: 'year', value: y.toString(), year: y });
    }
    return tl;
  }
  
  function generateQuartersTimeline(startYear, endYear) {
    const tl = [];
    for (let y = startYear; y <= endYear; y++) {
      for (let q = 0; q < 4; q++) {
        tl.push({ type: 'quarter', value: quarters[q], year: y });
      }
      tl.push({ type: 'year', value: y.toString(), year: y });
    }
    return tl;
  }

  const mixedTimeline = generateMixedTimeline(2020, 2030);
  const quartersTimeline = generateQuartersTimeline(2020, 2030);

  // Compute "today" positions
  const currentMonthIndex = mixedTimeline.findIndex(
    item => item.type === 'month' && item.year === currentYear && item.value === currentMonth
  );
  const currentQuarterIndex = quartersTimeline.findIndex(
    item => item.type === 'quarter' && item.year === currentYear && item.value === currentQuarter
  );
  const currentYearIndex = years.indexOf(currentYear.toString());

  // View state
  let selectedView = 'quarters'; // 'months' | 'quarters' | 'years'

  // Initialize startIndex/endIndex placeholder (will be set properly in initializeCurrentTimePointers)
  let startIndex = 0;
  let endIndex = 0;

  let dragging = null;
  let timelineContainer;
  let optionElements = {};
  let timelineDropdown;
  let viewSelector;

  // Input-bound values
  let startInput = '';
  let endInput = '';

  function initializeCurrentTimePointers() {
    if (selectedView === 'months') {
      startIndex = Math.max(0, currentMonthIndex);
      endIndex = Math.min(mixedTimeline.length - 1, currentMonthIndex + 2);
    } else if (selectedView === 'quarters') {
      startIndex = Math.max(0, currentQuarterIndex);
      endIndex = Math.min(quartersTimeline.length - 1, currentQuarterIndex + 2);
    } else {
      startIndex = Math.max(0, currentYearIndex);
      endIndex = Math.min(years.length - 1, currentYearIndex + 1);
    }
  }

 function toggleTimeline(option) {
  if (activeOption === option && isTimelineVisible) {
    isTimelineVisible = false;
    activeOption = null;
  } else {
    isTimelineVisible = true;
    activeOption = option;
    selectedView = option;
    // Keep range card visible if it was visible
    if (isRangeCardVisible) {
      setTimeout(() => {
        updateRangeInputs();
      }, 50);
    }
    setTimeout(positionTimelineDropdown, 10);
    setTimeout(scrollToSelection, 20);
  }
}
  
  function positionTimelineDropdown() {
    if (!timelineDropdown || !activeOption || !optionElements[activeOption] || !viewSelector) return;
    
    // Get the view selector position
    const viewSelectorRect = viewSelector.getBoundingClientRect();
    
    // Set the width to match the view selector width
    timelineDropdown.style.width = '100%';
    timelineDropdown.style.left = '0';
    
    // Ensure the dropdown is visible and properly positioned
    if (window.innerWidth < 768) {
      // For mobile, make it full width
      timelineDropdown.style.maxWidth = '100%';
    } else {
      // For desktop, limit width and center
      timelineDropdown.style.maxWidth = '800px';
      timelineDropdown.style.left = '50%';
      timelineDropdown.style.transform = 'translateX(-50%)';
    }
  }

  function saveState() {
    if (typeof localStorage === 'undefined') return;
    const state = { view: selectedView };
    if (selectedView === 'months') Object.assign(state, { monthsStart: startIndex, monthsEnd: endIndex });
    if (selectedView === 'quarters') Object.assign(state, { quartersStart: startIndex, quartersEnd: endIndex });
    if (selectedView === 'years') Object.assign(state, { yearsStart: startIndex, yearsEnd: endIndex });
    localStorage.setItem('timelineState', JSON.stringify(state));
  }

  function loadSavedState() {
    if (typeof localStorage === 'undefined') return false;
    const saved = localStorage.getItem('timelineState');
    if (!saved) return false;
    const s = JSON.parse(saved);
    selectedView = s.view;
    if (selectedView === 'months' && s.monthsStart != null) {
      startIndex = s.monthsStart; endIndex = s.monthsEnd;
    } else if (selectedView === 'quarters' && s.quartersStart != null) {
      startIndex = s.quartersStart; endIndex = s.quartersEnd;
    } else if (selectedView === 'years' && s.yearsStart != null) {
      startIndex = s.yearsStart; endIndex = s.yearsEnd;
    } else {
      initializeCurrentTimePointers();
    }
    return true;
  }

  $: visibleItems =
    selectedView === 'months'
      ? mixedTimeline
      : selectedView === 'quarters'
        ? quartersTimeline
        : years.map(y => ({ type: 'year', value: y, year: parseInt(y) }));

  // When view changes, reset the indexes to current date
  $: {
    selectedView;
    initializeCurrentTimePointers();
    setTimeout(scrollToSelection, 10);
  }

  $: startInput = formatItemDisplay(visibleItems[startIndex]);
  $: endInput = formatItemDisplay(visibleItems[endIndex]);

  function formatItemDisplay(item) {
    return item.type === 'quarter'
      ? `${item.value} ${item.year}`
      : item.value + (item.type === 'month' ? ` ${item.year}` : '');
  }
function updateRangeInputs() {
  if (startIndex !== null && endIndex !== null) {
    startInput = formatItemDisplay(visibleItems[startIndex]);
    endInput = formatItemDisplay(visibleItems[endIndex]);
  }
}

function setStartByValue() {
  const idx = visibleItems.findIndex(item => formatItemDisplay(item) === startInput);
  if (idx >= 0 && idx <= endIndex) {
    startIndex = idx;
    isTimelineVisible = true; // Keep timeline visible
    setTimeout(scrollToSelection, 10);
    saveState();
  }
}

function setEndByValue() {
  const idx = visibleItems.findIndex(item => formatItemDisplay(item) === endInput);
  if (idx >= startIndex && idx >= 0) {
    endIndex = idx;
    isTimelineVisible = true; // Keep timeline visible
    setTimeout(scrollToSelection, 10);
    saveState();
  }
}
  function scrollToSelection() {
    if (!timelineContainer) return;
    const el = timelineContainer.querySelector('.selected');
    if (!el) return;
    const cw = timelineContainer.offsetWidth;
    const left = el.offsetLeft;
    const ew = el.offsetWidth;
    timelineContainer.scrollLeft = left - cw/2 + ew/2;
  }

  // Close dropdown when clicking outside
// Update the handleClickOutside function
function handleClickOutside(event) {
  // Check if click is inside the range card or its inputs
  const rangeCard = document.querySelector('.range-card');
  const isRangeCardClick = rangeCard && rangeCard.contains(event.target);
  const addRangeButton = document.querySelector('.add-range');
  const isAddRangeClick = addRangeButton && addRangeButton.contains(event.target);
  
  // Check if click is on input fields or their container
  const isInputClick = event.target.tagName === 'INPUT' || 
                      event.target.tagName === 'LABEL' ||
                      event.target.classList.contains('input-container');

  // Get all view options
  let clickedOption = false;
  for (const view in optionElements) {
    if (optionElements[view].contains(event.target)) {
      clickedOption = true;
      break;
    }
  }
   if (!event.target.closest('.relative')) {
    isDropdownOpen = false;
  }
  // Only close if click is outside everything
  if (!timelineDropdown?.contains(event.target) && 
      !isRangeCardClick && 
      !clickedOption && 
      !isAddRangeClick &&
      !isInputClick) {
    isTimelineVisible = false;
    isRangeCardVisible = false;
    activeOption = null;
  }
}
  onMount(() => {
    if (!loadSavedState()) initializeCurrentTimePointers();
    setTimeout(scrollToSelection, 10);
    
    // Recalculate positions on resize
    window.addEventListener('resize', positionTimelineDropdown);
    
    // Add event listener for clicks outside the dropdown
    document.addEventListener('click', handleClickOutside);
    
    return () => {
      document.removeEventListener('click', handleClickOutside);
      window.removeEventListener('resize', positionTimelineDropdown);
    };
  });

$: {
  if (isRangeCardVisible && startIndex !== null && endIndex !== null) {
    startInput = formatItemDisplay(visibleItems[startIndex]);
    endInput = formatItemDisplay(visibleItems[endIndex]);
  }
}

  function handleMouseDown(type) { dragging = type; }
function handleMouseUp() {
  dragging = null;
  if (isScrolling) {
    clearInterval(scrollInterval);
    isScrolling = false;
  }
  saveState();
  setTimeout(scrollToSelection, 10);
}

function handleMouseMove(idx, event) {
  if (!dragging) return;

  const container = timelineContainer;
  const containerRect = container.getBoundingClientRect();
  const rightEdge = containerRect.right - 20;
  const leftEdge = containerRect.left + 20;
  const elements = container.querySelectorAll('.item');
  console.log('elements', elements);
  if (dragging === 'end' && event.clientX >= rightEdge) {
    if (!isScrolling) {
      isScrolling = true;
      scrollInterval = setInterval(() => {
        container.scrollLeft += scrollSpeed + 30;

        const nextIdx = findItemAtRightEdge(container, elements);
        if (nextIdx > endIndex && nextIdx < elements.length) {
          endIndex = nextIdx;
          if (isRangeCardVisible) {
            endInput = formatItemDisplay(visibleItems[endIndex]);
          }
          saveState();
        }
      }, 16);
    }
  }
  else if (dragging === 'start' && event.clientX <= leftEdge) {
    if (!isScrolling) {
      isScrolling = true;
      scrollInterval = setInterval(() => {
        container.scrollLeft -= scrollSpeed;

        const firstVisibleIdx = findItemAtLeftEdge(container, elements);
        if (firstVisibleIdx < startIndex && firstVisibleIdx >= 0) {
          startIndex = firstVisibleIdx;
          if (isRangeCardVisible) {
            startInput = formatItemDisplay(visibleItems[startIndex]);
          }
          saveState();
        }
      }, 16);
    }
  }
  else {
    if (isScrolling) {
      clearInterval(scrollInterval);
      isScrolling = false;
    }

    if (dragging === 'start' && idx <= endIndex) {
      startIndex = idx;
      if (isRangeCardVisible) {
        startInput = formatItemDisplay(visibleItems[idx]);
      }
      saveState();
    }
    if (dragging === 'end' && idx >= startIndex) {
      endIndex = idx;
      if (isRangeCardVisible) {
        endInput = formatItemDisplay(visibleItems[idx]);
      }
      saveState();
    }
  }
}


// Helper function to find the item at the right edge of the container

function findItemAtRightEdge(container, elements) {
  const containerRect = container.getBoundingClientRect();
  let rightMostVisibleIdx = elements.length - 1;

  for (let i = 0; i < elements.length; i++) {
    const rect = elements[i].getBoundingClientRect();

    if (rect.right > containerRect.right) {
      // Immediately return the index of the first partially hidden element
      return i;
    } else if (rect.left < containerRect.right) {
      rightMostVisibleIdx = i;
    }
  }

  return rightMostVisibleIdx;
}


// Helper function to find the item at the left edge of the container
function findItemAtLeftEdge(container, elements) {
  const containerRect = container.getBoundingClientRect();
  
  for (let i = 0; i < elements.length; i++) {
    const el = elements[i];
    const rect = el.getBoundingClientRect();
    
    // If this element is partially visible at the left edge
    if (rect.left < containerRect.left && rect.right > containerRect.left) {
      return i;
    }
  }
  
  return 0; // Default to first element if none found
}
  // Calculate range frame position dynamically
  function getRangeFrameStyle() {
    if (!timelineContainer) return '';
    
    const startEl = timelineContainer.querySelector(`.item:nth-child(${startIndex + 1})`);
    const endEl = timelineContainer.querySelector(`.item:nth-child(${endIndex + 1})`);
    
    if (!startEl || !endEl) return '';
    
    const left = startEl.offsetLeft - 4;
    const width = (endEl.offsetLeft - startEl.offsetLeft) + endEl.offsetWidth + 8;
    
    return `left: ${left}px; width: ${width}px;`;
  }
</script>

<style>
  :root {
    --bg-container: #f8fafc;
    --item-base: #e2e8f0;
    --item-hover-brightness: 1.1;
    --transition-fast: 0.2s ease;
    --transition-medium: 0.3s ease;
    --color-month: #c5cdf5;
    --color-quarter: #caf0f6;
    --color-year: hsl(0, 54%, 97%);
    --color-selected-gradient: linear-gradient(135deg, #4f46e5, #4338ca);
    --scrollbar-track: #e3e3e3;
    --scrollbar-thumb: #3b82f6;
  }

  body {
    background: var(--bg-container);
    font-family: 'Work Sans', sans-serif;
    font-weight: 300;
    margin: 0;
    padding: 0;
  }

  .item {
    position: relative;
    min-width: 4rem;
    padding: 0.2rem 1rem;
    color:#000000;
    background: var(--item-base);
    border-radius: 6px;
    text-align: center;
    cursor: grab;
    transition: background var(--transition-medium), transform var(--transition-fast);
    cursor: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='24' height='24' viewBox='0 0 24 24' fill='%233b82f6'%3E%3Ccircle cx='12' cy='12' r='6'/%3E%3C/svg%3E") 12 12, auto;
  }
  
  .item:active {
    cursor: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='24' height='24' viewBox='0 0 24 24' fill='%232563eb'%3E%3Ccircle cx='12' cy='12' r='8'/%3E%3C/svg%3E") 12 12, auto;
  }
  
  .item:hover {
    filter: brightness(var(--item-hover-brightness));
    transform: translateY(-2px);
  }

  /* view-specific colors */
  .item.month { background: var(--color-month); }
  .item.quarter { background: var(--color-quarter); }
  .item.year { background: var(--color-year); }


.add-range {
    margin-left: 1rem;
    background: #3b82f6;
    color: white;
  }

  .add-range:hover {
    background: #2563eb;
    color: rgb(0, 0, 0);
  }


    .select-none::-webkit-scrollbar {
    height: 2px;  /* Reduced height for horizontal scrollbar */
  }

  .select-none::-webkit-scrollbar-track {
    background: var(--scrollbar-track);
    border-radius: 2px;
  }

  .select-none::-webkit-scrollbar-thumb {
    background: var(--scrollbar-thumb);
    border-radius: 2px;
    transition: background 0.2s ease;
  }

  .select-none::-webkit-scrollbar-thumb:hover {
    background: #2563eb;  /* Darker blue on hover */
  }

  /* For Firefox */
  .select-none {
    scrollbar-width: thin;
    scrollbar-color: var(--scrollbar-thumb) var(--scrollbar-track);
  }


  /* Responsive adjustments */
  @media (max-width: 768px) {
    .view-nav {
      flex-direction: column;
      width: 100%;
    }
    
    .view-option {
      width: 100%;
    }
    
    .timeline-dropdown {
      width: 100%;
      left: 0;
      transform: translateY(-20px);
    }
    
    .timeline-dropdown.visible {
      transform: translateY(0);
    }
    
    .input-container {
      flex-direction: column;
    }
  }
</style>

<div class="flex" >
  <div class="flex items-center justify-center gap-4 w-full relative" bind:this={viewSelector}>
<!-- Replace the existing nav element with this dropdown -->
<div class="relative w-30 mx-auto">
  <button 
    class="w-full  px-0 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-md shadow-sm hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
    on:click={() => isDropdownOpen = !isDropdownOpen}
  >
    {selectedView === 'months' ? 'Monthly' : selectedView === 'quarters' ? 'Quarterly' : 'Yearly'}
    <!-- Dropdown arrow -->
    <span class="absolute inset-y-0 right-0 flex items-center pr-0 pointer-events-none transition-transform duration-10"  style="transform: rotate({isDropdownOpen ? '180deg' : '0deg'})">
      <svg class="w-5 h-5 text-gray-400" viewBox="0 0 20 20" fill="currentColor">
        <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
      </svg>
    </span>
  </button>

  {#if isDropdownOpen}
    <div 
      class="absolute z-10 w-full mt-1 bg-white rounded-md shadow-lg"
      on:click|stopPropagation
    >
      <div class="py-1">
        <button
          class="block w-full px-2 py-2 text-sm text-gray-700 hover:bg-blue-50 hover:text-blue-500 {selectedView === 'months' ? 'bg-blue-50 text-blue-500' : ''}"
          on:click={() => {
            toggleTimeline('months');
            isDropdownOpen = false;
          }}
        >
          Monthly
        </button>
        <button
          class="block w-full px-2 py-2 text-sm text-gray-700 hover:bg-blue-50 hover:text-blue-500 {selectedView === 'quarters' ? 'bg-blue-50 text-blue-500' : ''}"
          on:click={() => {
            toggleTimeline('quarters');
            isDropdownOpen = false;
          }}
        >
          Quarterly
        </button>
        <button
          class="block w-full px-2 py-2 text-sm text-gray-700 hover:bg-blue-50 hover:text-blue-500 {selectedView === 'years' ? 'bg-blue-50 text-blue-500' : ''}"
          on:click={() => {
            toggleTimeline('years');
            isDropdownOpen = false;
          }}
        >
          Yearly
        </button>
      </div>
    </div>
  {/if}
</div>


    <!-- Timeline Dropdown -->
     <div 
    class="flex-1 top-0 left-0 right-0 mt-0 bg-white rounded-2xl shadow-lg transform transition-all duration-300 overflow-hidden
     opacity-100 translate-y-0 visible "
    bind:this={timelineDropdown}
  >
      <div   class="select-none overflow-x-auto scroll-smooth py-0 relative cursor-default w-full"
 bind:this={timelineContainer} on:mouseup={handleMouseUp} on:mouseleave={handleMouseUp}>
<div class="flex gap-2 p-1 bg-white rounded-lg shadow-sm relative w-max mx-auto">
          {#each visibleItems as item, idx}
            <div
              class="item {item.type} {idx >= startIndex && idx <= endIndex ? 'selected' : ''}"
              on:mousedown={() => handleMouseDown(idx === startIndex ? 'start' : idx === endIndex ? 'end' : '')}
              on:mousemove={() => handleMouseMove(idx,event)}
            >
              {formatItemDisplay(item)}
              {#if idx === startIndex || idx === endIndex}
                <div 
    class="absolute w-1 -top-0 -bottom-0 bg-gray-400 pointer-events-none transition-all duration-200 ease-in-out z-20 rounded-sm
    {idx === startIndex ? '-left-0.5' : '-right-0.5'}
    hover:bg-white hover:shadow-[0_0_0_2px_rgba(59,130,246,0.3)]"
    data-type={idx === startIndex ? 'start' : 'end'}
    on:mousedown={() => handleMouseDown(idx === startIndex ? 'start' : 'end')}
  ></div>
              {/if}
            </div>
          {/each}
          
          <!-- Add the range frame - now using a function to calculate style -->
          {#if startIndex !== null && endIndex !== null}
            <div 
  class="absolute inset-y-0 pointer-events-none bg-gray-400/20 border-2 border-blue-500 rounded-lg transition-all duration-200 ease-in-out z-0"
              style={getRangeFrameStyle()}
            ></div>
          {/if}
        </div>
      </div>
    </div>
  </div>
</div>