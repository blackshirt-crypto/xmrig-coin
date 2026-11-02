# ⛏️⛏️ XMRig for COIN (Linux OS) ⛏️⛏️

**A patched build of XMRig 6.26.0 for mining COIN (coin-net), maintained by [Blackshirt Crypto](https://blkshirtpool.com).**

> ⚠️ **This miner is for COIN only.** Do not use it for Monero or any other coin — use the official XMRig for those.

---

## Credits

This is a fork of **[XMRig](https://github.com/xmrig/xmrig)**, the open-source CPU/GPU miner created and maintained by the XMRig team. All of the real work is theirs — we changed **one line** so it can mine COIN. XMRig's original README is kept in this repo as [`README-XMRIG.md`](README-XMRIG.md).

- Original miner: **XMRig** — https://github.com/xmrig/xmrig
- COIN project: **coin-net** — https://github.com/coin-net
- This fork: **Blackshirt Crypto** — https://blkshirtpool.com

---

## Why do I need a special miner?

COIN uses **RandomX v2** (`rx/2`). The official XMRig also supports `rx/2`, but it adds an extra final step (a "commitment") that is meant for Monero's future upgrade. The COIN node **does not** use that step.

The result: with official XMRig, every block you find is **rejected** by the COIN network (`Block not accepted`). You would mine for nothing.

This build turns that one step off:

```
src/crypto/randomx/randomx.cpp   line 62
Tweak_V2_COMMITMENT = 1;   →   Tweak_V2_COMMITMENT = 0;
```

That's the whole change (plus the version name `6.26.0-coin`). You can see it yourself in the commit history of the `coin` branch.

---

## What you need before you start

| You need | Why | How to check |
|---|---|---|
| A **64-bit Linux** computer | This release is built for Linux x86-64 | `uname -m` should print `x86_64` |
| At least **4 GB of RAM** (8 GB+ recommended) | RandomX uses about 2.5 GB while mining | `free -h` |
| A **COIN wallet** | Your mining rewards go to your wallet address | Use the official COIN wallet from https://github.com/coin-net |
| A **COIN node (`coind`) running and synced** | When solo mining, the miner gets its work from your own node | See Step 3 below |

---

## Step 1 — Download and verify the miner

Open a terminal and copy/paste these lines **one at a time**:

```bash
cd ~
wget https://github.com/blackshirt-crypto/xmrig-coin/releases/download/v6.26.0-coin/xmrig-6.26.0-coin-linux-static-x64.tar.gz
wget https://github.com/blackshirt-crypto/xmrig-coin/releases/download/v6.26.0-coin/SHA256SUMS
sha256sum -c SHA256SUMS
```

✅ You should see: `xmrig-6.26.0-coin-linux-static-x64.tar.gz: OK`

❌ If it says `FAILED` — **stop**. Delete the file and download it again. Never run a miner that fails the check.

Now unpack it:

```bash
tar -xzf xmrig-6.26.0-coin-linux-static-x64.tar.gz
cd xmrig-6.26.0-coin
ls
```

You should see: `xmrig`, `config.json`, `README.md`, `LICENSE`, `randomx_boost.sh`

---

## Step 2 — Get your COIN wallet address (the right one!)

Open your COIN wallet and copy your **primary address**.

| Address starts with | What it is | Can you mine to it? |
|---|---|---|
| **`C`** | Primary address | ✅ **Yes — use this one** |
| **`D`** | Subaddress (made with "Create new address") | ❌ **No** — the node will refuse it |

In the GUI wallet, your primary address is the **first** address on the **Receive** page (Account #0, address #0).
In the CLI wallet, type `address` — use the line marked `#0`.

---

## Step 3 — Make sure your COIN node is running and synced

Solo mining means **your own node** gives the miner its work. Start `coind` (or open the COIN GUI wallet in Local node mode) and let it finish syncing.

Check it from a terminal on the same computer:

```bash
curl -s http://127.0.0.1:38881/get_info | grep -E '"height"|"synchronized"'
```

✅ You want to see `"synchronized": true`

❌ `"synchronized": false` → wait a few minutes and check again.
❌ Nothing at all → your node isn't running. Start `coind` first.

---

## Step 4 — Put your address into the config

Replace `PASTE_YOUR_C_ADDRESS_HERE` below with your address from Step 2, then run the line:

```bash
sed -i 's/YOUR_COIN_C_ADDRESS/PASTE_YOUR_C_ADDRESS_HERE/' config.json
```

Check that it worked:

```bash
grep '"user"' config.json
```

✅ You should see your address, starting with `C`.

<details>
<summary>📄 What's inside config.json (click to expand)</summary>

```json
{
    "autosave": true,
    "cpu": true,
    "randomx": { "1gb-pages": true },
    "pools": [{
        "algo": "rx/2",
        "url": "127.0.0.1:38881",
        "user": "YOUR_COIN_C_ADDRESS",
        "pass": "x",
        "rig-id": "rig1",
        "daemon": true
    }]
}
```

- `"algo": "rx/2"` — **required**, do not change it
- `"url"` — where your node is (`127.0.0.1` = this same computer, port `38881`)
- `"daemon": true` — solo mining directly against your node
- `"rig-id"` — a name for this computer (optional)

</details>

---

## Step 5 — Start mining

```bash
sudo ./xmrig -c config.json
```

> 💡 **Why `sudo`?** It lets XMRig apply CPU tweaks (MSR) and large memory pages that make it **10–30% faster**. It works without `sudo` too — just slower.

### What good looks like

```
 * POOL #1      127.0.0.1:38881 algo rx/2
 net      new job from 127.0.0.1:38881 diff 860450 algo rx/2 height 7325
 msr      register values for "ryzen_17h" preset have been set successfully
 randomx  dataset ready
 cpu      READY threads 12/12
 miner    speed 10s/60s/15m 5239.4 n/a n/a H/s
```

When you find a block you'll see a green **`accepted`** line. 🎉
Newly mined coins are **locked for 60 blocks (about 2 hours)** before you can spend them — they will show in your wallet as locked.

Useful keys while XMRig is running:

| Key | Does |
|---|---|
| `h` | Show hashrate |
| `p` | Pause |
| `r` | Resume |
| `Ctrl+C` | Stop the miner |

---

## Mining from a different computer than your node

You can run the node on one computer and miners on others (home network or Tailscale).

**On the node computer**, start `coind` with a second, restricted RPC port for your miners:

```bash
./coind --rpc-restricted-bind-ip 0.0.0.0 --rpc-restricted-bind-port 38889 --confirm-external-bind
```

- Port `38881` stays private to the node computer (full access, your wallet uses it)
- Port `38889` is for miners (mining only, no admin commands)

**On each miner computer**, change the `url` in `config.json` to the node's IP and port `38889`:

```bash
sed -i 's/127.0.0.1:38881/NODE_IP_HERE:38889/' config.json
```

Test it from the miner computer first:

```bash
curl -s http://NODE_IP_HERE:38889/get_info | grep '"height"'
```

---

## Make it faster

| Tip | How | Gain |
|---|---|---|
| Run with `sudo` | `sudo ./xmrig -c config.json` | MSR tweak + 1 GB pages |
| Or apply the boost once per reboot | `sudo ./randomx_boost.sh` | Same MSR tweak without running XMRig as root |
| Turn on **XMP / DOCP** in your BIOS | RAM often runs at a slow default (e.g. 2133 MHz) | 10–15% |
| Let XMRig pick the threads | Don't hand-copy thread lists from other computers | Avoids lost speed |

---

## Troubleshooting

| You see | What it means | Fix |
|---|---|---|
| `Mining to subaddress is not supported yet` | Your address starts with `D` | Use your primary **`C`** address (Step 2) |
| `rejected ... "Block not accepted"` | You're running **official** XMRig, not this build | Run `./xmrig --version` — it must say **6.26.0-coin** |
| `Core is busy` | Your node is still syncing | Wait until `"synchronized": true` (Step 3) |
| `connect error: "connection refused"` | Node isn't running, or wrong IP/port | Start `coind`; check `url` in `config.json` |
| `connection refused` **only from another computer** | The node only listens to its own computer by default | Use the two-port setup in **Mining from a different computer** (port `38889`) |
| `Failed to bind` / `Address already in use` (in the node) | Two nodes running on the same computer (e.g. `coind` **and** the GUI wallet in Local node mode) | Run only one node. If `coind` is running, set the GUI to **Remote node → 127.0.0.1 : 38881** |
| No jobs / strange errors after connecting | Miner pointed at port `38880` (node-to-node port) | Miners use `38881` (same computer) or `38889` (other computers) — never `38880` |
| `FAILED TO APPLY MSR MOD` | XMRig wasn't run with `sudo` | Use `sudo ./xmrig -c config.json` |
| `failed to allocate RandomX dataset using 1GB pages` | Normal without root | Run with `sudo`, or ignore (regular huge pages still work) |

---

## FAQ

**Can I use my GPU?**
RandomX is designed for CPUs. GPUs mine it very poorly — use your CPU.

**What about the 1% donation?**
This build keeps XMRig's default 1% donation to the **XMRig developers**, who wrote the miner. It does not go to Blackshirt Crypto.

**Is there a pool?**
Not yet. A COIN pool is **planned** at **[blkshirtpool.com](https://blkshirtpool.com)**. Until then, solo mine against your own node as shown above.

**Windows / macOS?**
Linux only for now. A Windows build is planned.

---

## Build it yourself (optional)

If you'd rather compile than download:

```bash
sudo apt install -y git build-essential cmake libuv1-dev libssl-dev libhwloc-dev
git clone -b coin https://github.com/blackshirt-crypto/xmrig-coin.git
cd xmrig-coin
mkdir build && cd build
cmake ..
make -j$(nproc)
./xmrig --version          # should say 6.26.0-coin
```

---

## License

XMRig is licensed under the **GNU General Public License v3.0**, and so is this fork. See [`LICENSE`](LICENSE).
The full source code for every release of this build is in this repository.

Unofficial build — not affiliated with the XMRig team or the COIN developers.
