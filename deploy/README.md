# Running openGym on an always-on server

Goal: the website and the Android app sync through a server that is always on, so neither the
laptop nor a connection at the gym is needed. The Android app keeps a full copy on the phone and
logs workouts offline; it syncs whenever it can reach the server.

```
phone (APK, works offline) ──┐
                             ├── https://<host> ── Caddy (TLS) ── openGym web ── api ── ./data
laptop browser ──────────────┘
```

## 1. Get a server

Any Ubuntu 22.04/24.04 VPS with a public IPv4 works (1 GB RAM is plenty). Free option:
**Oracle Cloud Always Free**.

1. Sign up at <https://cloud.oracle.com> (a card is asked for verification; Always Free
   resources are not charged).
2. **Compute → Instances → Create instance**
   - Image: **Canonical Ubuntu 24.04**
   - Shape: **VM.Standard.A1.Flex** (Ampere, 1 OCPU / 6 GB, free). If it says "out of capacity",
     use **VM.Standard.E2.1.Micro** (also free) or try again later.
   - SSH keys: *Generate a key pair* and download the private key.
3. Open the web ports: on the instance page, click the **subnet → Security list → Add ingress
   rules**: source `0.0.0.0/0`, TCP, destination ports `80,443`.
4. Note the instance's **public IP address**.

Paid alternative with no capacity issues: Hetzner CX22 (~€4/month), same steps from 5 on.

## 2. Install openGym on it

From the laptop:

```bash
chmod 600 ~/Downloads/ssh-key-*.key
ssh -i ~/Downloads/ssh-key-*.key ubuntu@<PUBLIC_IP>
```

On the server:

```bash
sudo apt-get update && sudo apt-get install -y git
git clone -b vps-deploy https://github.com/solomon-mh/openGym.git
cd openGym
sudo ./deploy/setup-vps.sh
```

It installs Docker, opens the firewall, sets HTTPS for `<a-b-c-d>.sslip.io` (your IP with dashes;
no domain needed), starts everything and schedules a daily backup to `/var/backups/opengym`.
It prints your address at the end, e.g. `https://132-145-10-20.sslip.io`.

Own a domain? Point an A record at the server and run
`sudo OPENGYM_HOST=gym.yourdomain.com ./deploy/setup-vps.sh` instead. Choose before creating your
profile: passkeys are bound to the hostname.

## 3. Create your profile, then lock signup

1. On the laptop, open the printed address. Until you lock it, the browser first asks for the
   `setup` login the script printed (so nobody else can grab the admin profile). Then
   **Create profile**. The first profile becomes the admin.
2. **Settings → Account → Password**: set one. It is your way in from browsers that can't make a
   passkey, and your recovery if a passkey is lost.
3. Back on the server: `sudo ./deploy/setup-vps.sh lock`. Signup becomes invite-only and the
   `setup` login goes away. Pair the phone after this step.

## 4. Phone (Android)

1. On the phone, download the APK from
   <https://github.com/DuarteSantos8/openGym/releases/latest> and install it (allow installs from
   the browser when asked).
2. Open it and choose to **connect to a server**.
3. On the laptop: **Settings → Account → Pair the mobile app**. A code appears (valid 5 minutes).
4. In the app enter `https://<your address>` and the code.

At the gym, log as usual, with or without signal. Everything syncs when the phone is online.

## Day-to-day

| Task | Command (in `~/openGym` on the server) |
|---|---|
| Update to the newest release | `sudo ./deploy/setup-vps.sh update` |
| Backup now | `sudo ./deploy/setup-vps.sh backup` |
| Logs | `sudo docker compose -f docker-compose.yml -f deploy/compose.vps.yml logs -f` |
| Health check | `curl https://<your address>/api/health` |

Copy a backup to the laptop now and then:
`scp -i <key> ubuntu@<PUBLIC_IP>:/var/backups/opengym/*.tar.gz ~/opengym-backups/`.
In the app, **Settings → Data & backup → Export backup** also gives you your data as one JSON file.
