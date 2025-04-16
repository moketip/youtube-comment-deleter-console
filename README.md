# YouTube Comment Deleter - Easy Console Script (Variable Speed)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) **Effortlessly delete your old YouTube comments directly from your browser console!**

Tired of the tedious process of manually deleting YouTube comments from your Google Activity? This script provides a simple **copy-paste solution** to automate the task. No downloads, no complex setup needed – just paste the code below into your browser's console and let it run!

This script (v1.0) offers **stable performance** and includes **variable speed control** so you can manage the deletion rate effectively.

**--- ⚠️ WARNING: USE WITH EXTREME CAUTION ---**
**Action is IRREVERSIBLE.** Deleting comments via this script is **PERMANENT**. There is **NO UNDO** function. Double-check that you want to proceed before running the script.

## ✨ Features

* **Simple Copy-Paste:** No file downloads needed for basic use.
* **Automated Deletion:** Clicks the delete button for you.
* **Variable Speed:** Control the deletion pace using console commands (`go(speed)`, `setSpeed(value)`).
* **Smart Delays:** Uses randomized timing to improve reliability.
* **Auto-Scroll:** Loads older comments by scrolling down the page.
* **Pure Console:** Runs entirely within your browser's developer tools.

## 🚀 How to Use (Easy Console Method)

1.  **Navigate:** Open your YouTube comments history page in Google My Activity (usually under `https://myactivity.google.com/product/youtube`).
2.  **Open Console:** Launch your browser's Developer Tools (often by pressing `F12`) and select the **"Console"** tab.
3.  **Copy the Code:** Select and copy the **entire** JavaScript code block shown below:

    ```javascript
    // ==UserScript==
    // @name         YouTube Comment Deleter (Google My Activity - Adjustable Speed)
    // @namespace    [http://tampermonkey.net/](http://tampermonkey.net/)
    // @version      1.0
    // @description  Deletes YouTube comments from Google My Activity page (single click) with adjustable speed control. USE WITH EXTREME CAUTION.
    // @match        [https://myactivity.google.com/product/youtube](https://myactivity.google.com/product/youtube)*
    // @grant        none
    // ==/UserScript==
    // Note: This header is for organizational purposes; the script is intended for direct console execution.

    (function() {
        'use strict';

        // --- Configuration (Base Delays - Approx 1.5x faster than v0.5) ---
        const BASE_CLICK_DELAY_MS = 800;   // Base wait time after clicking 'X' (0.8 seconds)
        const BASE_SCROLL_DELAY_MS = 1450; // Base wait time after scrolling (1.45 seconds)
        const BASE_CHECK_INTERVAL_MS = 1200; // Base time between the start of checks (1.2 seconds)
        const BASE_RANDOM_DELAY_MAX_MS = 500; // Max random additional delay (0 to 0.5 seconds)

        // --- Speed Control ---
        let speedMultiplier = 1.0; // Default speed. > 1.0 is faster, < 1.0 is slower.

        // Function to get a delay with randomness, adjusted by speedMultiplier
        function getAdjustedDelay(baseMs, baseRandomMaxMs) {
            if (speedMultiplier <= 0) speedMultiplier = 0.1; // Prevent division by zero/negative
            const randomPart = Math.random() * baseRandomMaxMs;
            const calculatedDelay = (baseMs + randomPart) / speedMultiplier;
            return Math.max(50, calculatedDelay); // Ensure a minimum delay (e.g., 50ms) to prevent issues
        }


        // --- Selectors (VERY IMPORTANT - Check if Google updates the page!) ---
        const DELETE_BUTTON_SELECTOR = 'button[aria-label*="Delete activity item"]';


        // --- Script State ---
        let timeoutId = null; // Changed from intervalId as we use setTimeout chain
        let isRunning = false;
        let consecutiveScrollFailures = 0;
        const MAX_SCROLL_FAILURES = 4;

        // --- Helper Functions ---
        function findVisibleElement(selector) {
            const elements = Array.from(document.querySelectorAll(selector));
            return elements.find(el => el.offsetParent !== null && !el.disabled && el.clientHeight > 0 && el.clientWidth > 0);
        }

        // --- Core Deletion Logic ---
        async function processNextItem() {
            if (!isRunning) {
                console.log("Process stopped.");
                if(timeoutId) clearTimeout(timeoutId);
                timeoutId = null;
                return;
            }

            // Clear previous timer if it exists (shouldn't be necessary with await, but safe)
            if(timeoutId) clearTimeout(timeoutId);
            timeoutId = null;

            console.log(`(Speed: x${speedMultiplier.toFixed(1)}) Searching for delete button ('X')...`);
            const deleteButton = findVisibleElement(DELETE_BUTTON_SELECTOR);

            if (deleteButton) {
                const clickDelay = getAdjustedDelay(BASE_CLICK_DELAY_MS, BASE_RANDOM_DELAY_MAX_MS);
                console.log(`Delete button ('X') found. Clicking it. Waiting for ~${(clickDelay / 1000).toFixed(1)}s...`);
                consecutiveScrollFailures = 0;
                deleteButton.click();

                // Wait after clicking
                await new Promise(resolve => setTimeout(resolve, clickDelay));
                // Schedule the next check
                timeoutId = setTimeout(processNextItem, getAdjustedDelay(BASE_CHECK_INTERVAL_MS, 0)); // No random addition to check interval base

            } else {
                // No delete buttons visible, try scrolling
                const scrollDelay = getAdjustedDelay(BASE_SCROLL_DELAY_MS, BASE_RANDOM_DELAY_MAX_MS);
                console.log(`No visible delete buttons found. Scrolling down. Waiting for ~${(scrollDelay / 1000).toFixed(1)}s...`);
                let scrollHeightBefore = document.documentElement.scrollHeight;
                window.scrollTo(0, scrollHeightBefore + window.innerHeight); // Scroll down

                // Wait for new content to load
                await new Promise(resolve => setTimeout(resolve, scrollDelay));

                let scrollHeightAfter = document.documentElement.scrollHeight;
                const newDeleteButton = findVisibleElement(DELETE_BUTTON_SELECTOR); // Check again

                if (scrollHeightAfter > scrollHeightBefore || newDeleteButton) {
                     console.log("Scrolled down. New content may have loaded.");
                     consecutiveScrollFailures = 0;
                } else {
                     consecutiveScrollFailures++;
                     console.log(`Scrolled, but no new content or delete buttons detected. Failure ${consecutiveScrollFailures}/${MAX_SCROLL_FAILURES}.`);
                     if (consecutiveScrollFailures >= MAX_SCROLL_FAILURES) {
                          console.log("Reached the bottom or no more comments found after multiple scrolls. Stopping script.");
                          stop(); // Call the stop function
                          return; // Stop further execution in this path
                     }
                }
                // Schedule the next check after scrolling attempt
                timeoutId = setTimeout(processNextItem, getAdjustedDelay(BASE_CHECK_INTERVAL_MS, 0)); // No random addition to check interval base
            }
        }

        // --- Control Functions ---
        function go(speed = 1.0) { // Default speed is 1.0 (uses the faster base delays)
            if (isRunning) {
                console.log("Deletion process is already running. Use setSpeed(value) to adjust.");
                return;
            }
            speedMultiplier = Math.max(0.1, speed); // Ensure speed is positive
            console.log(`%cStarting YouTube Comment Deletion Process (Speed: x${speedMultiplier.toFixed(1)})...`, "color: orange; font-weight: bold;");
            console.log("Ensure you are on the 'My Activity -> YouTube comments' page.");
            console.warn("%c--- WARNING: THIS ACTION IS PERMANENT AND CANNOT BE UNDONE ---", "color: red; font-weight: bold;");
            console.warn("High speeds (> 2.5 or 3.0) might cause errors again. Adjust using setSpeed().");
            console.log("To stop the script: stop()");
            console.log("To change speed while running: setSpeed(value) (e.g., setSpeed(1.5), setSpeed(0.8))");


            isRunning = true;
            consecutiveScrollFailures = 0;
            // Start the process
            timeoutId = setTimeout(processNextItem, 500); // Start shortly after invoked
        }

        function stop() {
            if (isRunning) {
                isRunning = false; // Signal the running process to stop
                if (timeoutId) {
                     clearTimeout(timeoutId);
                     timeoutId = null;
                }
                console.log("%cDeletion process stopping... (will halt before next action)", "color: green; font-weight: bold;");
            } else {
                console.log("Deletion process was not running.");
            }
            // Re-expose functions
            window.go = go;
            window.stop = stop;
            window.setSpeed = setSpeed;
        }

        function setSpeed(newSpeed) {
            if (newSpeed <= 0) {
                console.warn("Speed multiplier must be positive. Setting to 0.1x instead.");
                speedMultiplier = 0.1;
            } else {
                 speedMultiplier = newSpeed;
                 console.log(`%cSpeed multiplier adjusted to: x${speedMultiplier.toFixed(1)}`, "color: blue;");
            }
            console.log("The new speed will take effect on the next action/delay calculation.");
        }


        // --- Expose Controls to Console ---
        window.go = go;
        window.stop = stop;
        window.setSpeed = setSpeed; // Expose the new speed control command

        console.log("YouTube Comment Deleter script (v1.0 - Adjustable Speed) loaded.");
        console.log("Ready! Use commands like go(), stop(), setSpeed(value).");

    })();
    ```

4.  **Paste & Run:** Paste the code into the console input line (it might look like `>`) and press `Enter`. The script will load and print its "loaded" message and usage hints.
5.  **Control Execution:** Now use the commands directly in the console:
    * `go()` - Starts deleting at the default speed.
    * `go(1.5)` - Starts deleting at 1.5x speed.
    * `stop()` - Stops the deletion process.
    * `setSpeed(0.8)` - Slows down the speed to 0.8x while it's running.

## ⚠️ Important Disclaimers

* **Page Structure Changes:** Google updates its websites. Future changes to the Google My Activity page *will likely break this script*. It relies on the current page design.
* **Rate Limiting/Blocks:** Very high speed settings might still cause temporary errors. Use the speed control wisely.
* **No Warranty:** Provided "as-is". Use responsibly and at your own risk.

## 📜 License

This project is licensed under the MIT License - see the (LICENSE) file for details
---