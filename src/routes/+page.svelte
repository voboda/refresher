<style>
html, p, form  {
  font-family: sans-serif;
}

form {
  border: 1px solid #555;
  padding: 1rem;
}
</style>

<script>
    import { onMount } from 'svelte';

    // Variables
    let currentTime = new Date();
    let localTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
    let berlinTimezone = 'Europe/Berlin';

    let url = 'https://tickets.events.ccc.de/38c3/'; // Default URL
    let timeInput = '11:59:59.800'; // Default Berlin time for refresh

    let ntpTime = null;
    let localStart = null;
    let message = '';
    let timeMessage = '';

    // Regex for HH:mm:ss.ms
    const timeRegex = /^([0-1]\d|2[0-3]):([0-5]\d):([0-5]\d)\.(\d{1,3})$/;

    // Fetch NTP time for Berlin
    async function fetchNtpTime() {
        try {
            const response = await fetch('https://worldtimeapi.org/api/timezone/Europe/Berlin');
            if (response.ok) {
                const data = await response.json();
                return new Date(data.utc_datetime);
            }
        } catch (error) {
            console.error('Failed to fetch Berlin time:', error);
        }
        return null;
    }

    // Format time with milliseconds
    function formatTimeWithMilliseconds(date, timeZone) {
        const options = {
            timeZone,
            hour: '2-digit',
            minute: '2-digit',
            second: '2-digit',
            hour12: false,
        };

        const formatter = new Intl.DateTimeFormat('en-US', options);
        const formattedTime = formatter.format(date);
        const milliseconds = String(date.getMilliseconds()).padStart(3, '0');

        return `${formattedTime}.${milliseconds}`;
    }

    // Convert Berlin time input to local time
    function berlinTimeToLocal(hours, minutes, seconds, milliseconds) {
        const berlinDate = new Date();
        berlinDate.setHours(hours, minutes, seconds, milliseconds);

        // Convert Berlin time to UTC
        const berlinToUTC = berlinDate.toISOString();

        // Convert UTC to local time
        const localDate = new Date(berlinToUTC);
        return localDate;
    }

    // Refresh the page at the specified Berlin time
    function refreshAt(hours, minutes, seconds, milliseconds) {
        // Convert Berlin time input to local time
        const localRefreshTime = berlinTimeToLocal(hours, minutes, seconds, milliseconds);

        const now = new Date();
        const timeout = localRefreshTime.getTime() - now.getTime();

        if (timeout <= 0) {
            // If the time is in the past, set it for the next day
            localRefreshTime.setDate(localRefreshTime.getDate() + 1);
        }

        setTimeout(() => {
            window.location.href = url;
        }, Math.max(0, timeout));

        message = `Page will load ${url} at ${formatTimeWithMilliseconds(localRefreshTime, localTimezone)} (local time).`;
    }

    // Validate time format and set timer
    function setTimer() {
        if (!timeRegex.test(timeInput)) {
            alert('Incorrect time format. Use: HH:mm:ss.ms (e.g., 11:59:59.800)');
            return;
        }

        const timeParts = timeInput.split(/[:.]/);
        const hours = parseInt(timeParts[0], 10);
        const minutes = parseInt(timeParts[1], 10);
        const seconds = parseInt(timeParts[2], 10);
        const milliseconds = parseInt(timeParts[3], 10);

        refreshAt(hours, minutes, seconds, milliseconds);
    }

    // Update clocks and timeMessage
    function updateClocks() {
        currentTime = new Date();

        // Update Local and Berlin Time
        if (ntpTime) {
            const ntpNow = new Date(ntpTime.getTime() + (currentTime - localStart));
            const diff = ntpNow.getTime() - currentTime.getTime();
            const direction = diff > 0 ? 'too slow' : 'too fast';
            timeMessage = `Note: Your clock is ${direction} by ${Math.abs(diff)} ms.`;
        } else {
            timeMessage = 'Note: Unable to fetch Berlin time. Displaying local time only.';
        }
    }

    // Initialize the app on page load
    onMount(async () => {
        // Update clocks every 50ms
        setInterval(updateClocks, 50);

        localStart = new Date();
        ntpTime = await fetchNtpTime();

        if (!ntpTime) {
            timeMessage = 'Failed to fetch Berlin time. Only local time will be displayed. Or try reloading.';
        }

        // Automatically set the default timer
        setTimer();

    });
</script>

<!-- UI -->
<div>
    <h1>Timed Page Loader with Berlin Time</h1>
    <p>Disclaimer: I made this quickly, so please test before relying on it!</p>

    <p>
        <strong>Local Time:</strong> {formatTimeWithMilliseconds(currentTime, localTimezone)} ({localTimezone})<br>
        <strong>Berlin Time:</strong> 
        {ntpTime 
            ? formatTimeWithMilliseconds(new Date(ntpTime.getTime() + (currentTime - localStart)), berlinTimezone)
            : 'Fetching Berlin time...'} 
        ({berlinTimezone})<br>
        {timeMessage}
    </p>

    <form on:submit|preventDefault={setTimer}>
        <label for="url">URL to load:</label><br>
        <input type="url" id="url" bind:value={url} required /><br><br>

        <label for="time">At what local time (HH:mm:ss.ms):</label><br>
        <input type="text" id="time" bind:value={timeInput} required /><br><br>

        <button type="submit">Set Timer</button>
    </form>

    <p>{message}</p>

    <p><a href="https://github.com/voboda/refresher">Source</a></p>
</div>

