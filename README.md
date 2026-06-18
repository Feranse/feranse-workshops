# Prompting with Intention — Workshop Deck

A self-contained, single-file presentation (`index.html`) for the AI workshop. No build step, no dependencies to install — it's pure HTML/CSS/JS plus a Google Fonts link.

## Deploy with GitHub Pages (takes ~2 minutes)

1. **Create a new repo** on GitHub (Settings → New repository). Public or private both work for Pages on a paid plan; public is required on the free plan.
2. **Upload `index.html`** to the root of the repo (drag-and-drop on the GitHub web UI works fine — no command line needed).
   - Optional: upload this `README.md` too, just for your own notes. It won't affect the site.
3. Go to the repo's **Settings → Pages**.
4. Under **Build and deployment → Source**, choose **Deploy from a branch**.
5. Under **Branch**, choose **main** (or whichever branch you uploaded to) and folder **/ (root)**. Click **Save**.
6. Wait about a minute, then refresh the Pages settings tab — it'll show your live URL, something like:
   `https://your-username.github.io/your-repo-name/`

That's it — open that link on the laptop you're presenting from (or AirPlay/HDMI it to a screen) and you're live.

## Using it during the workshop

- **Arrow keys (← / →)** or the **on-screen arrows** at the bottom move between slides.
- **Swipe** works on touch screens/tablets.
- The **dots** at the bottom let you jump to any slide directly — handy if you want to skip ahead or revisit one during Q&A.
- Every framework/feature slide has a small **source link** in the bottom-left corner if anyone wants to verify a claim live.

## Making edits later

Since it's one HTML file, you can just edit `index.html` directly on GitHub (click the pencil icon on the file page) or re-upload a new version — Pages will redeploy automatically within a minute or two of any push to the branch you selected.

## Note on Google Fonts

The deck loads Fraunces, Hanken Grotesk, and IBM Plex Mono from Google's font CDN. If you'll be presenting somewhere with **no internet access**, the deck will still render but will fall back to system fonts — everything will still work, just with a slightly different look. For full fidelity, presenting with a Wi-Fi connection (even just for the first load) is recommended.
