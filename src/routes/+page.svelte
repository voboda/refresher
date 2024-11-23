<style>
html, p, h1, form {
  font-family: sans-serif;
}

form {
border: 1px solid #000;
padding: 1em;
}
</style>
<script>
    import { onMount } from 'svelte';

    // Variables
    let localClock = '';
    let localTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
    let berlinClock = 'Fetching Berlin time...';
    let berlinTimezone = 'Europe/Berlin';
    let timeDifference = 'Calculating...';
    let message = '';

    let url = 'https://tickets.events.ccc.de/38c3/'; // Default URL
    let timeInput = '11:59:59.800'; // Default refresh time
    let ntpTime = null;
    let localStart = null;


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

    // Update clocks and calculate time difference
    function updateClocks() {
        const now = new Date();
        localClock = formatTimeWithMilliseconds(now, localTimezone);

        if (ntpTime) {
            const ntpNow = new Date(ntpTime.getTime() + (now - localStart));
            berlinClock = formatTimeWithMilliseconds(ntpNow, berlinTimezone);

            // Calculate and display the time difference in milliseconds
            const diff = ntpNow.getTime() - now.getTime();
            const direction = diff > 0 ? 'too slow' : 'too fast';
            timeDifference = `Note: your clock is ${direction} by ${Math.abs(diff)} ms`;
        } else {
            berlinClock = 'Fetching Berlin time...';
            timeDifference = 'Note: Unable to calculate time difference.';
        }
    }

    // Refresh the page at the specified time
    function refreshAt(hours, minutes, seconds, milliseconds) {
        const now = new Date();
        const then = new Date();

        if (
            now.getHours() > hours ||
            (now.getHours() === hours && now.getMinutes() > minutes) ||
            (now.getHours() === hours && now.getMinutes() === minutes && now.getSeconds() >= seconds)
        ) {
            then.setDate(now.getDate() + 1);
        }

        then.setHours(hours);
        then.setMinutes(minutes);
        then.setSeconds(seconds);
        then.setMilliseconds(milliseconds);

        const timeout = then.getTime() - now.getTime();
        setTimeout(() => {
            window.location.href = url;
        }, timeout);

        message = `Page will load ${url} at ${formatTimeWithMilliseconds(then, localTimezone)} (local time).`;
    }

    // Set timer and submit default values on page load
    function setPageLoadTimer() {
        const timeParts = timeInput.split(/[:.]/);
        const hours = parseInt(timeParts[0], 10);
        const minutes = parseInt(timeParts[1], 10);
        const seconds = parseInt(timeParts[2], 10);
        const milliseconds = parseInt(timeParts[3], 10);

        refreshAt(hours, minutes, seconds, milliseconds);
    }

    // Initialize the app on page load
    onMount(async () => {
        ntpTime = await fetchNtpTime();
        localStart = new Date();

        if (!ntpTime) {
            alert('Failed to fetch Berlin time. Only local time will be displayed.');
        }

        // Start updating the clocks every 50ms
        setInterval(updateClocks, 50);

        // Set the default timer
        setPageLoadTimer();
    });
</script>

<!-- UI -->
<div>
    <h1>Load the CCC ticket page at a specific time</h1>

    <p>
        <strong>Local Time:</strong> {localClock} ({localTimezone})<br>
        <strong>Berlin Time:</strong> {berlinClock} ({berlinTimezone})<br>
        {timeDifference}
    </p>

    <form on:submit|preventDefault={() => setPageLoadTimer()}>
        <label for="url">URL to load:</label><br>
        <input type="url" id="url" bind:value={url} required /><br><br>

        <label for="time">Time (HH:mm:ss.ms):</label><br>
        <input type="text" id="time" bind:value={timeInput} required /><br><br>

        <button type="submit">Set Timer</button>
    </form>

    <p><strong>{message}</strong></p>
</div>
