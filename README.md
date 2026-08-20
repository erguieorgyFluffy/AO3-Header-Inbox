# AO3 Header Inbox

Tampermonkey userscript that adds an **Inbox** link beside your AO3 username and displays the actual number of unread messages.

## Features

* Shows `Inbox (N)` with the actual unread count.
* Checks immediately when an AO3 page is loaded or refreshed.
* Automatically checks every **5 minutes**.
* Keeps the previous count visible while updating.
* Works across AO3 pages without visiting the Inbox.

## Installation

1. Install [Tampermonkey](https://www.tampermonkey.net/).
2. Open `ao3-header-inbox.user.js`.
3. Click **Raw**.
4. Install the script in Tampermonkey.

## How it works

The script checks your AO3 user dashboard for the `Inbox (N)` value and displays that number in the AO3 header.
