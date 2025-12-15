<script setup>
import { computed, onMounted, reactive, ref } from 'vue'

const GOOGLE_CLIENT_ID = '686447695619-c4e4cffgkav62dsgtblg0tvuat9bj69u.apps.googleusercontent.com'
const GMAIL_SCOPE = 'https://www.googleapis.com/auth/gmail.send'

const palette = {
  orange: '#FF7A3D',
  paper: '#FFF9F3',
  bg: '#EFE9E4',
  ink: '#2B2B2B',
  muted: '#7A716C',
  border: '#E7D7C8',
}

const mode = ref('letter')
const accessToken = ref(null)
const tokenExpiry = ref(null)
const tokenClient = ref(null)
const sending = ref(false)

const letter = reactive({ to: '', subject: '', message: '' })
const postcard = reactive({ to: '', subject: '', message: '', image: '' })

const statusLetter = reactive({ text: '', isError: false })
const statusPostcard = reactive({ text: '', isError: false })

const authLabel = computed(() => (accessToken.value ? 'Authorized for Gmail send' : 'Not authorized'))

function onUpload(event) {
  const file = event.target.files[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (e) => {
    postcard.image = e.target.result
  }
  reader.readAsDataURL(file)
}

function escapeHtml(value) {
  return String(value || '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')
}

function base64Encode(str) {
  const bytes = new TextEncoder().encode(str)
  let binary = ''
  for (const b of bytes) binary += String.fromCharCode(b)
  return btoa(binary)
}

function base64UrlEncode(str) {
  return base64Encode(str).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
}

function buildLetterHtml({ to, subject, message }) {
  const safeMessage = escapeHtml(message).replace(/\n/g, '<br>')
  const safeSubject = escapeHtml(subject)
  const safeTo = escapeHtml(to)
  const formattedDate = new Date().toLocaleDateString('en-GB', {
    weekday: 'long',
    day: 'numeric',
    month: 'long',
    year: 'numeric',
  })
  return `
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>${safeSubject || 'Just Checking In Today'}</title>
</head>

<body style="margin:0;padding:0;background:#f2efe9;">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0">
    <tr>
      <td align="center" style="padding:26px 12px;">

        <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0"
               style="
                 max-width:640px;
                 background:#fffdf8;
                 border:1px solid #ded4c7;
                 font-family:Georgia,'Times New Roman',serif;
               ">

          <tr>
            <td style="
              padding:4px;
              background:
              repeating-linear-gradient(
                45deg,
                #b94a48 0px,
                #b94a48 8px,
                #ffffff 8px,
                #ffffff 16px,
                #357ab8 16px,
                #357ab8 24px,
                #ffffff 24px,
                #ffffff 32px
              );
            "></td>
          </tr>

          <tr>
            <td style="padding:16px 24px 4px 24px;font-size:13px;color:#6f6a63;">
              United Kingdom<br>
              Air Mail
            </td>
          </tr>

          <tr>
            <td style="padding:0 24px 14px 24px;font-size:13px;color:#6f6a63;">
              Sent to: ${safeTo}<br>
              <span style="font-style:italic;">
                ${formattedDate}
              </span>
            </td>
          </tr>

          <tr>
            <td style="
              padding:0 24px 26px 24px;
              font-size:17px;
              line-height:1.75;
              color:#2b2b2b;
            ">

              <h2 style="margin:0 0 16px 0;font-size:18px;font-weight:700;">${safeSubject || 'The letter title'}</h2>

              <div style="margin:0 0 16px 0;">
                ${safeMessage || 'I just wanted to write and check in with you today. Nothing important really happened, but it felt like one of those days where I wanted to send an actual letter instead of a quick message.'}
              </div>

            </td>
          </tr>

          <tr>
            <td style="
              padding:12px 24px 18px 24px;
              font-size:11px;
              color:#8a837c;
              border-top:1px solid #e1d8cc;
              font-style:italic;
            ">
              sent as an electronic letter
            </td>
          </tr>

        </table>

      </td>
    </tr>
  </table>
</body>
</html>
  `
}

function buildPostcardHtml({ to, subject, message, image }) {
  const safeMessage = escapeHtml(message).replace(/\n/g, '<br>')
  const safeSubject = escapeHtml(subject)
  const safeTo = escapeHtml(to)
  const imageSrc =
    image ||
    'https://cdn.britannica.com/32/252532-050-A7B608E6/Durham-Cathedral-Durham-England-UK-United-Kingdom.jpg'
  const caption =
    image ? 'Your attached postcard photo' : 'Durham Cathedral, overlooking the River Wear'
  return `
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>${safeSubject || 'Postcard from Durham'}</title>
</head>

<body style="margin:0;padding:0;background:#f2efe9;">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0">
    <tr>
      <td align="center" style="padding:28px 12px;">

        <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0"
               style="max-width:640px;
                      background:#fffdf8;
                      border:1px solid #e1d8cc;
                      font-family:Georgia,'Times New Roman',serif;">

          <tr>
            <td style="
              padding:6px;
              background:
              repeating-linear-gradient(
                45deg,
                #c0392b 0px,
                #c0392b 10px,
                #ffffff 10px,
                #ffffff 20px,
                #2980b9 20px,
                #2980b9 30px,
                #ffffff 30px,
                #ffffff 40px
              );
            "></td>
          </tr>

          <tr>
            <td style="padding:14px 18px 10px 18px;font-size:13px;color:#6f6a63;">
              <table width="100%" role="presentation">
                <tr>
                  <td>Durham, England 🇬🇧</td>
                  <td align="right">Air Mail ✈️</td>
                </tr>
              </table>
            </td>
          </tr>

          <tr>
            <td style="padding:0 18px;">
              <img
                src="${imageSrc}"
                alt="${safeSubject || 'Postcard image'}"
                style="
                  width:100%;
                  display:block;
                  border:1px solid #e1d8cc;
                "
              >
            </td>
          </tr>

          <tr>
            <td style="padding:6px 18px 14px 18px;
                       font-size:12px;
                       color:#7a746c;
                       font-style:italic;">
              ${caption}
            </td>
          </tr>

          <tr>
            <td style="padding:0 18px 20px 18px;
                       font-size:17px;
                       line-height:1.7;
                       color:#2b2b2b;">

              <p style="margin:0 0 14px 0;">${safeSubject || 'Postcard from Durham'}</p>

              <p style="margin:0 0 14px 0;">
                ${safeMessage ||
                  'I saw this today and thought you’d like it. Durham feels quiet in a really nice way — old stone, cold air, and a lot of walking. It’s one of those places that makes you slow down without noticing.'}
              </p>

              <p style="margin:0;">
                <strong>To: ${safeTo || 'a friend afar'}</strong>
              </p>

            </td>
          </tr>

          <tr>
            <td style="padding:12px 18px 18px 18px;
                       font-size:12px;
                       color:#7a746c;
                       border-top:1px solid #e1d8cc;">
              An electronic postcard — sent slowly, read whenever you like.
            </td>
          </tr>

        </table>

      </td>
    </tr>
  </table>
</body>
</html>
  `
}

function buildRawMessage({ to, subject, html }) {
  const encodedSubject = base64Encode(subject || '')
  const mime = [
    'From: me',
    `To: ${to}`,
    'Content-Type: text/html; charset=utf-8',
    'MIME-Version: 1.0',
    `Subject: =?utf-8?B?${encodedSubject}?=`,
    '',
    html,
  ].join('\n')
  return base64UrlEncode(mime)
}

async function sendEmail({ to, subject, html }) {
  if (!accessToken.value) throw new Error('Authorize Gmail first')
  const raw = buildRawMessage({ to, subject, html })
  const response = await fetch('https://gmail.googleapis.com/gmail/v1/users/me/messages/send', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${accessToken.value}`,
    },
    body: JSON.stringify({ raw }),
  })
  if (!response.ok) {
    const err = await response.json().catch(() => ({}))
    throw new Error(err.error?.message || 'Failed to send email')
  }
}

function persistAuth(token, expiresAt) {
  if (!token) {
    localStorage.removeItem('penpal_gmail_auth')
    return
  }
  localStorage.setItem(
    'penpal_gmail_auth',
    JSON.stringify({
      token,
      expiresAt,
    })
  )
}

function loadPersistedAuth() {
  const raw = localStorage.getItem('penpal_gmail_auth')
  if (!raw) return null
  try {
    const parsed = JSON.parse(raw)
    if (parsed.expiresAt && Date.now() > parsed.expiresAt) {
      localStorage.removeItem('penpal_gmail_auth')
      return null
    }
    return parsed
  } catch {
    return null
  }
}

function isTokenValid() {
  return accessToken.value && tokenExpiry.value && Date.now() < tokenExpiry.value
}

function setAuthState(token, expiresInSeconds) {
  accessToken.value = token || null
  tokenExpiry.value = token && expiresInSeconds ? Date.now() + expiresInSeconds * 1000 : null
  persistAuth(accessToken.value, tokenExpiry.value)
}

function initGoogleAuth() {
  const statuses = [statusLetter, statusPostcard]
  if (!window.google || !google.accounts || !google.accounts.oauth2) {
    statuses.forEach((s) => {
      s.text = 'Google auth not loaded. Check your network.'
      s.isError = true
    })
    return
  }
  if (!GOOGLE_CLIENT_ID) {
    statuses.forEach((s) => {
      s.text = 'Set GOOGLE_CLIENT_ID in App.vue to your OAuth client ID.'
      s.isError = true
    })
  }
  tokenClient.value = google.accounts.oauth2.initTokenClient({
    client_id: GOOGLE_CLIENT_ID,
    scope: GMAIL_SCOPE,
    callback: (resp) => {
      if (resp && resp.access_token) {
        setAuthState(resp.access_token, resp.expires_in || 3600)
        statuses.forEach((s) => {
          s.text = 'Ready to send via Gmail.'
          s.isError = false
        })
      } else {
        statuses.forEach((s) => {
          s.text = 'Authorization failed.'
          s.isError = true
        })
      }
    },
  })
}

function authorize() {
  const statuses = [statusLetter, statusPostcard]
  if (isTokenValid()) {
    statuses.forEach((s) => {
      s.text = 'Using saved Gmail authorization.'
      s.isError = false
    })
    return
  }
  if (!GOOGLE_CLIENT_ID) {
    statuses.forEach((s) => {
      s.text = 'Set GOOGLE_CLIENT_ID in App.vue to your OAuth client ID.'
      s.isError = true
    })
    return
  }
  if (!tokenClient.value) {
    initGoogleAuth()
  }
  if (!tokenClient.value) return
  tokenClient.value.requestAccessToken({ prompt: accessToken.value ? '' : 'consent' })
}

async function handleSend(type) {
  const statusObj = type === 'letter' ? statusLetter : statusPostcard
  statusObj.text = ''
  statusObj.isError = false
  sending.value = true
  try {
    let payload
    if (type === 'letter') {
      const { to, subject, message } = letter
      if (!to || !message) throw new Error('Please fill in the recipient and your message.')
      payload = {
        to: to.trim(),
        subject: (subject || 'A letter for you').trim(),
        html: buildLetterHtml({ to, subject, message }),
      }
    } else {
      const { to, subject, message, image } = postcard
      if (!to || !message) throw new Error('Please fill in the recipient and your message.')
      payload = {
        to: to.trim(),
        subject: (subject || 'A postcard for you').trim(),
        html: buildPostcardHtml({ to, subject, message, image }),
      }
    }
    await sendEmail(payload)
    statusObj.text = 'Sent via Gmail ✨'
    statusObj.isError = false
  } catch (err) {
    statusObj.text = err.message
    statusObj.isError = true
  } finally {
    sending.value = false
  }
}

onMounted(() => {
  const saved = loadPersistedAuth()
  if (saved && saved.token && saved.expiresAt && Date.now() < saved.expiresAt) {
    setAuthState(saved.token, (saved.expiresAt - Date.now()) / 1000)
    statusLetter.text = 'Using saved Gmail authorization.'
    statusPostcard.text = 'Using saved Gmail authorization.'
  }
  setTimeout(() => initGoogleAuth(), 300)
})
</script>

<template>
  <div class="center" v-cloak>
    <div class="auth-row">
      <div class="auth-pill">{{ authLabel }}</div>
      <button class="auth-btn" type="button" @click="authorize">Authorize Gmail</button>
    </div>

    <div class="switcher">
      <button class="switch-btn" :class="{ active: mode === 'letter' }" @click="mode = 'letter'">✉️ Letter</button>
      <button class="switch-btn" :class="{ active: mode === 'postcard' }" @click="mode = 'postcard'">
        📮 Postcard
      </button>
    </div>

    <div id="letterMode" class="paper" :class="{ show: mode === 'letter' }">
      <div class="header">Write a Letter</div>

      <label class="label" for="letterTo">To (email)</label>
      <input id="letterTo" v-model="letter.to" type="email" placeholder="your.friend@example.com" required />

      <label class="label" style="margin-top: 22px" for="letterSubject">Letter Title</label>
      <input id="letterSubject" v-model="letter.subject" type="text" placeholder="Today’s Letter…" required />

      <label class="label" style="margin-top: 22px" for="letterMessage">Message</label>
      <textarea id="letterMessage" v-model="letter.message" placeholder="Write your letter here…" required />

      <div class="hint">Sent from your authorized Gmail account with the PenPal paper look.</div>
      <div class="status" :class="{ error: statusLetter.isError }">{{ statusLetter.text }}</div>
      <button class="send-btn" type="button" :disabled="!accessToken || sending" @click="handleSend('letter')">
        {{ sending && mode === 'letter' ? 'Sending…' : 'Send Letter ✉️' }}
      </button>
      <div style="clear: both"></div>
    </div>

    <div id="postcardMode" class="paper" :class="{ show: mode === 'postcard' }">
      <div class="header">Write a Postcard</div>

      <label class="label" for="postcardTo">To (email)</label>
      <input id="postcardTo" v-model="postcard.to" type="email" placeholder="postcard.pal@example.com" required />

      <label class="label" style="margin-top: 16px" for="postcardSubject">Subject</label>
      <input id="postcardSubject" v-model="postcard.subject" type="text" placeholder="A postcard from the road" required />

      <label class="upload-label" for="uploadInput">Upload Photo 📷</label>
      <input id="uploadInput" type="file" accept="image/*" @change="onUpload" />

      <div class="photo-block">
        <img v-if="postcard.image" :src="postcard.image" alt="Postcard" />
        <span v-else style="color: #555">No photo selected</span>
      </div>

      <label class="label" for="postcardMessage">Message</label>
      <textarea id="postcardMessage" v-model="postcard.message" placeholder="Your postcard message…" required />

      <div class="hint">Add a photo and message—everything stays on this page.</div>
      <div class="status" :class="{ error: statusPostcard.isError }">{{ statusPostcard.text }}</div>
      <button class="send-btn" type="button" :disabled="!accessToken || sending" @click="handleSend('postcard')">
        {{ sending && mode === 'postcard' ? 'Sending…' : 'Send Postcard 📮' }}
      </button>
      <div style="clear: both"></div>
    </div>
  </div>
</template>

<style scoped>
:global([v-cloak]) {
  display: none;
}
:global(body) {
  margin: 0;
  background: var(--bg);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Arial, sans-serif;
  color: var(--ink);
}

:global(:root) {
  --orange: #ff7a3d;
  --paper: #fff9f3;
  --bg: #efe9e4;
  --ink: #2b2b2b;
  --muted: #7a716c;
  --border: #e7d7c8;
}

.center {
  max-width: 880px;
  margin: 40px auto;
  padding: 20px;
}

.auth-row {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  flex-wrap: wrap;
  margin-bottom: 12px;
}
.auth-pill {
  background: #e6d9d1;
  color: #5c524e;
  padding: 8px 12px;
  border-radius: 999px;
  font-weight: 600;
  border: 1px solid var(--border);
  font-size: 14px;
}
.auth-btn {
  padding: 10px 16px;
  border: 1px solid var(--orange);
  background: white;
  color: var(--orange);
  border-radius: 10px;
  font-size: 14px;
  cursor: pointer;
  font-weight: 700;
}

.switcher {
  text-align: center;
  margin-bottom: 24px;
}
.switch-btn {
  padding: 10px 22px;
  border: none;
  border-radius: 10px;
  font-size: 16px;
  margin: 0 6px;
  cursor: pointer;
  font-weight: 600;
  background: #e6d9d1;
  color: #5c524e;
}
.switch-btn.active {
  background: var(--orange);
  color: white;
}

.paper {
  background: var(--paper);
  padding: 32px 40px;
  border-radius: 12px;
  border: 1px solid var(--border);
  box-shadow: 0px 3px 14px rgba(0, 0, 0, 0.05);
  display: none;
}
.paper.show {
  display: block;
}

.header {
  text-align: center;
  font-size: 26px;
  font-weight: 700;
  color: var(--orange);
  margin-bottom: 26px;
}
.label {
  display: block;
  font-size: 14px;
  color: var(--muted);
  margin-bottom: 6px;
  font-weight: 600;
}

input,
textarea {
  width: 100%;
  padding: 12px;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 16px;
  outline: none;
  background: white;
}
textarea {
  resize: vertical;
  line-height: 1.6;
  min-height: 180px;
}

.send-btn {
  margin-top: 26px;
  background: var(--orange);
  color: white;
  padding: 14px 32px;
  border-radius: 12px;
  border: none;
  font-size: 17px;
  font-weight: 600;
  cursor: pointer;
  float: right;
}
.send-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.photo-block {
  width: 100%;
  height: 260px;
  border-radius: 10px;
  border: 2px solid var(--border);
  overflow: hidden;
  background: #ddd;
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 16px;
}
.photo-block img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.upload-label {
  background: var(--orange);
  color: white;
  padding: 8px 14px;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
  display: inline-block;
  margin-bottom: 16px;
  font-weight: 600;
}

.status {
  margin-top: 14px;
  color: var(--muted);
  font-size: 14px;
  min-height: 18px;
  float: left;
  padding-top: 10px;
}
.status.error {
  color: #c0392b;
}
.hint {
  color: var(--muted);
  font-size: 13px;
  margin-top: 12px;
}

@media (max-width: 640px) {
  .send-btn,
  .status {
    float: none;
    width: 100%;
    text-align: right;
  }
}
</style>
