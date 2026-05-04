# 🤖 wp_chatbot.01

A Node.js WhatsApp bot built using Baileys to automate SMM panel operations like **cancel, speed, partial, refill, and fake completed handling**.

---

## 🚀 Features

* Bulk order processing
* Automatic status checking via API
* Smart routing to provider groups
* Customer feedback responses
* Fake completed handling logic

---

## 🛠️ Tech Stack

* Node.js
* Baileys
* Axios

---

## ⚙️ Setup

```bash
git clone https://github.com/your-username/whatsapp-smm-bot.git
cd whatsapp-smm-bot
npm install
node index.js
```

Update API config in code:

```js
const SMM_API_KEY = 'YOUR_API_KEY';
const BASE_URL = 'YOUR_BASE_URL';
```

---

## 📌 Usage

Send message:

```
123456 cancel
123456 speed
123456 refill
```

Bot will:

* Fetch order status
* Send request to provider
* Reply with result

---

## 📁 Structure

```bash
.
├── index.js
├── auth_info/
└── README.md
```

---

## 👨‍💻 Author

Madhav Ahuja

