# Question 01 - NEXUS-7 CTF Write-up

## Overview

This document is a full beginner-friendly walkthrough for the **NEXUS-7** web challenge:

- Target: `https://challange2.ctfgeu.online/`
- Theme: client-side authentication, hidden logic, decoys, delayed diagnostics
- Goal: recover **4 submission values**
  - the **access code**
  - the **operator ID**
  - **flag 1**
  - **flag 2**

This write-up is designed so that someone reading it can understand:

- how to think about this challenge
- how to inspect a web challenge safely and methodically
- what each command does
- how to read misleading JavaScript
- how to identify real logic vs decoys
- how the final answers are derived

---

## Final Answers

These are the four values the challenge expects:

- Access code: `gh0st_1n_th3_l3dg3r`
- Operator ID: `TREVOR_P`
- Flag 1: `CTF{gh0st_l3dg3r_0p3r@t0r_tr3v0r}`
- Flag 2: `CTF{l3st3r_w@s_0p3r@t0r_c_@ll_@l0ng}`

If you want to learn how those were found, keep reading.

---

## What This Challenge Is Testing

This challenge is not about brute force or guessing. It is testing whether you can:

1. read the page source
2. inspect external JavaScript
3. notice misleading values and decoy flags
4. trace the actual code path used during authentication
5. follow delayed logic that only runs after successful auth
6. recover hidden data from encoded values

The story text already gives away the core idea:

- the key is hidden in JavaScript
- Base64 is involved
- encoding was layered
- the visible interface is not trustworthy
- something important appears only after waiting

That means our mindset should be:

- do not guess credentials
- do not trust labels like "legacy", "primary", or "secret"
- follow the code that actually executes

---

## Recommended Solving Mindset

When solving a web CTF challenge like this, use this order:

1. Inspect the HTML
2. Find all linked assets such as JavaScript and CSS
3. Read the JavaScript carefully
4. Identify the authentication function
5. Identify which variables are really used
6. Decode the real credential values
7. Search for hidden flows triggered after auth
8. Verify whether there are fake flags or decoy secrets

This challenge rewards patient reading more than advanced exploitation.

---

## Step 1: Fetch the Main Page

### Command

```bash
curl -iL https://challange2.ctfgeu.online/
```

### What this command does

- `curl` makes an HTTP request from the terminal
- `-i` tells `curl` to include the response headers
- `-L` tells `curl` to follow redirects if the server sends any

### Why use it

We want to retrieve the raw HTML, not just see the rendered page in a browser. Raw HTML often contains:

- comments
- hidden hints
- asset references
- script tags
- misleading UI text

### What we learned from the HTML

The HTML revealed several important things:

- the page loads `style.css`
- the page loads `app.js`
- the auth panel asks for:
  - `ACCESS KEY`
  - `OPERATOR ID`
- there is a hidden place where `flag1` is displayed after success
- the vault panel says:
  - "The betrayer is Operator C"
  - "The answer is buried deeper"
- the briefing contains a direct hint:
  - the auth key was Base64 encoded and hidden in the validator

### Important lesson

The HTML gave us the **direction**, but not the answer. The real logic is in the JavaScript file.

---

## Step 2: Fetch the JavaScript

### Command

```bash
curl -sL https://challange2.ctfgeu.online/app.js
```

### What this command does

- `curl` downloads the JavaScript file
- `-s` means "silent", so progress output is hidden
- `-L` follows redirects if needed

### Why this matters

In many web CTF challenges, the client-side JavaScript contains:

- validation logic
- hidden constants
- encoded secrets
- fake clues
- delayed execution

This challenge is exactly that kind of challenge.

---

## Step 3: Identify the Authentication Object

Inside `app.js`, the most important block is:

```js
const _NEXUS_AUTH = {
  _primary: "amUwYXRfeDNsX3U0dW5feTB5",
  _diagnostic_checksum: "dHUwZmdfMWFfZ3UzX3kzcXQzZQ==",
  _legacy_token: "TkVYVVM3QVVUSEtFWQ==",
  _op_hash: "P_ROVERT",
  _op_legacy: "MIKEY_D_ADMIN",
  _config: {
    useChecksum: false,
    useLegacy: true,
    _internalMode: "diag"
  }
};
```

### Why this block matters

This looks like it stores all the possible authentication-related values.

But this is where the challenge tries to trick you:

- `_primary` looks important
- `_legacy_token` looks important
- `_op_legacy` looks important
- the config names are misleading

The only way to know what matters is to inspect the validator.

---

## Step 4: Find the Real Validator Function

The critical function is:

```js
function _nexus_validate_operator(inputKey, inputOp) {
  let authBlob;
  if (_NEXUS_AUTH._config._internalMode === "diag") {
    authBlob = _NEXUS_AUTH._diagnostic_checksum;
  } else if (_NEXUS_AUTH._config.useLegacy) {
    authBlob = _NEXUS_AUTH._legacy_token;
  } else {
    authBlob = _NEXUS_AUTH._primary;
  }

  const decoded = _rot13(_b64d(authBlob));
  const correctOp = _NEXUS_AUTH._op_hash.split("").reverse().join("");

  const keyMatch = (inputKey.trim() === decoded);
  const opMatch = (inputOp.trim().toUpperCase() === correctOp.toUpperCase());

  return { keyMatch, opMatch, _internal_decoded: decoded };
}
```

### What this function is doing

It accepts two user inputs:

- `inputKey`
- `inputOp`

Then it:

1. decides which auth blob to use
2. Base64-decodes that blob
3. applies ROT13 to the decoded text
4. reverses `_op_hash` to get the correct operator ID
5. compares your input against those computed values

### The key observation

The code does **not** use `_primary`.

It does **not** use `_op_legacy`.

It does **not** use the legacy operator string.

Because `_internalMode === "diag"`, the validator uses:

```js
_NEXUS_AUTH._diagnostic_checksum
```

That is the real source of the correct access key.

---

## Step 5: Decode the Real Access Code

The real auth blob is:

```txt
dHUwZmdfMWFfZ3UzX3kzcXQzZQ==
```

The code first calls `_b64d`, which is just Base64 decode:

```js
function _b64d(str) {
  try { return atob(str); } catch(e) { return ""; }
}
```

Then it calls `_rot13`:

```js
function _rot13(str) {
  return str.replace(/[a-zA-Z]/g, c => {
    const base = c <= 'Z' ? 65 : 97;
    return String.fromCharCode(((c.charCodeAt(0) - base + 13) % 26) + base);
  });
}
```

### Command used to decode it

```bash
node -e "const b='dHUwZmdfMWFfZ3UzX3kzcXQzZQ=='; const s=Buffer.from(b,'base64').toString(); const r=s.replace(/[a-zA-Z]/g,c=>String.fromCharCode((c<='Z'?65:97)+((c.charCodeAt(0)-(c<='Z'?65:97)+13)%26))); console.log('b64:',s); console.log('rot13:',r);"
```

### What this command does

- `node -e` runs JavaScript directly from the terminal
- `Buffer.from(b, 'base64').toString()` decodes Base64
- the `.replace(...)` logic performs ROT13
- `console.log(...)` prints the intermediate and final values

### Output

```txt
b64: tu0fg_1a_gu3_y3qt3e
rot13: gh0st_1n_th3_l3dg3r
```

### Result

So the real access code is:

```txt
gh0st_1n_th3_l3dg3r
```

---

## Step 6: Recover the Operator ID

The validator computes the operator like this:

```js
const correctOp = _NEXUS_AUTH._op_hash.split("").reverse().join("");
```

The value stored in `_op_hash` is:

```txt
P_ROVERT
```

Reverse it:

```txt
TREVOR_P
```

### Command used

```bash
node -e "const s='P_ROVERT'; console.log(s.split('').reverse().join(''));"
```

### What this command does

- defines the string `P_ROVERT`
- splits it into an array of characters
- reverses the array
- joins it back into one string
- prints the result

### Result

The operator ID is:

```txt
TREVOR_P
```

---

## Step 7: Understand Why Other Values Are Wrong

This challenge includes decoys on purpose.

### Decoy 1: `_primary`

```txt
amUwYXRfeDNsX3U0dW5feTB5
```

If you decode it the same way, it becomes:

```txt
wr0ng_k3y_h4ha_l0l
```

That is an intentional joke and trap.

### Decoy 2: `_legacy_token`

```txt
TkVYVVM3QVVUSEtFWQ==
```

This Base64-decodes to:

```txt
NEXUS7AUTHKEY
```

After ROT13 it becomes:

```txt
ARKHF7NHGUXRL
```

This is also not the real answer because the code path never uses it while `_internalMode` is `"diag"`.

### Decoy 3: `_SYSTEM_TOKEN`

Later in the file, there is a fake flag:

```js
const _SYSTEM_TOKEN = "CTF{f@ke_tr@il_n0th1ng_h3re_k33p_l00k1ng}";
```

The console greeting explicitly warns that this is a decoy:

```js
console.log("%c[NEXUS-7] Warning: _SYSTEM_TOKEN is a decoy. Don't trust what you find first.", styles[2]);
```

### Important lesson

Never submit the first flag-looking string you see in JavaScript. Always verify whether the code actually uses it.

---

## Step 8: Find Flag 1

After successful authentication, the code does this:

```js
const f1 = _buildFlag1();
flag1Box.textContent = f1;
vaultFlag.textContent = f1;
```

So we inspect `_buildFlag1()`:

```js
function _buildFlag1() {
  const _p = ["CTF{", "gh0st_", "l3dg3r_", "0p3r@t0r_", "tr3v0r}"];
  return _p.join("");
}
```

### What this means

Flag 1 is directly assembled by concatenating string fragments.

### Result

```txt
CTF{gh0st_l3dg3r_0p3r@t0r_tr3v0r}
```

---

## Step 9: Notice the Delayed Hint

This is where many solvers stop too early.

After successful login, the code also does:

```js
setTimeout(_shadowCheck, 7000);
```

This means a second hidden flow runs **7 seconds after auth**.

The page also displays a hint after auth:

- diagnostics run in the dark
- some things only surface if you give them time
- check where no one looks

That tells us the second flag is not shown immediately in the visible UI.

---

## Step 10: Read the Shadow Logic

The relevant functions are:

```js
function _shadowInit(authKey) {
  _authKeyUsed = authKey;
  _SYS._authTime = Date.now();
  setTimeout(_shadowCheck, 7000);
}
```

```js
function _shadowCheck() {
  if (!_authKeyUsed) return;

  const realKey = _rot13(_b64d(_NEXUS_AUTH._diagnostic_checksum));
  if (_authKeyUsed.trim() !== realKey) return;

  const elapsed = (Date.now() - _SYS._authTime) / 1000;
  if (elapsed < 5 || elapsed > 30) return;

  _shadowReady = true;
  _shadowPayload = _assembleFlag2();

  const footer = document.querySelector("footer");
  if (footer) {
    const obf = _shadowPayload.split("").map(c =>
      String.fromCharCode(c.charCodeAt(0) ^ 0x42)
    ).join("");
    footer.dataset.nx7diag = btoa(obf);
  }
}
```

### What this tells us

Flag 2 is only prepared if all of these are true:

1. auth happened
2. the correct key was used
3. enough time has passed
4. not too much time has passed

If successful, the payload is:

- generated by `_assembleFlag2()`
- XOR-obfuscated with `0x42`
- Base64-encoded
- stored in `footer.dataset.nx7diag`

### Why this matters

The second flag is hidden in the page footer dataset, not displayed openly.

That matches the clue:

> Check where no one looks.

---

## Step 11: Understand How Flag 2 Is Built

The code defines:

```js
const _DIAG = {
  modes: {
    alpha:   "0sXX6g==",
    beta:    "/aLi5aI=",
    gamma:   "487m0eI=",
    delta:   "zqLk4aM=",
    epsilon: "zqHhouM=",
    zeta:    "0eWh484=",
    eta:     "zqThxqQ=",
    theta:   "8s7R/f0=",
    iota:    "ztH9of8=",
    kappa:   "9uw="
  },
  _priority: [0, 1, 2, 4, 5, 7, 8, 9],
  _legacy_priority: [0, 2, 4, 6, 8, 1, 3, 5, 7, 9]
};
```

Then:

```js
function _shadowKey() {
  const op = _NEXUS_AUTH._op_hash.split("").reverse().join("");
  return op.split("").reduce((a, c) => a + c.charCodeAt(0), 0) % 256;
}
```

Then:

```js
function _xorStr(b64str, key) {
  const raw = atob(b64str);
  return raw.split("").map(c => String.fromCharCode(c.charCodeAt(0) ^ key)).join("");
}
```

And finally:

```js
function _assembleFlag2() {
  const key = _shadowKey();
  const modeVals = Object.values(_DIAG.modes);
  const order = _DIAG._priority;

  let assembled = "";
  for (const idx of order) {
    const fragment = modeVals[idx];
    assembled += _xorStr(fragment, key);
  }

  return assembled;
}
```

### What is happening here

Flag 2 is built from fragments:

1. take the operator ID
2. compute a numeric key from the character codes
3. Base64-decode each fragment
4. XOR each byte with that key
5. reorder fragments using `_priority`
6. concatenate them into the final string

---

## Step 12: Compute the Shadow Key

The operator ID is:

```txt
TREVOR_P
```

The key is:

```js
sum of character codes % 256
```

### Command used

```bash
node - <<'NODE'
const op='TREVOR_P';
const key=op.split('').reduce((a,c)=>a+c.charCodeAt(0),0)%256;
console.log(key);
NODE
```

### Result

```txt
145
```

So the XOR key is `145`.

---

## Step 13: Decode the Flag 2 Fragments

### Command used

```bash
node - <<'NODE'
const modes={
  alpha:'0sXX6g==',
  beta:'/aLi5aI=',
  gamma:'487m0eI=',
  delta:'zqLk4aM=',
  epsilon:'zqHhouM=',
  zeta:'0eWh484=',
  eta:'zqThxqQ=',
  theta:'8s7R/f0=',
  iota:'ztH9of8=',
  kappa:'9uw='
};
const order=[0,1,2,4,5,7,8,9];
const op='TREVOR_P';
const key=op.split('').reduce((a,c)=>a+c.charCodeAt(0),0)%256;
const vals=Object.values(modes);
function xorStr(b64,key){
  const raw=Buffer.from(b64,'base64');
  return [...raw].map(ch=>String.fromCharCode(ch^key)).join('');
}
let assembled='';
for(const idx of order){
  const frag=xorStr(vals[idx],key);
  console.log(idx, JSON.stringify(frag));
  assembled+=frag;
}
console.log('assembled', assembled);
NODE
```

### What this command does

- defines the same data structure as the challenge
- reconstructs the same XOR key
- decodes every fragment in the same order as the live code
- prints each recovered piece
- prints the fully assembled string

### Output

```txt
0 "CTF{"
1 "l3st3"
2 "r_w@s"
4 "_0p3r"
5 "@t0r_"
7 "c_@ll"
8 "_@l0n"
9 "g}"
assembled CTF{l3st3r_w@s_0p3r@t0r_c_@ll_@l0ng}
```

### Result

Flag 2 is:

```txt
CTF{l3st3r_w@s_0p3r@t0r_c_@ll_@l0ng}
```

---

## Step 14: Why the Challenge Says "Some Things Only Surface if You Wait"

This is an important design detail.

Even though we can statically recover Flag 2 by reading the code, the application itself only prepares the hidden diagnostic payload after a delay.

That means there are two ways to solve it:

### Method A: Static analysis

Read the code and reconstruct the hidden logic yourself.

### Method B: Dynamic analysis

1. enter the correct access code
2. enter the correct operator ID
3. click authenticate
4. wait about 7 seconds
5. inspect the `footer` element or call the recovery function

The recovery helper exposed by the page is:

```js
window._nx7_diagnostic_flush = _nx7_recovery_mode;
```

So in the browser console, after authenticating and waiting, you could call:

```js
_nx7_diagnostic_flush()
```

That would return the hidden payload if it has been prepared.

---

## Step 15: Decoys and Misleading Structure

This challenge is good practice because it deliberately lies to you.

Here are the main traps:

### Misleading variable names

- `_primary` sounds correct but is wrong
- `_legacy_token` sounds important but is unused in the real path
- `_op_legacy` sounds like a valid operator but is wrong

### Fake secret values

- `_SYSTEM_TOKEN` looks like a flag but is fake
- `_INTERNAL_SECRET` looks important but is irrelevant
- `_SHADOW_MAP` looks promising but is a dead end

### Misleading config

The config says:

```js
useChecksum: false
useLegacy: true
_internalMode: "diag"
```

A careless solver might think `useLegacy: true` means the legacy token is the answer.

But the `if` statement checks `_internalMode === "diag"` first, so the diagnostic checksum overrides that.

### Hidden time gate

Flag 2 is not visible immediately. If you stop after unlocking the vault, you miss the second half of the challenge.

---

## Step 16: Full Solving Flow in Plain English

If you had to explain this challenge to someone in one smooth sequence, it would be:

1. Open the page source.
2. Notice that the challenge hints the auth key is hidden in JavaScript.
3. Download `app.js`.
4. Find the `_NEXUS_AUTH` object and the `_nexus_validate_operator()` function.
5. Verify which auth blob is actually selected.
6. See that `_internalMode === "diag"` forces the validator to use `_diagnostic_checksum`.
7. Base64-decode that value.
8. Apply ROT13 to the decoded text.
9. Recover the access code `gh0st_1n_th3_l3dg3r`.
10. Reverse `_op_hash` to recover `TREVOR_P`.
11. Inspect what happens after successful authentication.
12. Find `_buildFlag1()` and recover Flag 1.
13. Notice `_shadowInit()` and the delayed `_shadowCheck()`.
14. Follow `_assembleFlag2()` and reconstruct the second hidden flag from XOR-decoded fragments.
15. Ignore the fake flag and other decoys.

---

## Step 17: Minimal Command Set to Solve the Challenge

If you only want the essential commands, these are enough:

### Fetch HTML

```bash
curl -iL https://challange2.ctfgeu.online/
```

### Fetch JavaScript

```bash
curl -sL https://challange2.ctfgeu.online/app.js
```

### Decode the access code

```bash
node -e "const b='dHUwZmdfMWFfZ3UzX3kzcXQzZQ=='; const s=Buffer.from(b,'base64').toString(); const r=s.replace(/[a-zA-Z]/g,c=>String.fromCharCode((c<='Z'?65:97)+((c.charCodeAt(0)-(c<='Z'?65:97)+13)%26))); console.log(r);"
```

### Decode the operator ID

```bash
node -e "console.log('P_ROVERT'.split('').reverse().join(''))"
```

### Rebuild Flag 2

```bash
node - <<'NODE'
const modes={alpha:'0sXX6g==',beta:'/aLi5aI=',gamma:'487m0eI=',delta:'zqLk4aM=',epsilon:'zqHhouM=',zeta:'0eWh484=',eta:'zqThxqQ=',theta:'8s7R/f0=',iota:'ztH9of8=',kappa:'9uw='};
const order=[0,1,2,4,5,7,8,9];
const op='TREVOR_P';
const key=op.split('').reduce((a,c)=>a+c.charCodeAt(0),0)%256;
const vals=Object.values(modes);
function xorStr(b64,key){const raw=Buffer.from(b64,'base64'); return [...raw].map(ch=>String.fromCharCode(ch^key)).join('');}
let out='';
for(const idx of order) out+=xorStr(vals[idx],key);
console.log(out);
NODE
```

---

## Step 18: What You Should Learn From This Challenge

This challenge teaches several very useful CTF habits:

### 1. Always inspect the real execution path

Do not trust labels or variable names. Follow the actual `if` conditions.

### 2. Encoding is not security

Base64, ROT13, reversal, and XOR are all reversible if you can read the code.

### 3. Web challenges often hide answers in client-side logic

If the browser has everything it needs to validate your input, then you probably have everything you need too.

### 4. Decoys are common

If a challenge is story-driven, fake secrets are often planted to punish shallow inspection.

### 5. Delayed behavior matters

Timers, event handlers, hidden datasets, and console helpers often contain the second half of the challenge.

---

## Step 19: Beginner Notes on the Tools Used

If you are new, here is what each tool contributed:

### `curl`

Used to fetch web content directly from the terminal.

Good for:

- HTML
- JavaScript
- CSS
- APIs
- headers

### `node -e`

Used to run short JavaScript snippets directly in the terminal.

Good for:

- Base64 decoding
- string reversal
- replaying challenge logic
- quickly testing transformations

### Browser developer tools

Useful for:

- reading source files
- inspecting the DOM
- watching `data-*` attributes
- calling hidden functions from the console
- observing delayed state changes

---

## Step 20: Final Submission Set

Submit these four values:

```txt
gh0st_1n_th3_l3dg3r
TREVOR_P
CTF{gh0st_l3dg3r_0p3r@t0r_tr3v0r}
CTF{l3st3r_w@s_0p3r@t0r_c_@ll_@l0ng}
```

---

## Short Summary

The challenge pretends the answer is buried behind a secure UI, but all important logic lives on the client side. The real access code comes from `_diagnostic_checksum`, not the primary or legacy values. The operator ID is simply `_op_hash` reversed. Flag 1 is directly built after successful auth. Flag 2 is hidden behind delayed diagnostic logic, assembled from XOR-decoded fragments, then tucked into the footer dataset.

If you solve this challenge correctly, the real skill you demonstrate is not "decoding strings." It is recognizing which parts of a noisy codebase actually matter.
