<script lang="ts">
  import { writable } from "svelte/store";
  
  let currentDate = '';
  let futureDate = '';
  let currentTime = '00:00';
  let futureTime = '00:00';
  let daysPerMonth = 28;
  let speedMultiplier = 1.0;

  let realDuration = '';
  let arrivalTime = '';

  let remainingSeconds = 0;
  let arrivalTimestamp = 0;
  let intervalId: number;

  const MONTHS: Record<string, number> = {
    jan: 1, january: 1,
    feb: 2, february: 2,
    mar: 3, march: 3,
    apr: 4, april: 4,
    may: 5,
    jun: 6, june: 6,
    jul: 7, july: 7,
    aug: 8, august: 8,
    sep: 9, september: 9,
    oct: 10, october: 10,
    nov: 11, november: 11,
    dec: 12, december: 12
  };
  
  const toastMessage = writable('');

  function showToast(message: string) {
    toastMessage.set(message);
    setTimeout(() => toastMessage.set(''), 8000);
  }

  function parseDate(dateStr: string): [number, number] {
    const parts = dateStr.replace('-', ' ').trim().toLowerCase().split(/\s+/);
    if (parts.length !== 2) throw new Error(`Invalid date format: ${dateStr}`);
    const month = MONTHS[parts[0]];
    if (!month) throw new Error(`Invalid month: ${parts[0]}`);
    const day = parseInt(parts[1], 10);
    if (isNaN(day) || day < 1 || day > daysPerMonth) {
      throw new Error(`Invalid day: ${parts[1]}`);
    }
    return [month, day];
  }

  function timeToMinutes(timeStr: string): number {
    const [hoursStr, minutesStr] = timeStr.split(':');
    const hours = parseInt(hoursStr, 10);
    const minutes = parseInt(minutesStr, 10);
    if (isNaN(hours) || isNaN(minutes) || hours > 23 || minutes > 59) {
      throw new Error(`Invalid time: ${timeStr}`);
    }
    return hours * 60 + minutes;
  }

  function totalMinutes(month: number, day: number, minutes: number): number {
    const totalDays = (month - 1) * daysPerMonth + (day - 1);
    return totalDays * 1440 + minutes; // 1440 minutes per day
  }

  function formatDuration(minutes: number): string {
    const days = Math.floor(minutes / (60 * 24));
    const hours = Math.floor((minutes % (60 * 24)) / 60);
    const mins = minutes % 60;
    return [
      days ? `${days}d` : '',
      hours ? `${hours}h` : '',
      mins ? `${mins}m` : ''
    ].filter(Boolean).join(' ') || '0m';
  }

  function calculate() {
    try {
      const [curMonth, curDay] = parseDate(currentDate);
      const [futMonth, futDay] = parseDate(futureDate);

      const curMinutes = timeToMinutes(currentTime);
      const futMinutes = timeToMinutes(futureTime);

      let curTotal = totalMinutes(curMonth, curDay, curMinutes);
      let futTotal = totalMinutes(futMonth, futDay, futMinutes);

      // Wrap to next year if future is "before" current
      if (futTotal < curTotal) {
        futTotal += daysPerMonth * 12 * 1440;
      }

      const fsMinutes = futTotal - curTotal;
      const realMinutes = Math.floor(fsMinutes / speedMultiplier);

      const now = new Date();
      const future = new Date(now.getTime() + realMinutes * 60 * 1000);

      realDuration = formatDuration(realMinutes);
      arrivalTime = future.toLocaleString('en-US', {
        month: 'short',
        day: '2-digit',
        hour: 'numeric',
        minute: '2-digit',
        hour12: true
      });

      arrivalTimestamp = future.getTime();
      startCountdown();
    } catch (e: any) {
      showToast(e.message);
      realDuration = '';
      arrivalTime = '';
    }
  }

  function startCountdown() {
    clearInterval(intervalId); // Clear any previous timer

    intervalId = setInterval(() => {
      const now = Date.now();
      const diff = Math.max(0, Math.floor((arrivalTimestamp - now) / 1000));
      remainingSeconds = diff;

      if (diff <= 0) {
        clearInterval(intervalId);
      }
    }, 1000);
  }

  function formatRemaining(seconds: number): string {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs}h ${mins}m ${secs}s`;
  }
</script>

<main class="p-8">
  <div class="border-b border-gray-200 pb-5">
    <h3 class="text-base font-semibold text-gray-900">Farming Simulator Time Calculator</h3>
    <p class="mt-2 max-w-4xl text-sm text-gray-500">Calculate real-world time needed to reach a future date in FS25</p>
  </div>
  <div class="flex gap-6 pt-8 border-b border-gray-200 pb-5">
    <div>
      <label for="currentDate" class="block text-sm/6 font-medium text-gray-900">Current Date</label>
      <div class="mt-2">
        <input bind:value={currentDate} type="text" name="currentDate" id="currentDate" class="block w-full rounded-md bg-white px-3 py-1.5 text-base text-gray-900 outline-1 -outline-offset-1 outline-gray-300 placeholder:text-gray-400 focus:outline-2 focus:-outline-offset-2 focus:outline-blue-600 sm:text-sm/6">
      </div>
      <p class="mt-2 text-sm text-gray-500">ex. mar-6, mar 6,</p>
      <p class="text-sm text-gray-500">march-06, march-6</p>
    </div>
    <div>
      <label for="currentTime" class="block text-sm/6 font-medium text-gray-900">Current Time</label>
      <div class="mt-2">
        <input bind:value={currentTime} type="text" name="currentTime" id="currentTime" class="block w-full rounded-md bg-white px-3 py-1.5 text-base text-gray-900 outline-1 -outline-offset-1 outline-gray-300 placeholder:text-gray-400 focus:outline-2 focus:-outline-offset-2 focus:outline-blue-600 sm:text-sm/6" placeholder="00:00">
      </div>
      <p class="mt-2 text-sm text-gray-500">24 hour format,</p>
      <p class="text-sm text-gray-500">ex. 04:30, 4:30, 13:20</p>
    </div>
    <div>
      <label for="futureDate" class="block text-sm/6 font-medium text-gray-900">Future Date</label>
      <div class="mt-2">
        <input bind:value={futureDate} type="text" name="futureDate" id="futureDate" class="block w-full rounded-md bg-white px-3 py-1.5 text-base text-gray-900 outline-1 -outline-offset-1 outline-gray-300 placeholder:text-gray-400 focus:outline-2 focus:-outline-offset-2 focus:outline-blue-600 sm:text-sm/6">
      </div>
      <p class="mt-2 text-sm text-gray-500">ex. mar-21, mar 21,</p>
      <p class="text-sm text-gray-500">march-21, march 21</p>
    </div>
    <div>
      <label for="futureTime" class="block text-sm/6 font-medium text-gray-900">Future Time</label>
      <div class="mt-2">
        <input bind:value={futureTime} type="text" name="futureTime" id="futureTime" class="block w-full rounded-md bg-white px-3 py-1.5 text-base text-gray-900 outline-1 -outline-offset-1 outline-gray-300 placeholder:text-gray-400 focus:outline-2 focus:-outline-offset-2 focus:outline-blue-600 sm:text-sm/6" placeholder="00:00">
      </div>
      <p class="mt-2 text-sm text-gray-500">24 hour format,</p>
      <p class="text-sm text-gray-500">ex. 04:30, 4:30, 13:20</p>
    </div>
    <div>
      <label for="daysPerMonth" class="block text-sm/6 font-medium text-gray-900">Days Per Month</label>
      <div class="mt-2">
        <input bind:value={daysPerMonth} type="number" name="daysPerMonth" id="daysPerMonth" class="block w-full rounded-md bg-white px-3 py-1.5 text-base text-gray-900 outline-1 -outline-offset-1 outline-gray-300 placeholder:text-gray-400 focus:outline-2 focus:-outline-offset-2 focus:outline-blue-600 sm:text-sm/6" placeholder="28">
      </div>
    </div>
    <div>
      <label for="speed" class="block text-sm/6 font-medium text-gray-900">Speed Multiplier</label>
      <div class="mt-2">
        <input bind:value={speedMultiplier} type="number" name="speedMultiplier" id="speedMultiplier" class="block w-full rounded-md bg-white px-3 py-1.5 text-base text-gray-900 outline-1 -outline-offset-1 outline-gray-300 placeholder:text-gray-400 focus:outline-2 focus:-outline-offset-2 focus:outline-blue-600 sm:text-sm/6" placeholder="1.0">
      </div>
    </div>
    <div class="pt-8">
      <button on:click={calculate} type="button" class="rounded-md bg-blue-600 px-3 py-2 text-sm font-semibold text-white shadow-xs hover:bg-blue-500 focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-blue-600">Calculate</button>
    </div>
  </div>
  <div>
    <div class="pt-8">
      {#if realDuration}
        <div class="rounded-lg bg-gray-50 p-6 shadow">
          <p class="text-lg font-semibold text-gray-800">Real time until future date: {realDuration}</p>
          <p class="text-md text-gray-600 mt-2">The real time will be: {arrivalTime}</p>
          {#if remainingSeconds > 0}
            <p class="mt-2 text-lg font-mono text-blue-600">Countdown: {formatRemaining(remainingSeconds)}</p>
          {/if}
        </div>
      {/if}
    </div>
  </div>
  {#if $toastMessage}
    <div class="fixed bottom-4 right-4 rounded-md bg-red-600 text-white px-4 py-2 shadow-lg animate-fade-in">
      {$toastMessage}
    </div>
  {/if}
</main>

<style>
  @keyframes fade-in {
    from { opacity: 0; transform: translateY(20px); }
    to { opacity: 1; transform: translateY(0); }
  }
  .animate-fade-in {
    animation: fade-in 0.3s ease-out forwards;
  }
</style>
