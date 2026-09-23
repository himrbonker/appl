import { engine } from "./engine.js";
import { resolveInput, formatForDisplay } from "./url.js";

const $ = (selector) => document.querySelector(selector);
const addressBar = $("#address");
const frames = $("#frames");
const status = $("#status");
const progressBar = $("#progress");
const progressRing = $("#progress-ring");
const reloadBtn = $("#reload");
const tabstrip = $("#tabstrip");
const newTabBtn = $("#new-tab");
const newtabEl = $("#newtab");
const shortcutsEl = $("#shortcuts");
const shortcutsPanel = $("#shortcuts-panel");
const quicklaunchEl = $("#quicklaunch");
const sidebarQuicklaunchEl = $("#sidebar-quicklaunch");
const splitBtn = $("#split-btn");
const heroSearchForm = $("#hero-search");
const heroSearchInput = $("#hero-search-input");
const starBtn = $("#star-btn");
const shareBtn = $("#share-btn");
const fullscreenBtn = $("#fullscreen-btn");
const sidebarHomeBtn = $("#sidebar-home");
const sidebarAddShortcutBtn = $("#sidebar-add-shortcut");
const bgBtn = $("#sidebar-customize");
const bgPanel = $("#bg-panel");
const bgPanelClose = $("#bg-panel-close");
const bgSwatchesEl = $("#bg-swatches");
const bgColorInput = $("#bg-color-input");
const bgUploadInput = $("#bg-upload-input");
const bgResetBtn = $("#bg-reset-btn");
const shortcutDialog = $("#shortcut-dialog");
const shortcutForm = $("#shortcut-form");
const shortcutDialogTitle = $("#shortcut-dialog-title");
const shortcutNameInput = $("#shortcut-name");
const shortcutUrlInput = $("#shortcut-url");
const shortcutCancelBtn = $("#shortcut-cancel");
const shortcutIconPreview = $("#shortcut-icon-preview");
const shortcutIconInput = $("#shortcut-icon-input");
const shortcutIconClearBtn = $("#shortcut-icon-clear");

let addressBarFocused = false;
addressBar.addEventListener("focus", () => (addressBarFocused = true));
addressBar.addEventListener("blur", () => (addressBarFocused = false));

const searchTemplate = () => "https://duckduckgo.com/?q=%s";
const startUrl = () => "";

let nextTabId = 1;
const tabs = [];
let activeId = null;

const activeTab = () => tabs.find(t => t.id === activeId) ?? null;
const tabById = (id) => tabs.find(t => t.id === id) ?? null;

const createTabState = () => {
    const id = nextTabId++;
    const frame = document.createElement("iframe");
    frame.className = "frame";
    frames.append(frame);
    return {
        id,
        frame,
        session: null,
        sessionPending: null,
        url: "",
        loading: false,
        visited: [],
        visitedIndex: -1
    };
};

const record = (tab, url) => {
    tab.url = url;
    if (tab.visited[tab.visitedIndex] === url)
        return;
    const existing = tab.visited.lastIndexOf(url, tab.visitedIndex - 1);
    if (existing >= 0) {
        tab.visitedIndex = existing;
        return;
    }
    tab.visited.splice(tab.visitedIndex + 1);
    tab.visited.push(url);
    tab.visitedIndex = tab.visited.length - 1;
};

const canGoBack = (tab) => tab.visitedIndex > 0;
const canGoForward = (tab) => tab.visitedIndex >= 0 && tab.visitedIndex < tab.visited.length - 1;
const goBack = (tab) => (canGoBack(tab) ? tab.visited[--tab.visitedIndex] ?? null : null);
const goForward = (tab) => (canGoForward(tab) ? tab.visited[++tab.visitedIndex] ?? null : null);

const ensureSession = async (tab) => {
    if (tab.session)
        return tab.session;
    if (tab.sessionPending)
        return tab.sessionPending;
    tab.sessionPending = engine.createSession(tab.frame, {
        url: url => {
            record(tab, url);
            render();
        },
        loading: () => {
            tab.loading = true;
            render();
        },
        ready: () => {
            tab.loading = false;
            render();
        },
        error: error => {
            tab.loading = false;
            if (tab.id === activeId)
                setStatus(error?.message ?? String(error));
        },
        escape: url => void navigate(tab, url)
    });
    try {
        tab.session = await tab.sessionPending;
    }
    finally {
        tab.sessionPending = null;
    }
    return tab.session;
};

const navigate = async (tab, input, options = {}) => {
    const { url, kind } = resolveInput(input, searchTemplate());
    switch (kind) {
        case "empty":
            return;
        case "blocked":
            if (tab.id === activeId)
                setStatus("That address cannot be opened through the proxy.");
            return;
        case "external":
            location.assign(url);
            return;
        default: {
            if (tab.id === activeId)
                setStatus("");
            tab.frame.removeAttribute("srcdoc");
            await ensureSession(tab);
            if (options.record === false)
                tab.url = url;
            else
                record(tab, url);
            tab.session.go(url);
            render();
        }
    }
};

const navigateActive = (input, options) => {
    const tab = activeTab();
    if (tab)
        void navigate(tab, input, options);
};

/* ---------------- Quick launch row ---------------- */

const quickLaunch = [
    {
        id: "games",
        label: "Games",
        url: "https://www.crazygames.com",
        icon:
            '<svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">' +
            '<rect x="2.5" y="8" width="19" height="9" rx="4.5" fill="none" stroke="currentColor" stroke-width="1.6"/>' +
            '<path d="M7 10.5v4M5 12.5h4" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>' +
            '<circle cx="16" cy="11" r="1" fill="currentColor"/><circle cx="18.2" cy="13.2" r="1" fill="currentColor"/>' +
            "</svg>"
    },
    {
        id: "ai",
        label: "AI",
        url: "https://chatgpt.com",
        icon:
            '<svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">' +
            '<rect x="5" y="7" width="14" height="11" rx="3" fill="none" stroke="currentColor" stroke-width="1.6"/>' +
            '<path d="M12 7V4" stroke="currentColor" stroke-width="1.6" stroke-linecap="round"/>' +
            '<circle cx="12" cy="3.3" r="1" fill="currentColor"/>' +
            '<circle cx="9" cy="12.5" r="1.15" fill="currentColor"/><circle cx="15" cy="12.5" r="1.15" fill="currentColor"/>' +
            "</svg>"
    },
    {
        id: "movies",
        label: "Movies",
        url: "https://moovie.fun",
        icon:
            '<svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">' +
            '<path d="m3 8 2.4-3.6h3l-1.8 3.6M9 8l1.8-3.6h3L12 8m3 0 1.8-3.6h3L18 8" stroke="currentColor" stroke-width="1.4" stroke-linejoin="round" fill="none"/>' +
            '<rect x="3" y="8" width="18" height="11" rx="1.5" fill="none" stroke="currentColor" stroke-width="1.6"/>' +
            "</svg>"
    },
    {
        id: "music",
        label: "Music",
        url: "https://open.spotify.com",
        icon:
            '<svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">' +
            '<path d="M9 17V5.8L20 4v11.2" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" fill="none"/>' +
            '<circle cx="6.5" cy="17" r="2.5" fill="none" stroke="currentColor" stroke-width="1.6"/>' +
            '<circle cx="17.5" cy="15.2" r="2.5" fill="none" stroke="currentColor" stroke-width="1.6"/>' +
            "</svg>"
    },
    {
        id: "chat",
        label: "Discord",
        url: "https://discord.com/app",
        icon:
            '<svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">' +
            '<path d="M4 18V8a3 3 0 0 1 3-3h10a3 3 0 0 1 3 3v10l-3.2-2.4H7.2Z" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>' +
            '<circle cx="9.5" cy="10.5" r="1.15" fill="currentColor"/><circle cx="14.5" cy="10.5" r="1.15" fill="currentColor"/>' +
            "</svg>"
    }
];

const renderQuickLaunch = () => {
    sidebarQuicklaunchEl.innerHTML = "";
    quicklaunchEl.innerHTML = "";

    for (const item of quickLaunch) {
        const sidebarBtn = document.createElement("button");
        sidebarBtn.type = "button";
        sidebarBtn.className = "sidebar__btn";
        sidebarBtn.setAttribute("aria-label", item.label);
        sidebarBtn.title = item.label;
        sidebarBtn.innerHTML = item.icon;
        sidebarBtn.addEventListener("click", () => navigateActive(item.url));
        sidebarQuicklaunchEl.append(sidebarBtn);

        const heroBtn = document.createElement("button");
        heroBtn.type = "button";
        heroBtn.className = "quicklaunch__item";
        const heroIcon = document.createElement("span");
        heroIcon.className = "quicklaunch__icon";
        heroIcon.innerHTML = item.icon;
        const heroLabel = document.createElement("span");
        heroLabel.className = "quicklaunch__label";
        heroLabel.textContent = item.label;
        heroBtn.append(heroIcon, heroLabel);
        heroBtn.addEventListener("click", () => navigateActive(item.url));
        quicklaunchEl.append(heroBtn);
    }

    const addBtn = document.createElement("button");
    addBtn.type = "button";
    addBtn.className = "quicklaunch__item";
    addBtn.innerHTML =
        '<span class="quicklaunch__icon"><svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">' +
        '<path d="M12 5v14M5 12h14" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round"/>' +
        '</svg></span><span class="quicklaunch__label">Add</span>';
    addBtn.addEventListener("click", () => openShortcutDialog(null));
    quicklaunchEl.append(addBtn);

    const allAppsBtn = document.createElement("button");
    allAppsBtn.type = "button";
    allAppsBtn.className = "quicklaunch__item";
    allAppsBtn.innerHTML =
        '<span class="quicklaunch__icon"><svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">' +
        '<circle cx="7" cy="7" r="1.6" fill="currentColor"/><circle cx="12" cy="7" r="1.6" fill="currentColor"/><circle cx="17" cy="7" r="1.6" fill="currentColor"/>' +
        '<circle cx="7" cy="12" r="1.6" fill="currentColor"/><circle cx="12" cy="12" r="1.6" fill="currentColor"/><circle cx="17" cy="12" r="1.6" fill="currentColor"/>' +
        '<circle cx="7" cy="17" r="1.6" fill="currentColor"/><circle cx="12" cy="17" r="1.6" fill="currentColor"/><circle cx="17" cy="17" r="1.6" fill="currentColor"/>' +
        '</svg></span><span class="quicklaunch__label">All Apps</span>';
    allAppsBtn.addEventListener("click", () => {
        shortcutsPanel.hidden = !shortcutsPanel.hidden;
    });
    quicklaunchEl.append(allAppsBtn);
};

splitBtn.addEventListener("click", () => {
    const active = frames.classList.toggle("split-view");
    splitBtn.setAttribute("aria-pressed", String(active));
});



const SHORTCUTS_KEY = "proxy:shortcuts";

const faviconUrl = (url) => {
    try {
        const host = new URL(url).hostname;
        return `https://www.google.com/s2/favicons?sz=64&domain=${encodeURIComponent(host)}`;
    }
    catch {
        return null;
    }
};

const defaultShortcuts = () => [
    { id: "s1", name: "DuckDuckGo", url: "https://duckduckgo.com" },
    { id: "s2", name: "Wikipedia", url: "https://wikipedia.org" },
    { id: "s3", name: "YouTube", url: "https://youtube.com" }
];

const loadShortcuts = () => {
    try {
        const raw = localStorage.getItem(SHORTCUTS_KEY);
        if (!raw)
            return defaultShortcuts();
        const parsed = JSON.parse(raw);
        return Array.isArray(parsed) ? parsed : defaultShortcuts();
    }
    catch {
        return defaultShortcuts();
    }
};

let shortcuts = loadShortcuts();

const saveShortcuts = () => {
    try {
        localStorage.setItem(SHORTCUTS_KEY, JSON.stringify(shortcuts));
    }
    catch { /* storage unavailable or full; keep going in-memory */ }
};

const colorForString = (str) => {
    let hash = 0;
    for (let i = 0; i < str.length; i++)
        hash = (hash * 31 + str.charCodeAt(i)) >>> 0;
    return `hsl(${hash % 360} 55% 42%)`;
};

const shortcutInitial = (shortcut) => {
    const source = (shortcut.name || formatForDisplay(shortcut.url) || "?").trim();
    return source.charAt(0).toUpperCase() || "?";
};

const isShortcutUrl = (url) => shortcuts.some(s => s.url === url);

let editingShortcutId = null;
let shortcutIconValue = null;

const updateIconPreview = () => {
    if (shortcutIconValue) {
        shortcutIconPreview.style.backgroundImage = `url("${shortcutIconValue}")`;
        shortcutIconPreview.textContent = "";
        shortcutIconClearBtn.hidden = false;
    }
    else {
        shortcutIconPreview.style.backgroundImage = "none";
        shortcutIconPreview.style.background = colorForString(
            shortcutNameInput.value || shortcutUrlInput.value || "?"
        );
        shortcutIconPreview.textContent = (
            (shortcutNameInput.value || shortcutUrlInput.value || "?").trim().charAt(0).toUpperCase() || "?"
        );
        shortcutIconClearBtn.hidden = true;
    }
};

const openShortcutDialog = (shortcut) => {
    editingShortcutId = shortcut ? shortcut.id : null;
    shortcutDialogTitle.textContent = shortcut ? "Edit shortcut" : "Add shortcut";
    shortcutNameInput.value = shortcut ? shortcut.name : "";
    shortcutUrlInput.value = shortcut ? shortcut.url : "";
    shortcutIconValue = shortcut?.icon ?? null;
    shortcutIconInput.value = "";
    updateIconPreview();
    shortcutDialog.showModal();
    shortcutNameInput.focus();
};

const removeShortcut = (id) => {
    shortcuts = shortcuts.filter(s => s.id !== id);
    saveShortcuts();
    renderShortcuts();
    render();
};

const addShortcutFromUrl = (url) => {
    if (isShortcutUrl(url))
        return;
    shortcuts.push({
        id: crypto.randomUUID?.() ?? String(Date.now()),
        name: formatForDisplay(url),
        url
    });
    saveShortcuts();
    renderShortcuts();
    render();
};

const renderShortcuts = () => {
    shortcutsEl.innerHTML = "";
    for (const shortcut of shortcuts) {
        const tile = document.createElement("div");
        tile.className = "shortcut";

        const link = document.createElement("button");
        link.type = "button";
        link.className = "shortcut__link";
        link.addEventListener("click", () => navigateActive(shortcut.url));

        const icon = document.createElement("span");
        icon.className = "shortcut__icon";
        icon.style.background = colorForString(shortcut.name || shortcut.url);
        icon.textContent = shortcutInitial(shortcut);

        const iconSrc = shortcut.icon || faviconUrl(shortcut.url);
        if (iconSrc) {
            const img = document.createElement("img");
            img.className = "shortcut__icon-img";
            img.src = iconSrc;
            img.alt = "";
            img.loading = "lazy";
            img.addEventListener("error", () => img.remove());
            icon.append(img);
        }

        const label = document.createElement("span");
        label.className = "shortcut__label";
        label.textContent = shortcut.name || formatForDisplay(shortcut.url);

        link.append(icon, label);

        const editBtn = document.createElement("button");
        editBtn.type = "button";
        editBtn.className = "shortcut__edit";
        editBtn.setAttribute("aria-label", `Edit ${shortcut.name || shortcut.url}`);
        editBtn.innerHTML =
            '<svg viewBox="0 0 24 24" width="11" height="11" aria-hidden="true">' +
            '<path d="m4 20 1-4 11-11 3 3-11 11-4 1Z" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linejoin="round"/>' +
            "</svg>";
        editBtn.addEventListener("click", event => {
            event.stopPropagation();
            openShortcutDialog(shortcut);
        });

        const removeBtn = document.createElement("button");
        removeBtn.type = "button";
        removeBtn.className = "shortcut__remove";
        removeBtn.setAttribute("aria-label", `Remove ${shortcut.name || shortcut.url}`);
        removeBtn.innerHTML =
            '<svg viewBox="0 0 24 24" width="10" height="10" aria-hidden="true">' +
            '<path d="M6 6l12 12M18 6 6 18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>' +
            "</svg>";
        removeBtn.addEventListener("click", event => {
            event.stopPropagation();
            removeShortcut(shortcut.id);
        });

        tile.append(link, editBtn, removeBtn);
        shortcutsEl.append(tile);
    }

    const addTile = document.createElement("button");
    addTile.type = "button";
    addTile.className = "shortcut--add";
    addTile.innerHTML =
        '<span class="shortcut__icon--add">+</span><span class="shortcut__label">Add shortcut</span>';
    addTile.addEventListener("click", () => openShortcutDialog(null));
    shortcutsEl.append(addTile);
};

shortcutIconInput.addEventListener("change", () => {
    const file = shortcutIconInput.files?.[0];
    if (!file)
        return;
    const reader = new FileReader();
    reader.onload = () => {
        shortcutIconValue = String(reader.result);
        updateIconPreview();
    };
    reader.readAsDataURL(file);
});

shortcutIconClearBtn.addEventListener("click", () => {
    shortcutIconValue = null;
    shortcutIconInput.value = "";
    updateIconPreview();
});

for (const input of [shortcutNameInput, shortcutUrlInput]) {
    input.addEventListener("input", () => {
        if (!shortcutIconValue)
            updateIconPreview();
    });
}

shortcutForm.addEventListener("submit", event => {
    event.preventDefault();
    const name = shortcutNameInput.value.trim();
    let url = shortcutUrlInput.value.trim();
    if (!url)
        return;
    if (!/^[a-z][a-z0-9+.-]*:/i.test(url))
        url = "https://" + url;
    try {
        url = new URL(url).href;
    }
    catch {
        shortcutUrlInput.focus();
        return;
    }

    if (editingShortcutId) {
        const existing = shortcuts.find(s => s.id === editingShortcutId);
        if (existing) {
            existing.name = name || formatForDisplay(url);
            existing.url = url;
            if (shortcutIconValue)
                existing.icon = shortcutIconValue;
            else
                delete existing.icon;
        }
    }
    else {
        const entry = {
            id: crypto.randomUUID?.() ?? String(Date.now()),
            name: name || formatForDisplay(url),
            url
        };
        if (shortcutIconValue)
            entry.icon = shortcutIconValue;
        shortcuts.push(entry);
    }
    saveShortcuts();
    renderShortcuts();
    render();
    shortcutDialog.close();
});

shortcutCancelBtn.addEventListener("click", () => shortcutDialog.close());
shortcutDialog.addEventListener("click", event => {
    if (event.target === shortcutDialog)
        shortcutDialog.close();
});

/* ---------------- New tab: background ---------------- */

const BACKGROUND_KEY = "proxy:background";

const bgPresets = [
    { id: "default", css: "linear-gradient(160deg, #1e1f22, #0f1011)" },
    { id: "indigo", css: "linear-gradient(160deg, #2b2a4a, #12121d)" },
    { id: "forest", css: "linear-gradient(160deg, #1c3329, #0d1613)" },
    { id: "sunset", css: "linear-gradient(160deg, #4a2b2b, #1a1010)" },
    { id: "ocean", css: "linear-gradient(160deg, #1c2b3a, #0c1620)" },
    { id: "slate", css: "linear-gradient(160deg, #35363c, #141518)" }
];

const loadBackground = () => {
    try {
        const raw = localStorage.getItem(BACKGROUND_KEY);
        if (!raw)
            return { type: "preset", value: "default" };
        const parsed = JSON.parse(raw);
        return parsed && parsed.type && parsed.value
            ? parsed
            : { type: "preset", value: "default" };
    }
    catch {
        return { type: "preset", value: "default" };
    }
};

let background = loadBackground();

const saveBackground = () => {
    try {
        localStorage.setItem(BACKGROUND_KEY, JSON.stringify(background));
    }
    catch { /* storage unavailable or full (e.g. large image); keep the in-memory value */ }
};

const applyBackground = () => {
    if (background.type === "image") {
        newtabEl.style.backgroundImage = `url("${background.value}")`;
        newtabEl.style.backgroundColor = "";
    }
    else if (background.type === "color") {
        newtabEl.style.backgroundImage = "none";
        newtabEl.style.backgroundColor = background.value;
    }
    else {
        const preset = bgPresets.find(p => p.id === background.value) ?? bgPresets[0];
        newtabEl.style.backgroundImage = preset.css;
        newtabEl.style.backgroundColor = "";
    }
};

const renderSwatches = () => {
    bgSwatchesEl.innerHTML = "";
    for (const preset of bgPresets) {
        const active = background.type === "preset" && background.value === preset.id;
        const swatch = document.createElement("button");
        swatch.type = "button";
        swatch.className = "bg-swatch" + (active ? " bg-swatch--active" : "");
        swatch.style.backgroundImage = preset.css;
        swatch.setAttribute("aria-label", preset.id);
        swatch.addEventListener("click", () => {
            background = { type: "preset", value: preset.id };
            saveBackground();
            applyBackground();
            renderSwatches();
        });
        bgSwatchesEl.append(swatch);
    }
};

bgBtn.addEventListener("click", () => {
    bgPanel.hidden = !bgPanel.hidden;
    bgBtn.setAttribute("aria-expanded", String(!bgPanel.hidden));
    if (!bgPanel.hidden)
        renderSwatches();
});

bgPanelClose.addEventListener("click", () => {
    bgPanel.hidden = true;
    bgBtn.setAttribute("aria-expanded", "false");
});

document.addEventListener("click", event => {
    if (bgPanel.hidden)
        return;
    if (bgPanel.contains(event.target) || bgBtn.contains(event.target))
        return;
    bgPanel.hidden = true;
    bgBtn.setAttribute("aria-expanded", "false");
});

bgColorInput.addEventListener("input", () => {
    background = { type: "color", value: bgColorInput.value };
    saveBackground();
    applyBackground();
    renderSwatches();
});

bgUploadInput.addEventListener("change", () => {
    const file = bgUploadInput.files?.[0];
    if (!file)
        return;
    const reader = new FileReader();
    reader.onload = () => {
        background = { type: "image", value: String(reader.result) };
        saveBackground();
        applyBackground();
        renderSwatches();
    };
    reader.readAsDataURL(file);
});

bgResetBtn.addEventListener("click", () => {
    background = { type: "preset", value: "default" };
    saveBackground();
    applyBackground();
    renderSwatches();
    bgUploadInput.value = "";
});

const tabLabel = (tab) => {
    if (tab.loading && !tab.url)
        return "New tab";
    if (!tab.url)
        return "New tab";
    return formatForDisplay(tab.url) || "New tab";
};

const renderTabs = () => {
    tabstrip.innerHTML = "";
    for (const tab of tabs) {
        const el = document.createElement("div");
        el.className = "tab" + (tab.id === activeId ? " tab--active" : "");
        el.dataset.tabId = String(tab.id);
        el.title = tab.url || "New tab";

        const icon = document.createElement("span");
        icon.className = "tab__icon";
        if (tab.loading) {
            icon.classList.add("tab__icon--loading");
        }
        else {
            icon.innerHTML =
                '<svg viewBox="0 0 24 24" width="12" height="12" aria-hidden="true">' +
                '<circle cx="12" cy="12" r="8" fill="none" stroke="currentColor" stroke-width="1.6"/>' +
                '<path d="M4 12h16M12 4a15 15 0 0 1 0 16M12 4a15 15 0 0 0 0 16" fill="none" stroke="currentColor" stroke-width="1.3"/>' +
                "</svg>";
        }

        const label = document.createElement("span");
        label.className = "tab__label";
        label.textContent = tabLabel(tab);

        const close = document.createElement("button");
        close.className = "tab__close";
        close.type = "button";
        close.setAttribute("aria-label", "Close tab");
        close.innerHTML =
            '<svg viewBox="0 0 24 24" width="11" height="11" aria-hidden="true">' +
            '<path d="M6 6l12 12M18 6 6 18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>' +
            "</svg>";
        close.addEventListener("click", event => {
            event.stopPropagation();
            closeTab(tab.id);
        });

        el.append(icon, label, close);
        el.addEventListener("click", () => switchTab(tab.id));
        tabstrip.append(el);
    }
};

const switchTab = (id) => {
    if (id === activeId)
        return;
    const tab = tabById(id);
    if (!tab)
        return;
    activeId = id;
    for (const t of tabs)
        t.frame.classList.toggle("frame--active", t.id === id);
    renderTabs();
    render();
};

const openTab = (input, { activate = true } = {}) => {
    const tab = createTabState();
    tabs.push(tab);
    if (activate) {
        for (const t of tabs)
            t.frame.classList.toggle("frame--active", t.id === tab.id);
        activeId = tab.id;
    }
    renderTabs();
    render();
    if (input !== undefined)
        void navigate(tab, input);
    return tab;
};

const destroyTab = (tab) => {
    tab.session?.destroy();
    tab.frame.remove();
};

const closeTab = (id) => {
    const index = tabs.findIndex(t => t.id === id);
    if (index === -1)
        return;
    const [tab] = tabs.splice(index, 1);
    destroyTab(tab);

    if (tabs.length === 0) {
        openTab(startUrl());
        return;
    }

    if (id === activeId) {
        const next = tabs[index] ?? tabs[index - 1];
        activeId = next.id;
        for (const t of tabs)
            t.frame.classList.toggle("frame--active", t.id === activeId);
    }
    renderTabs();
    render();
};

const render = () => {
    const tab = activeTab();
    if (!tab)
        return;
    if (!addressBarFocused)
        addressBar.value = tab.url ? formatForDisplay(tab.url) : "";
    $("#back").disabled = !canGoBack(tab);
    $("#forward").disabled = !canGoForward(tab);
    reloadBtn.disabled = !tab.session;
    const loading = tab.loading;
    progressBar.hidden = !loading;
    progressRing.hidden = !loading;
    reloadBtn.classList.toggle("nav-btn--spin", loading);
    if (loading)
        setStatus("Loading");
    else if (status.textContent === "Loading")
        setStatus("");
    const blank = !tab.url && !tab.loading;
    newtabEl.classList.toggle("newtab--visible", blank);
    sidebarHomeBtn.classList.toggle("sidebar__btn--active", blank);
    starBtn.disabled = !tab.url;
    starBtn.classList.toggle("star-btn--active", !!tab.url && isShortcutUrl(tab.url));
    shareBtn.disabled = !tab.url;
    renderTabs();
    document.title = tab.url ? `${formatForDisplay(tab.url)} — Appl` : "Appl";
};

const setStatus = (message) => {
    status.textContent = message ?? "";
    status.hidden = !message;
};

$("#omnibox").addEventListener("submit", event => {
    event.preventDefault();
    addressBar.blur();
    navigateActive(addressBar.value);
});

heroSearchForm.addEventListener("submit", event => {
    event.preventDefault();
    const value = heroSearchInput.value;
    heroSearchInput.value = "";
    navigateActive(value);
});

$("#back").addEventListener("click", () => {
    const tab = activeTab();
    if (!tab)
        return;
    const url = goBack(tab);
    if (url)
        void navigate(tab, url, { record: false });
});

$("#forward").addEventListener("click", () => {
    const tab = activeTab();
    if (!tab)
        return;
    const url = goForward(tab);
    if (url)
        void navigate(tab, url, { record: false });
});

reloadBtn.addEventListener("click", () => activeTab()?.session?.reload());

newTabBtn.addEventListener("click", () => openTab(startUrl()));

const goHome = () => {
    const tab = activeTab();
    if (!tab)
        return;
    if (!tab.url && !tab.loading && !tab.session)
        return;
    const index = tabs.findIndex(t => t.id === tab.id);
    destroyTab(tab);
    const fresh = createTabState();
    tabs[index] = fresh;
    activeId = fresh.id;
    for (const t of tabs)
        t.frame.classList.toggle("frame--active", t.id === activeId);
    renderTabs();
    render();
};

sidebarHomeBtn.addEventListener("click", goHome);
sidebarAddShortcutBtn.addEventListener("click", () => openShortcutDialog(null));

starBtn.addEventListener("click", () => {
    const tab = activeTab();
    if (!tab?.url)
        return;
    if (isShortcutUrl(tab.url)) {
        const match = shortcuts.find(s => s.url === tab.url);
        if (match)
            removeShortcut(match.id);
    }
    else {
        addShortcutFromUrl(tab.url);
    }
});

shareBtn.addEventListener("click", async () => {
    const tab = activeTab();
    if (!tab?.url)
        return;
    try {
        await navigator.clipboard.writeText(tab.url);
        setStatus("Link copied");
        setTimeout(() => {
            if (status.textContent === "Link copied")
                setStatus("");
        }, 1600);
    }
    catch {
        setStatus("Couldn't copy link");
    }
});

fullscreenBtn.addEventListener("click", () => {
    if (document.fullscreenElement)
        document.exitFullscreen?.();
    else
        document.documentElement.requestFullscreen?.();
});

addEventListener("keydown", event => {
    if (!(event.ctrlKey || event.metaKey))
        return;
    switch (event.key) {
        case "l":
            event.preventDefault();
            addressBar.focus();
            addressBar.select();
            break;
        case "t":
            event.preventDefault();
            openTab(startUrl());
            break;
        case "w":
            if (activeId !== null) {
                event.preventDefault();
                closeTab(activeId);
            }
            break;
    }
});

applyBackground();
renderQuickLaunch();
renderShortcuts();
openTab(startUrl());
