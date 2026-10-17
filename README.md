<div align="center">

# xyrexs-baileys

Dokumentasi penggunaan library WhatsApp Web API berbasis TypeScript untuk Node.js.

</div>

## Daftar Isi

- [Tentang](#tentang)
- [Persyaratan](#persyaratan)
- [Instalasi](#instalasi)
- [Membuat koneksi dan pairing](#membuat-koneksi-dan-pairing)
- [Menerima pesan](#menerima-pesan)
- [Mengirim pesan](#mengirim-pesan)
- [Media](#media)
- [Reply dan quote](#reply-dan-quote)
- [Pesan interaktif dan tombol](#pesan-interaktif-dan-tombol)
- [Album media](#album-media)
- [AI message icon](#ai-message-icon)
- [Poll, reaction, edit, dan hapus](#poll-reaction-edit-dan-hapus)
- [Lokasi dan kontak](#lokasi-dan-kontak)
- [Presence dan tanda baca](#presence-dan-tanda-baca)
- [Grup](#grup)
- [Profil dan blokir](#profil-dan-blokir)
- [WhatsApp Channel / Newsletter](#whatsapp-channel--newsletter)
- [Utility yang sering dipakai](#utility-yang-sering-dipakai)
- [Deploy dan keamanan](#deploy-dan-keamanan)
- [Troubleshooting](#troubleshooting)

## Tentang

`xyrexs-baileys` adalah library tidak resmi untuk berinteraksi dengan WhatsApp Web multi-device. Library ini bukan WhatsApp Business Platform resmi. API dan format pesan dapat berubah mengikuti perubahan protokol WhatsApp.

Contoh pada dokumen ini menggunakan ESM dan JavaScript. Untuk TypeScript, tambahkan tipe sesuai kebutuhan proyek. Fitur tertentu seperti tombol legacy atau pesan interaktif dapat berbeda dukungannya menurut versi aplikasi WhatsApp penerima.

## Persyaratan

- Node.js sesuai batas versi pada `engine-requirements.js` dan `package.json`.
- npm.
- Akun WhatsApp yang dapat ditautkan sebagai perangkat pendamping.
- Koneksi internet yang stabil.

## Instalasi

Jika package sudah diterbitkan ke npm:

```bash
npm install xyrexs-baileys
```

Jika menggunakan source repository, masuk ke folder proyek lalu jalankan:

```bash
npm install
npm run build:tsc
```

Contoh import ESM dari package:

```js
import makeWASocket, {
  DisconnectReason,
  fetchLatestBaileysVersion,
  useMultiFileAuthState
} from 'xyrexs-baileys'
import { Boom } from '@hapi/boom'
import pino from 'pino'
```

## Membuat koneksi dan pairing

Simpan sebagai `index.js`. Saat pertama kali dijalankan, kode ini menampilkan QR di terminal. Session disimpan di folder `xyrexs_auth` agar tidak perlu pairing ulang setiap restart.

```js
import makeWASocket, {
  DisconnectReason,
  fetchLatestBaileysVersion,
  useMultiFileAuthState
} from 'xyrexs-baileys'
import { Boom } from '@hapi/boom'
import pino from 'pino'

const start = async () => {
  const { state, saveCreds } = await useMultiFileAuthState('./xyrexs_auth')
  const { version } = await fetchLatestBaileysVersion()
  const sock = makeWASocket({
    version,
    auth: state,
    logger: pino({ level: 'silent' }),
    printQRInTerminal: true
  })

  sock.ev.on('creds.update', saveCreds)

  sock.ev.on('connection.update', ({ connection, lastDisconnect }) => {
    if (connection === 'open') console.log('WhatsApp terhubung')
    if (connection === 'close') {
      const code = lastDisconnect?.error instanceof Boom
        ? lastDisconnect.error.output?.statusCode
        : undefined
      if (code !== DisconnectReason.loggedOut) start()
      else console.log('Session logout. Hapus session hanya jika ingin pairing ulang.')
    }
  })

  return sock
}

start().catch(console.error)
```

### Pairing menggunakan kode

Gunakan kode pairing bila ingin menautkan perangkat tanpa QR. Nomor harus memakai kode negara dan hanya angka, misalnya `6281234567890`.

```js
const { state, saveCreds } = await useMultiFileAuthState('./xyrexs_auth')
const sock = makeWASocket({ auth: state, logger: pino({ level: 'silent' }) })
sock.ev.on('creds.update', saveCreds)

if (!sock.authState.creds.registered) {
  const code = await sock.requestPairingCode('6281234567890')
  console.log('Masukkan kode ini di WhatsApp:', code)
}
```

Fork ini juga menerima kode pairing khusus melalui argumen kedua. Gunakan hanya jika metode tersebut masih didukung oleh versi protokol yang sedang dipakai.

```js
const code = await sock.requestPairingCode('6281234567890', 'XYREX123')
console.log(code)
```

Jangan membagikan folder session atau kredensial kepada orang lain.

## Menerima pesan

Pesan baru diterima melalui event `messages.upsert`. `remoteJid` adalah JID chat. Untuk teks biasa, isi dapat berada di `conversation` atau `extendedTextMessage.text`.

```js
sock.ev.on('messages.upsert', async ({ messages, type }) => {
  if (type !== 'notify') return

  for (const msg of messages) {
    if (!msg.message || msg.key.fromMe) continue

    const jid = msg.key.remoteJid
    const text = msg.message.conversation
      ?? msg.message.extendedTextMessage?.text

    if (!jid || !text) continue
    console.log({ jid, text })

    if (text === '!ping') {
      await sock.sendMessage(jid, { text: 'Pong!' }, { quoted: msg })
    }
  }
})
```

Untuk pesan dengan wrapper seperti ephemeral atau view-once, gunakan helper `normalizeMessageContent` yang diekspor library, lalu periksa isi pesan hasil normalisasi. Tidak semua pesan yang masuk berisi teks; media, reaction, poll update, dan pesan sistem memiliki struktur berbeda.

## Mengirim pesan

Ganti nomor tujuan dengan nomor penerima dalam format internasional tanpa tanda `+`, lalu tambahkan `@s.whatsapp.net`.

### Teks

```js
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'Halo dari xyrexs-baileys'
})
```

### Teks dengan mention

```js
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'Halo @6281234567890',
  mentions: ['6281234567890@s.whatsapp.net']
})
```

### Link dengan preview

```js
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'Dokumentasi: https://github.com/nstar-y/bail'
}, { linkPreview: undefined })
```

Preview link bergantung pada kemampuan fetch preview dan perilaku WhatsApp. Hindari mengirim URL yang tidak tepercaya.

### Pesan dengan formatting

WhatsApp mendukung pola formatting teks seperti `*tebal*`, `_miring_`, `~coret~`, dan `` ```monospace``` `` pada klien yang mendukungnya.

## Media

Media dapat diberikan sebagai URL melalui `{ url }`, `Buffer`, atau path lokal. Pastikan URL dapat diakses server dan file berasal dari sumber tepercaya.

### Gambar

```js
await sock.sendMessage(jid, {
  image: { url: 'https://example.com/image.jpg' },
  caption: 'Keterangan gambar'
})
```

### Video

```js
await sock.sendMessage(jid, {
  video: { url: 'https://example.com/video.mp4' },
  caption: 'Keterangan video'
})
```

### Audio

```js
await sock.sendMessage(jid, {
  audio: { url: 'https://example.com/audio.mp3' },
  mimetype: 'audio/mpeg'
})
```

Untuk mengirim audio sebagai voice note, gunakan `ptt: true` jika format dan versi klien mendukungnya.

```js
await sock.sendMessage(jid, {
  audio: { url: './voice-note.ogg' },
  mimetype: 'audio/ogg; codecs=opus',
  ptt: true
})
```

### Dokumen

```js
await sock.sendMessage(jid, {
  document: { url: './report.pdf' },
  mimetype: 'application/pdf',
  fileName: 'report.pdf',
  caption: 'Dokumen terlampir'
})
```

### Stiker

```js
await sock.sendMessage(jid, {
  sticker: { url: './sticker.webp' }
})
```

Stiker harus berupa format yang didukung WhatsApp, biasanya WebP. Library tidak otomatis menjamin semua file gambar dapat langsung dikirim sebagai stiker.

## Reply dan quote

Gunakan opsi `quoted` untuk membalas pesan tertentu.

```js
await sock.sendMessage(jid, {
  text: 'Ini balasannya'
}, { quoted: msg })
```

`msg` adalah objek pesan yang sebelumnya diterima melalui event. Untuk pesan yang tidak memiliki objek pesan asal, kirim tanpa opsi `quoted`.

## Pesan interaktif dan tombol

### Tombol legacy

```js
await sock.sendMessage(jid, {
  text: 'Pilih menu',
  footer: 'xyrexs-baileys',
  buttons: [
    { buttonId: 'menu_profile', buttonText: { displayText: 'Profil' }, type: 1 },
    { buttonId: 'menu_help', buttonText: { displayText: 'Bantuan' }, type: 1 }
  ],
  headerType: 1
})
```

Balasan tombol biasanya diterima sebagai `buttonsResponseMessage` atau tipe interaktif lain bergantung pada klien WhatsApp. Karena tombol legacy dapat dibatasi atau tidak ditampilkan pada sebagian versi, uji dengan perangkat target.

### Tombol interaktif

```js
await sock.sendMessage(jid, {
  text: 'Pilih tindakan',
  title: 'Menu Xyrexs',
  footer: 'Silakan pilih',
  interactiveButtons: [
    {
      name: 'quick_reply',
      buttonParamsJson: JSON.stringify({
        display_text: 'Lihat profil',
        id: 'profile'
      })
    },
    {
      name: 'cta_url',
      buttonParamsJson: JSON.stringify({
        display_text: 'Buka situs',
        url: 'https://example.com'
      })
    },
    {
      name: 'cta_copy',
      buttonParamsJson: JSON.stringify({
        display_text: 'Salin kode',
        id: 'copy-code',
        copy_code: 'XYREXS-123'
      })
    }
  ]
})
```

### Tombol dengan gambar atau video

```js
await sock.sendMessage(jid, {
  image: { url: './menu.jpg' },
  caption: 'Pilih menu di bawah',
  title: 'Menu',
  footer: 'xyrexs-baileys',
  interactiveButtons: [
    {
      name: 'quick_reply',
      buttonParamsJson: JSON.stringify({ display_text: 'Mulai', id: 'start' })
    }
  ]
})
```

### Daftar pilihan

```js
await sock.sendMessage(jid, {
  text: 'Pilih salah satu item',
  title: 'Daftar menu',
  footer: 'xyrexs-baileys',
  interactiveButtons: [
    {
      name: 'single_select',
      buttonParamsJson: JSON.stringify({
        title: 'Buka daftar',
        sections: [
          {
            title: 'Menu utama',
            rows: [
              { header: 'Info', title: 'Profil', description: 'Lihat profil bot', id: 'profile' },
              { header: 'Bantuan', title: 'Panduan', description: 'Baca bantuan', id: 'help' }
            ]
          }
        ]
      })
    }
  ]
})
```

## Album media

Fitur album pada fork ini menerima array media melalui properti `album`.

```js
await sock.sendMessage(jid, {
  album: [
    { image: { url: './image-1.jpg' } },
    { image: { url: './image-2.jpg' } },
    { video: { url: './clip.mp4' } }
  ],
  caption: 'Album dari xyrexs-baileys'
})
```

Periksa batas ukuran dan jumlah item yang berlaku di WhatsApp. Hasil tampilan album dapat berbeda menurut klien penerima.

## AI message icon

Fork ini menambahkan properti `ai: true` untuk meminta ikon AI pada pesan yang mendukung fitur tersebut.

```js
await sock.sendMessage(jid, {
  text: 'Jawaban dibuat oleh AI',
  ai: true
})
```

Properti ini adalah metadata tampilan pesan, bukan integrasi dengan model AI. Dukungan tampilannya bergantung pada versi WhatsApp.

## Poll, reaction, edit, dan hapus

### Poll

```js
await sock.sendMessage(jid, {
  poll: {
    name: 'Pilih waktu rapat',
    values: ['Pagi', 'Siang', 'Malam'],
    selectableCount: 1
  }
})
```

### Reaction pada pesan

```js
await sock.sendMessage(jid, {
  react: {
    text: '👍',
    key: msg.key
  }
})
```

Untuk menghapus reaction sendiri, kirim teks reaction kosong pada key pesan yang sama.

```js
await sock.sendMessage(jid, {
  react: {
    text: '',
    key: msg.key
  }
})
```

### Edit pesan yang dikirim sendiri

```js
await sock.sendMessage(jid, {
  text: 'Teks yang sudah diperbarui',
  edit: sentMessage.key
})
```

`sentMessage` adalah hasil dari `sendMessage` sebelumnya. Kemampuan edit dibatasi oleh aturan WhatsApp.

### Hapus pesan

```js
await sock.sendMessage(jid, {
  delete: msg.key
})
```

Menghapus untuk semua orang hanya berlaku jika pesan, pengirim, dan waktu memenuhi ketentuan WhatsApp. Jangan menganggap operasi selalu berhasil.

## Lokasi dan kontak

### Lokasi

```js
await sock.sendMessage(jid, {
  location: {
    degreesLatitude: -6.200000,
    degreesLongitude: 106.816666,
    name: 'Jakarta',
    address: 'Indonesia'
  }
})
```

### Kontak

```js
await sock.sendMessage(jid, {
  contacts: {
    displayName: 'Kontak Contoh',
    contacts: [
      {
        vcard: 'BEGIN:VCARD\nVERSION:3.0\nFN:Kontak Contoh\nTEL;type=CELL;type=VOICE;waid=6281234567890:+6281234567890\nEND:VCARD'
      }
    ]
  }
})
```

## Presence dan tanda baca

### Menampilkan status mengetik

```js
await sock.sendPresenceUpdate('composing', jid)
await sock.sendPresenceUpdate('paused', jid)
```

Nilai umum meliputi `available`, `unavailable`, `composing`, `recording`, dan `paused`. Presence tidak menjamin penerima akan selalu melihat indikator tersebut.

### Membaca pesan

```js
await sock.readMessages([msg.key])
```

### Berlangganan presence kontak

```js
await sock.presenceSubscribe(jid)
```

## Grup

JID grup biasanya berakhiran `@g.us`. Operasi admin hanya berhasil jika akun memiliki izin yang diperlukan.

### Membuat grup

```js
const group = await sock.groupCreate('Grup Xyrexs', [
  '6281234567890@s.whatsapp.net',
  '6289876543210@s.whatsapp.net'
])
console.log(group)
```

### Mengubah subjek grup

```js
await sock.groupUpdateSubject(group.id, 'Nama Grup Baru')
```

### Menambah atau mengeluarkan anggota

```js
await sock.groupParticipantsUpdate(
  group.id,
  ['6281234567890@s.whatsapp.net'],
  'add'
)
```

Aksi yang umum tersedia meliputi `add`, `remove`, `promote`, dan `demote`, bergantung pada izin serta dukungan WhatsApp.

### Mengambil metadata grup

```js
const metadata = await sock.groupMetadata('120000000000000000@g.us')
console.log(metadata.subject, metadata.participants)
```

## Profil dan blokir

### Mengubah nama profil

```js
await sock.updateProfileName('Nama Bot')
```

### Mengubah status profil

```js
await sock.updateProfileStatus('Bot aktif')
```

### Mengambil URL foto profil

```js
const url = await sock.profilePictureUrl(jid, 'image').catch(() => null)
console.log(url)
```

### Memblokir atau membuka blokir

```js
await sock.updateBlockStatus(jid, 'block')
await sock.updateBlockStatus(jid, 'unblock')
```

## WhatsApp Channel / Newsletter

Channel menggunakan JID yang biasanya berakhiran `@newsletter`.

### Membaca metadata channel

```js
const metadata = await sock.newsletterMetadata('jid', '123456789@newsletter')
console.log(metadata)
```

### Mengambil metadata menggunakan invite code

```js
const metadata = await sock.newsletterMetadata('invite', 'KODE_INVITE')
console.log(metadata)
```

### Mengikuti dan berhenti mengikuti

```js
await sock.newsletterFollow('123456789@newsletter')
await sock.newsletterUnfollow('123456789@newsletter')
```

### Membuat channel

```js
const channel = await sock.newsletterCreate('Channel Xyrexs', 'Deskripsi channel')
console.log(channel)
```

### Memperbarui nama dan deskripsi

```js
await sock.newsletterUpdateName('123456789@newsletter', 'Nama Baru')
await sock.newsletterUpdateDescription('123456789@newsletter', 'Deskripsi baru')
```

### Memperbarui atau menghapus foto channel

```js
await sock.newsletterUpdatePicture('123456789@newsletter', { url: './channel.jpg' })
await sock.newsletterRemovePicture('123456789@newsletter')
```

### Mute dan unmute channel

```js
await sock.newsletterMute('123456789@newsletter')
await sock.newsletterUnmute('123456789@newsletter')
```

### Reaction pada pesan channel

```js
await sock.newsletterReactMessage('123456789@newsletter', '175', '👍')
```

Argumen kedua adalah server message ID, bukan JID atau ID pesan chat biasa. Akses dan dukungan tiap operasi dapat berubah mengikuti protokol WhatsApp.

## Utility yang sering dipakai

Library mengekspor beberapa utilitas Baileys melalui entry point utama, termasuk `delay`, `jidNormalizedUser`, `downloadMediaMessage`, `normalizeMessageContent`, `fetchLatestBaileysVersion`, `useMultiFileAuthState`, dan helper store. Import hanya helper yang memang tersedia pada versi package yang digunakan.

```js
import {
  delay,
  jidNormalizedUser,
  normalizeMessageContent
} from 'xyrexs-baileys'

await delay(1000)
const normalizedJid = jidNormalizedUser('6281234567890@s.whatsapp.net')
console.log(normalizedJid)
```

Untuk mengunduh media dari pesan masuk, gunakan helper yang diekspor versi ini dan cek tipe pesan terlebih dahulu. Hindari mengunduh file berukuran besar tanpa batas ukuran atau validasi sumber.

## Deploy dan keamanan

- Jangan commit folder autentikasi/session ke Git.
- Jangan mengirim isi file kredensial ke log atau chat.
- Batasi ukuran file dan timeout saat menerima atau mengunduh media.
- Terapkan rate limit pada command bot dan validasi input pengguna.
- Tangani error setiap operasi jaringan menggunakan `try/catch`.
- Jangan melakukan spam, pengiriman massal tanpa izin, atau tindakan yang melanggar ketentuan WhatsApp.
- Gunakan `logger: pino({ level: 'silent' })` hanya jika memang tidak membutuhkan log; saat debugging, pilih level yang sesuai.
- Simpan state di penyimpanan persisten pada server produksi dan siapkan strategi backup yang aman.
- Uji fitur di akun dan perangkat yang Anda kontrol sebelum deploy.

## Troubleshooting

### QR atau pairing tidak muncul

Pastikan terminal interaktif tersedia, akun belum tertaut pada batas perangkat, nomor memakai kode negara, dan tidak ada error jaringan. Untuk pairing code, panggil `requestPairingCode` hanya saat `sock.authState.creds.registered` bernilai `false`.

### Bot terus reconnect

Periksa `lastDisconnect`, status code, jaringan, waktu server, dan apakah session logout. Jangan melakukan reconnect berulang tanpa jeda; gunakan backoff untuk lingkungan produksi.

### Pesan tidak terkirim

Pastikan JID tujuan benar, socket sudah terhubung, akun tidak dibatasi, dan format media sesuai. Tangkap error dari `sendMessage` dan uji terlebih dahulu dengan pesan teks sederhana.

### Tombol tidak tampil

Tombol legacy maupun interaktif tidak didukung seragam di semua versi WhatsApp. Uji format lain atau kirim pesan teks biasa sebagai fallback.

### Build gagal

Jalankan `npm install`, lalu `npm run build:tsc`. Periksa versi Node.js dan pesan error pertama yang muncul. Error tipe atau dependensi yang hilang harus diselesaikan sebelum menggunakan build untuk produksi.

## Lisensi dan catatan

Periksa file `LICENSE` untuk ketentuan lisensi yang berlaku pada source ini. `xyrexs-baileys` adalah proyek tidak resmi dan tidak berafiliasi dengan WhatsApp atau Meta. API internal dapat berubah tanpa pemberitahuan seiring perubahan protokol.
