# Eirdom — Family Setup Guide
> Getting connected to the home network, streaming movies and TV,
> and requesting new content

---

## Welcome to Eirdom

Your home network is set up so that one account — your domain account
— gives you access to the family media library, lets you request new
movies and shows, and connects you to the secure home WiFi.

This guide covers everything you need to get set up on a new device.

---

## Step 1 — Connect to WiFi

### Phones, Tablets, Laptops

Connect to the **Eirdom** WiFi network. This is the main trusted
network for family devices.

> The **Eirdom-IoT** network is for smart home devices (TVs,
> speakers, thermostats). **Eirdom-Guest** is for visitors.

When connecting to **Eirdom** for the first time:

1. Select **Eirdom** from your WiFi list
2. You will be prompted for a username and password
3. Enter your domain credentials:
   - **Username:** your first name (e.g. `tyler`)
   - **Password:** your account password
4. If prompted to trust a certificate, tap **Trust** or **Connect**

> **Note for Windows laptops:** If the computer is joined to the
> domain, it will connect automatically. No manual steps needed.

---

## Step 2 — Watch Movies and TV (Jellyfin)

Jellyfin is the home media server — think of it as your private
Netflix. It has all the movies and TV shows in the library available
for streaming on any device.

### Recommended Apps by Device

| Device | App | Where to get it |
|--------|-----|----------------|
| Apple TV | Jellyfin | App Store |
| iPhone / iPad | Jellyfin | App Store |
| Fire TV Stick | Jellyfin | Amazon Appstore |
| Android TV / Shield | Jellyfin | Google Play |
| Android Phone | Jellyfin | Google Play |
| Windows / Mac | Browser | `https://jellyfin.eirdom.homes` |
| Roku | Jellyfin | Roku Channel Store |

### Setting Up the Jellyfin App

1. Open the Jellyfin app after installing
2. Tap **Add Server** (or **Get Started**)
3. Enter the server address:
   ```
   https://jellyfin.eirdom.homes
   ```
4. Enter your credentials:
   - **Username:** your domain username (e.g. `tyler`)
   - **Password:** your domain password
5. Tap **Sign In**

### For the Best Picture Quality

In the Jellyfin app settings, configure:

| Setting | Value | Why |
|---------|-------|-----|
| Max streaming bitrate | **Original** (or **Auto**) | Gets the full quality file |
| Direct Play | **Enabled** | Plays the file without re-encoding |
| Direct Stream | **Enabled** | Same as above, fallback option |

> **4K tip:** To watch 4K HDR content you need a capable device.
> Apple TV 4K, Fire TV Stick 4K, Nvidia Shield, and most modern
> Android TVs handle 4K HDR natively. Older devices may not be
> able to play 4K files.

---

## Step 3 — Request New Movies and TV Shows (Jellyseerr)

Jellyseerr is where you request new content to be added to the
library. Think of it as the "add to wishlist" button.

**Web address:** `https://requests.eirdom.homes`

### How to Request

1. Go to `https://requests.eirdom.homes` in a browser or the
   Jellyseerr app
2. Click **Sign In with Jellyfin**
3. Enter your Jellyfin credentials (same username and password)
4. Search for a movie or TV show using the search bar
5. Click the result → click **Request**
6. For TV shows, you can request specific seasons or the full series

Once requested, it will be added to the download queue automatically.
You'll get a notification when it's available in Jellyfin.

### Jellyseerr App

Available on iOS and Android — search for **Overseerr** (same app,
different name on mobile):

- iOS: search "Overseerr" in App Store
- Android: search "Overseerr" in Google Play
- Server URL: `https://requests.eirdom.homes`

---

## Step 4 — Away From Home

### Watching Jellyfin Outside the House

Jellyfin is accessible from outside the home at the same address:
`https://jellyfin.eirdom.homes`

This works on any device with an internet connection — just open the
Jellyfin app and you're connected. The connection goes through
Cloudflare for security.

> **Note on 4K outside the house:** 4K streams are large files.
> On a slow internet connection, 4K may buffer. The app will
> automatically fall back to a lower quality if your connection
> can't keep up.

### Requesting Content Away From Home

`https://requests.eirdom.homes` works from anywhere too.

---

## Frequently Asked Questions

**Q: I forgot my password. How do I reset it?**
Ask Tyler to reset it in the admin portal. Your password is the same
for Jellyfin, the WiFi, and everything else on the home network.

**Q: A movie I requested says "Processing" — when will it be ready?**
Downloads are queued automatically. Most movies appear within a few
hours depending on availability. TV show episodes may take longer.
You'll get a Jellyseerr notification when it lands.

**Q: The Jellyfin app can't find the server.**
Make sure you entered `https://jellyfin.eirdom.homes` — note the
`https://` at the beginning. If you're on a new device, check that
you're connected to the Eirdom WiFi (when at home) or have internet
access (when away).

**Q: My 4K movie is buffering.**
Try setting the bitrate to a lower option (e.g. 25 Mbps instead of
Original) in the Jellyfin app settings under **Quality**. This
triggers a lower quality stream that your connection can handle more
easily. At home this should not be an issue — only affects streaming
over the internet.

**Q: Can I download content to watch offline?**
Yes — the Jellyfin app supports downloading on iOS and Android.
Open a movie or episode → tap the download icon. Downloaded content
is only available in the app on that device.

**Q: I see content in Jellyfin that I didn't request.**
That's normal — the library includes content Tyler has downloaded
for the whole family.

---

## Quick Reference

| What | Address | Login |
|------|---------|-------|
| Watch movies / TV | `https://jellyfin.eirdom.homes` | Domain username + password |
| Request new content | `https://requests.eirdom.homes` | Same credentials (via Jellyfin) |
| Home WiFi | **Eirdom** | Domain username + password |

---

*Questions? Ask Tyler.*