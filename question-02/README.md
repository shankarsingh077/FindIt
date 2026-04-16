# Question 02 - WRAITHNET Write-up

## Overview

This walkthrough covers the WRAITHNET challenge:

- Target: `https://final.ctfgeu.online/`
- Theme: hidden fragments, client-side traps, event-loop behavior, misleading observation paths
- Goal: recover the 4 fragments exposed across the page source and runtime behavior

The four recovered fragments are:

- `Karma_Police`
- `flag{base64_is_too_easy_try_again}`
- `flag{wr0ng_p4th_f00l}`
- `flag{m3cr0_m6cr0_sYnc}`

This challenge is built to waste your time if you only look at the obvious surface. Some clues are static, some only appear at runtime, and some are designed to mislead you into thinking you found the answer too early.

---

## What This Challenge Is Testing

The challenge description gives away the main ideas:

- the site is unstable
- observation changes behavior
- some fragments are hidden in places automation may miss
- at least one fragment is emergent rather than directly written

That tells us to check:

1. raw HTML comments
2. linked JavaScript
3. runtime-only state
4. clipboard behavior
5. timing and event-loop ordering
6. any anti-debug or observation-dependent branches

---

## Step 1: Fetch the Main Page

### Command

```bash
curl -iL https://final.ctfgeu.online/
```

### What this command does

- `curl` requests the page from the terminal
- `-i` includes the HTTP response headers
- `-L` follows redirects

### Why use it

The raw HTML often contains:

- comments
- hidden values
- asset references
- attributes not obvious in the rendered UI

### What the HTML immediately reveals

The source contained:

1. a hex string in an HTML comment
2. a Base64 string in another HTML comment
3. a suspicious `data-rc-blob` value
4. repeated log text with mixed Unicode lookalike characters
5. a linked script file: `sys.js`

The important hidden comments were:

```html
<!-- "4b61726d615f506f6c696365" -->
<!-- ZmxhZ3tiYXNlNjRfaXNfdG9vX2Vhc3lfdHJ5X2FnYWlufQ== -->
```

---

## Step 2: Decode the Static HTML Comments

### Command

```bash
python3 - <<'PY'
import base64
print(bytes.fromhex('4b61726d615f506f6c696365').decode())
print(base64.b64decode('ZmxhZ3tiYXNlNjRfaXNfdG9vX2Vhc3lfdHJ5X2FnYWlufQ==').decode())
PY
```

### What this command does

- `bytes.fromhex(...)` converts a hex string into raw bytes, then into readable text
- `base64.b64decode(...)` decodes the Base64 string

### Results

The two static fragments are:

- `Karma_Police`
- `flag{base64_is_too_easy_try_again}`

### Important note

The second one looks like an easy flag on purpose. The text itself says Base64 alone is too easy and to keep looking. So this is a fragment, but not the end of the challenge.

---

## Step 3: Pull the JavaScript

### Command

```bash
curl -sL https://final.ctfgeu.online/sys.js
```

### What this command does

- downloads the main JavaScript file quietly
- lets us inspect the logic that the browser runs

### Key parts of `sys.js`

The most important block is:

```js
let _f = [];
_f.push('f', 'l', 'a', 'g', '{');

let _m = [];
setTimeout(() => {
    _m.push('_', 'm', '6', 'c', 'r', '0');
    Promise.resolve().then(() => _m.push('_', 's', 'Y', 'n', 'c', '}'));
}, 0);

Promise.resolve().then(() => {
    _m.push('m', '3', 'c', 'r', '0');
});
```

And later:

```js
setTimeout(() => {
    const full0 = _f.join('') + _m.join('');
    document.getElementById('seq').innerText = "LOADED. SALT READY.";
    window._wS = _m;
}, 500);
```

This is the core runtime puzzle.

---

## Step 4: Solve the Emergent Sequence

The challenge text says:

> Somewhere inside this construct, there exists a sequence not written, not stored, but emerging.

That is exactly what the JavaScript is doing.

The fragment is built through event-loop ordering:

1. `_f` becomes `flag{`
2. the `Promise.resolve().then(...)` runs before the `setTimeout(..., 0)` callback
3. so `_m` first becomes `m3cr0`
4. then the timeout runs and pushes `_m6cr0`
5. then the nested Promise runs and pushes `_sYnc}`

So `_m` becomes:

```txt
m3cr0_m6cr0_sYnc}
```

Combined with `_f`, the full emergent fragment is:

```txt
flag{m3cr0_m6cr0_sYnc}
```

### Quick verification command

```bash
node - <<'NODE'
let _f = [];
_f.push('f', 'l', 'a', 'g', '{');

let _m = [];
setTimeout(() => {
  _m.push('_', 'm', '6', 'c', 'r', '0');
  Promise.resolve().then(() => _m.push('_', 's', 'Y', 'n', 'c', '}'));
}, 0);

Promise.resolve().then(() => {
  _m.push('m', '3', 'c', 'r', '0');
});

setTimeout(() => {
  console.log(_f.join('') + _m.join(''));
}, 10);
NODE
```

### Result

```txt
flag{m3cr0_m6cr0_sYnc}
```

---

## Step 5: Find the Clipboard Trap

Another important section in `sys.js` is:

```js
document.addEventListener('copy', (e) => {
    const sel = window.getSelection().toString();
    if (sel.includes('AUTH') || sel.includes('network')) {
        e.clipboardData.setData('text/plain', "SYS_LOG: " + unescape("%66%6C%61%67%7B%77%72%30%6E%67%5F%70%34%74%68%5F%66%30%30%6C%7D"));
        e.preventDefault();
    }
});
```

### What this means

If you copy the auth log, the site replaces your clipboard contents with a different string.

The `unescape(...)` payload decodes to:

```txt
flag{wr0ng_p4th_f00l}
```

### Why this matters

The page literally warns:

> Do not copy paste blindly.

So this fragment is intentionally delivered through clipboard interception. It is both:

- a real fragment exposed by the page
- a warning that careless copying will send you down the wrong path

### Quick decode command

```bash
node -e "console.log(unescape('%66%6C%61%67%7B%77%72%30%6E%67%5F%70%34%74%68%5F%66%30%30%6C%7D'))"
```

### Result

```txt
flag{wr0ng_p4th_f00l}
```

---

## Step 6: Runtime Observation and Anti-Debug Behavior

The script also contains this branch:

```js
let _dt = false;
setInterval(() => {
    const _s = performance.now();
    debugger;
    if ((performance.now() - _s) > 100) {
        _dt = true;
        document.body.classList.add('dt-open');
    }
}, 1000);
```

### What this does

- repeatedly triggers `debugger`
- if stepping or breakpoints slow execution enough, the page marks itself with `dt-open`
- CSS then slightly shifts the sector grid

This matches the challenge description:

- some structures respond differently when observed
- some values shift when measured

### Why this matters

This branch supports the story and creates misdirection, but it did not yield a cleaner fragment than the four listed above. It is a classic CTF trap:

- it encourages deep inspection
- it changes the presentation
- it makes the challenge feel more dynamic
- but the meaningful recoveries still come from the source, event loop, and clipboard trap

---

## Step 7: The Sector Grid and `data-rc-blob`

The page also includes:

- a `data-rc-blob`
- a sector grid with many `pd-*` classes
- a `Karma_Police` clue that looks like it could be a key

These are strong misdirection channels. They suggest:

- RC4 or another keyed transform
- width-based encoding
- observation-dependent rendering

In practice, they are useful because they keep you looking in the right direction, but the clean recoverable fragments for this solve came from:

1. the HTML comments
2. the event-loop-generated sequence
3. the clipboard trap

That is the shortest reliable solve path.

---

## Final Recovered Fragments

The four fragments recovered from the challenge are:

```txt
Karma_Police
flag{base64_is_too_easy_try_again}
flag{wr0ng_p4th_f00l}
flag{m3cr0_m6cr0_sYnc}
```

---

## Short Solve Flow

If you want the short version:

1. Fetch the HTML.
2. Decode the hex comment to get `Karma_Police`.
3. Decode the Base64 comment to get `flag{base64_is_too_easy_try_again}`.
4. Read `sys.js`.
5. Follow the Promise and `setTimeout` ordering to recover `flag{m3cr0_m6cr0_sYnc}`.
6. Inspect the `copy` event handler to recover `flag{wr0ng_p4th_f00l}`.
7. Treat the anti-debug and shifting grid as observation-based misdirection.

---

## What You Should Learn From This Challenge

This challenge teaches several useful habits:

### 1. Read both static and dynamic logic

Some clues are directly in the HTML. Others only appear after JavaScript runs.

### 2. Event-loop order matters

Promises and timeouts do not execute in the same order they appear in source.

### 3. Clipboard traps are real attack and puzzle techniques

What you copy is not always what you selected.

### 4. Anti-debug code can be signal and noise at the same time

It may reveal how the challenge is themed without being the shortest route to the answer.

### 5. Not every flag-looking string is equal

This challenge mixes:

- a visible encoded fragment
- an emergent runtime fragment
- a clipboard-delivered wrong-path fragment
- a standalone keyword-like fragment

That is why careful classification matters more than blindly submitting the first thing that looks correct.
