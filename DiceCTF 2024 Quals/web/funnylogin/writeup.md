## Enum

In this challenge, we're given a website with a simple login page.

Looking at the app.js code,

```js
const express = require('express');
const crypto = require('crypto');

const app = express();

const db = require('better-sqlite3')('db.sqlite3');
db.exec(`DROP TABLE IF EXISTS users;`);
db.exec(`CREATE TABLE users(
    id INTEGER PRIMARY KEY,
    username TEXT,
    password TEXT
);`);

const FLAG = process.env.FLAG || "dice{test_flag}";
const PORT = process.env.PORT || 3000;

const users = [...Array(100_000)].map(() => ({ user: `user-${crypto.randomUUID()}`, pass: crypto.randomBytes(8).toString("hex") }));
db.exec(`INSERT INTO users (id, username, password) VALUES ${users.map((u,i) => `(${i}, '${u.user}', '${u.pass}')`).join(", ")}`);

const isAdmin = {};
const newAdmin = users[Math.floor(Math.random() * users.length)];
isAdmin[newAdmin.user] = true;

app.use(express.urlencoded({ extended: false }));
app.use(express.static("public"));

app.post("/api/login", (req, res) => {
    const { user, pass } = req.body;

    const query = `SELECT id FROM users WHERE username = '${user}' AND password = '${pass}';`;
    try {
        const id = db.prepare(query).get()?.id;
        if (!id) {
            return res.redirect("/?message=Incorrect username or password");
        }

        if (users[id] && isAdmin[user]) {
            return res.redirect("/?flag=" + encodeURIComponent(FLAG));
        }
        return res.redirect("/?message=This system is currently only available to admins...");
    }
    catch {
        return res.redirect("/?message=Nice try...");
    }
});

app.listen(PORT, () => console.log(`web/funnylogin listening on port ${PORT}`));
```

It first generates a `users` object with below code.

```js
const users = [...Array(100_000)].map(() => ({ user: `user-${crypto.randomUUID()}`, pass: crypto.randomBytes(8).toString("hex") }));
db.exec(`INSERT INTO users (id, username, password) VALUES ${users.map((u,i) => `(${i}, '${u.user}', '${u.pass}')`).join(", ")}`);
```

It would randomly create different users with random usernames and insert it into the sqlite database.

Next it is making an `isAdmin` object and randomly select one of the users as admin.

```js
const isAdmin = {};
const newAdmin = users[Math.floor(Math.random() * users.length)];
isAdmin[newAdmin.user] = true;
```

## Vulnerability

A simple SQLi.
```js
const query = `SELECT id FROM users WHERE username = '${user}' AND password = '${pass}';`;
```

Moving forward, we see a condition as follows.

```js
if (users[id] && isAdmin[user]) {
            return res.redirect("/?flag=" + encodeURIComponent(FLAG));
        }
```

If we get a valid user i.e. `users[id]` and return something from `isAdmin[user]`, we'll pass this check.

To solve the challenge, we can use the following payload for user and pass params.

```
user=__proto__&pass='+OR+id+=+1;+--+-
```

`__proto__` would return `[Object: null prototype] {}` that would pass the check.
`'+OR+id+=+1;+--+-` would return `id = 1` to get a valid user.

Flag: `dice{test_flag}`