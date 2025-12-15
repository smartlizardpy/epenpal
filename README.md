# PenPal Desk — Gmail Sender (Vue + Vite)

Client-side Gmail sender with a PenPal-inspired letter/postcard UI. Users authorize Gmail in-browser, compose a letter or postcard (with optional photo), and the app posts directly to Gmail API.

## Setup
```sh
npm install
```

## Configure Gmail OAuth
1) In Google Cloud Console, enable Gmail API and create an OAuth Web client.
2) Authorized JavaScript origin: `http://localhost:5173` (Vite dev). Add production origin if deploying.
3) Redirect URI: same origin (`http://localhost:5173`).
4) Copy the Client ID and set `GOOGLE_CLIENT_ID` in `src/App.vue`.

## Run
- Dev: `npm run dev`
- Build: `npm run build`
- Preview build: `npm run preview`
- Deploy to GitHub Pages: `npm run deploy` (builds with `/epenpal/` base and pushes `dist` via `gh-pages`)

## How it works
- Google Identity Services provides an access token for the `gmail.send` scope.
- The browser builds a MIME email with the PenPal letter/postcard HTML, base64url encodes it, and POSTs to `https://gmail.googleapis.com/gmail/v1/users/me/messages/send`.
- No server secrets; everything runs in the client.

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```
