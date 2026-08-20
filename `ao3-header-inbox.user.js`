// ==UserScript==
// @name         AO3 Header Inbox
// @namespace    http://tampermonkey.net/
// @version      4.1
// @description  Adds an Inbox link beside the AO3 username and shows the actual unread Inbox count.
// @author       GPT-5.6 Luna
// @match        https://archiveofourown.org/*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function () {
    'use strict';

    // =========================================================
    // CONFIG
    // =========================================================

    const CHECK_INTERVAL =
        5 * 60 * 1000;

    const REQUEST_TIMEOUT =
        15 * 1000;

    const INITIAL_DELAY =
        1500;

    const HEADER_UPDATE_DELAY =
        100;

    // =========================================================
    // CONSTANTS
    // =========================================================

    const INBOX_ID =
        'ao3-header-inbox';

    const STORAGE_COUNT =
        'ao3-header-inbox-count';

    const STORAGE_USER =
        'ao3-header-inbox-user';

    // =========================================================
    // STATE
    // =========================================================

    let checkTimer = null;
    let headerUpdateTimer = null;

    let headerObserver = null;
    let documentObserver = null;
    let observedGreeting = null;

    let checking = false;

    /*
     * Every newly loaded tab/page starts with this true.
     *
     * It gets one immediate real AO3 check.
     */
    let freshTabCheckPending = true;

    /*
     * Last known successful count.
     *
     * This is loaded BEFORE the header is displayed, so the
     * user does not see:
     *
     * Inbox
     * Inbox (2)
     *
     * on every navigation.
     */
    let unreadCount = loadStoredCount();

    // =========================================================
    // STORAGE
    // =========================================================

    function loadStoredCount() {
        try {
            const raw =
                localStorage.getItem(
                    STORAGE_COUNT
                );

            if (raw === null) {
                return null;
            }

            const count =
                Number(raw);

            if (
                !Number.isInteger(count) ||
                count < 0
            ) {
                return null;
            }

            return count;

        } catch {
            return null;
        }
    }

    function loadStoredUser() {
        try {
            return localStorage.getItem(
                STORAGE_USER
            );
        } catch {
            return null;
        }
    }

    function saveCount(
        count,
        username
    ) {
        try {
            localStorage.setItem(
                STORAGE_COUNT,
                String(count)
            );

            localStorage.setItem(
                STORAGE_USER,
                username
            );
        } catch {
            // localStorage may be unavailable.
        }
    }

    // =========================================================
    // FIND AO3 HEADER
    // =========================================================

    function findGreeting() {
        return document.querySelector(
            '#greeting'
        );
    }

    function findUserNavigation() {
        const greeting =
            findGreeting();

        if (!greeting) {
            return null;
        }

        return greeting.querySelector(
            'ul.user.navigation.actions'
        );
    }

    // =========================================================
    // CURRENT USER
    // =========================================================

    function getCurrentUser() {
        const greeting =
            findGreeting();

        if (!greeting) {
            return null;
        }

        const profileLink =
            greeting.querySelector(
                'a.dropdown-toggle[href^="/users/"]'
            );

        if (!profileLink) {
            return null;
        }

        const href =
            profileLink.getAttribute(
                'href'
            );

        if (!href) {
            return null;
        }

        const match =
            href.match(
                /^\/users\/([^/?#]+)/
            );

        if (!match) {
            return null;
        }

        let username;

        try {
            username =
                decodeURIComponent(
                    match[1]
                );
        } catch {
            username =
                match[1];
        }

        return {
            username,

            encodedUsername:
                match[1],

            dashboardPath:
                `/users/${match[1]}`,

            inboxPath:
                `/users/${match[1]}/inbox`
        };
    }

    // =========================================================
    // INBOX LINK
    // =========================================================

    function findInboxElement() {
        return document.getElementById(
            INBOX_ID
        );
    }

    function removeInboxElement() {
        const inbox =
            findInboxElement();

        if (inbox) {
            inbox.remove();
        }
    }

    function ensureInboxLink() {
        const navigation =
            findUserNavigation();

        const user =
            getCurrentUser();

        if (
            !navigation ||
            !user
        ) {
            removeInboxElement();
            return;
        }

        let inbox =
            findInboxElement();

        /*
         * AO3 may rebuild #greeting.
         */
        if (
            inbox &&
            inbox.parentElement !== navigation
        ) {
            inbox.remove();
            inbox = null;
        }

        if (!inbox) {
            inbox =
                document.createElement(
                    'li'
                );

            inbox.id =
                INBOX_ID;

            const link =
                document.createElement(
                    'a'
                );

            link.href =
                user.inboxPath;

            inbox.appendChild(
                link
            );

            const userDropdown =
                navigation.querySelector(
                    'li.dropdown'
                );

            if (userDropdown) {
                userDropdown.insertAdjacentElement(
                    'afterend',
                    inbox
                );
            } else {
                navigation.prepend(
                    inbox
                );
            }
        }

        const link =
            inbox.querySelector('a');

        if (link) {
            link.href =
                user.inboxPath;
        }

        /*
         * IMPORTANT:
         *
         * Immediately render the persisted count.
         *
         * This prevents the Inbox -> Inbox (2) flicker.
         */
        updateInboxDisplay();
    }

    // =========================================================
    // DISPLAY
    // =========================================================

    function updateInboxDisplay(
        count = unreadCount
    ) {
        const inbox =
            findInboxElement();

        if (!inbox) {
            return;
        }

        const link =
            inbox.querySelector('a');

        if (!link) {
            return;
        }

        /*
         * If there has never been a successful check,
         * there is genuinely no number to display.
         */
        if (
            count === null ||
            !Number.isInteger(count) ||
            count < 0
        ) {
            link.textContent =
                'Inbox';

            link.setAttribute(
                'aria-label',
                'Inbox'
            );

            return;
        }

        if (count > 0) {
            link.textContent =
                `Inbox (${count})`;

            link.setAttribute(
                'aria-label',
                `Inbox, ${count} unread message${count === 1 ? '' : 's'}`
            );
        } else {
            link.textContent =
                'Inbox';

            link.setAttribute(
                'aria-label',
                'Inbox'
            );
        }
    }

    // =========================================================
    // FETCH AO3 USER DASHBOARD
    // =========================================================

    async function fetchUnreadInboxCount(
        user
    ) {
        const controller =
            new AbortController();

        const timeout =
            setTimeout(
                () => {
                    controller.abort();
                },
                REQUEST_TIMEOUT
            );

        try {
            /*
             * THIS is the important AO3 endpoint.
             *
             * /users/<username>
             *
             * NOT /dashboard
             * NOT the header
             * NOT the inbox page itself.
             */
            const response =
                await fetch(
                    user.dashboardPath,
                    {
                        method: 'GET',
                        credentials:
                            'same-origin',
                        cache:
                            'no-store',
                        signal:
                            controller.signal
                    }
                );

            if (!response.ok) {
                throw new Error(
                    `AO3 returned HTTP ${response.status}`
                );
            }

            const html =
                await response.text();

            const parsed =
                new DOMParser()
                    .parseFromString(
                        html,
                        'text/html'
                    );

            /*
             * AO3's user dashboard contains:
             *
             * <div id="dashboard">
             *
             * and inside it:
             *
             * Inbox (2)
             */
            const dashboard =
                parsed.querySelector(
                    '#dashboard'
                );

            if (!dashboard) {
                throw new Error(
                    'AO3 #dashboard not found'
                );
            }

            /*
             * Check both links and the <span class="current">
             * AO3 uses for the currently selected dashboard item.
             */
            const candidates =
                dashboard.querySelectorAll(
                    'a, span.current'
                );

            for (
                const element
                of candidates
            ) {
                const text =
                    element.textContent
                        .replace(/\s+/g, ' ')
                        .trim();

                const match =
                    text.match(
                        /^Inbox\s*\(\s*(\d+)\s*\)$/i
                    );

                if (!match) {
                    continue;
                }

                const count =
                    Number(
                        match[1]
                    );

                if (
                    Number.isInteger(count) &&
                    count >= 0
                ) {
                    return count;
                }
            }

            throw new Error(
                'Inbox (N) was not found in #dashboard'
            );

        } finally {
            clearTimeout(timeout);
        }
    }

    // =========================================================
    // CHECK
    // =========================================================

    async function checkInbox() {
        if (checking) {
            return;
        }

        if (document.hidden) {
            scheduleNextCheck();
            return;
        }

        const user =
            getCurrentUser();

        if (!user) {
            removeInboxElement();
            scheduleNextCheck();
            return;
        }

        /*
         * Make sure the link exists and immediately display
         * the previous successful count.
         */
        ensureInboxLink();

        /*
         * If this is a different logged-in account, don't
         * display the previous account's number.
         */
        const storedUser =
            loadStoredUser();

        if (
            storedUser &&
            storedUser !== user.username
        ) {
            unreadCount = null;
            updateInboxDisplay();
        }

        checking = true;

        try {
            const count =
                await fetchUnreadInboxCount(
                    user
                );

            /*
             * REAL CURRENT AO3 COUNT.
             */
            unreadCount =
                count;

            /*
             * Persist it so the next page/tab can show it
             * immediately.
             */
            saveCount(
                count,
                user.username
            );

            updateInboxDisplay(
                count
            );

            console.debug(
                '[AO3 Header Inbox] actual unread count:',
                count
            );

        } catch (error) {
            /*
             * NEVER erase the previous successful count
             * because of a temporary request failure.
             */
            console.warn(
                '[AO3 Header Inbox] check failed:',
                error
            );

            updateInboxDisplay();

        } finally {
            checking = false;

            freshTabCheckPending =
                false;

            scheduleNextCheck();
        }
    }

    // =========================================================
    // SCHEDULING
    // =========================================================

    function scheduleNextCheck() {
        clearTimeout(
            checkTimer
        );

        checkTimer = null;

        if (document.hidden) {
            return;
        }

        /*
         * Fresh page/tab:
         * immediate check.
         */
        if (
            freshTabCheckPending
        ) {
            checkTimer =
                setTimeout(
                    () => {
                        checkInbox();
                    },
                    250
                );

            return;
        }

        /*
         * Existing page:
         * normal 5-minute check.
         */
        checkTimer =
            setTimeout(
                () => {
                    checkInbox();
                },
                CHECK_INTERVAL
            );
    }

    // =========================================================
    // HEADER OBSERVER
    // =========================================================

    function scheduleHeaderUpdate() {
        clearTimeout(
            headerUpdateTimer
        );

        headerUpdateTimer =
            setTimeout(
                () => {
                    ensureInboxLink();
                },
                HEADER_UPDATE_DELAY
            );
    }

    function observeGreeting(
        greeting
    ) {
        if (!greeting) {
            return;
        }

        if (
            headerObserver &&
            observedGreeting === greeting
        ) {
            return;
        }

        if (headerObserver) {
            headerObserver.disconnect();
        }

        observedGreeting =
            greeting;

        headerObserver =
            new MutationObserver(
                () => {
                    scheduleHeaderUpdate();
                }
            );

        headerObserver.observe(
            greeting,
            {
                childList: true,
                subtree: true
            }
        );
    }

    // =========================================================
    // HEADER REPLACEMENT
    // =========================================================

    function observeHeaderReplacement() {
        if (
            documentObserver ||
            !document.body
        ) {
            return;
        }

        documentObserver =
            new MutationObserver(
                (mutations) => {
                    let changed = false;

                    for (
                        const mutation
                        of mutations
                    ) {
                        for (
                            const node
                            of mutation.addedNodes
                        ) {
                            if (
                                node instanceof Element &&
                                (
                                    node.id === 'greeting' ||
                                    node.querySelector?.(
                                        '#greeting'
                                    )
                                )
                            ) {
                                changed = true;
                                break;
                            }
                        }

                        if (changed) {
                            break;
                        }

                        for (
                            const node
                            of mutation.removedNodes
                        ) {
                            if (
                                node instanceof Element &&
                                (
                                    node.id === 'greeting' ||
                                    node.querySelector?.(
                                        '#greeting'
                                    )
                                )
                            ) {
                                changed = true;
                                break;
                            }
                        }

                        if (changed) {
                            break;
                        }
                    }

                    if (!changed) {
                        return;
                    }

                    const greeting =
                        findGreeting();

                    if (greeting) {
                        observeGreeting(
                            greeting
                        );

                        /*
                         * This now immediately restores the
                         * persisted count instead of displaying
                         * a blank Inbox.
                         */
                        ensureInboxLink();

                    } else {
                        removeInboxElement();
                    }
                }
            );

        documentObserver.observe(
            document.body,
            {
                childList: true
            }
        );
    }

    // =========================================================
    // VISIBILITY
    // =========================================================

    document.addEventListener(
        'visibilitychange',
        () => {
            if (document.hidden) {
                clearTimeout(
                    checkTimer
                );

                checkTimer = null;

                return;
            }

            /*
             * Returning to an existing tab does not make it
             * a fresh tab.
             */
            scheduleNextCheck();
        }
    );

    // =========================================================
    // INITIALIZATION
    // =========================================================

    function init() {
        const greeting =
            findGreeting();

        if (greeting) {
            observeGreeting(
                greeting
            );
        }

        observeHeaderReplacement();

        /*
         * This immediately creates the link AND uses the
         * persisted count, if one exists.
         */
        ensureInboxLink();

        setTimeout(
            () => {
                const currentGreeting =
                    findGreeting();

                if (currentGreeting) {
                    observeGreeting(
                        currentGreeting
                    );
                }

                ensureInboxLink();

                /*
                 * New tab/page gets its actual AO3 check.
                 */
                checkInbox();

            },
            INITIAL_DELAY
        );
    }

    if (
        document.readyState ===
        'loading'
    ) {
        document.addEventListener(
            'DOMContentLoaded',
            init,
            {
                once: true
            }
        );
    } else {
        init();
    }

})();
