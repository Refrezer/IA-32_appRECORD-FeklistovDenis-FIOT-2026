<div style="text-align: center;">
  <strong>НАЦІОНАЛЬНИЙ ТЕХНІЧНИЙ УНІВЕРСИТЕТ УКРАЇНИ</strong><br>
  <strong>«КИЇВСЬКИЙ ПОЛІТЕХНІЧНИЙ ІНСТИТУТ імені ІГОРЯ СІКОРСЬКОГО»</strong><br>
  Факультет інформатики та обчислювальної техніки<br>
  Кафедра інформаційних систем та технологій<br>
  <br><br>
  <strong>Лабораторна робота № 3</strong><br>
  з дисципліни «WEB-орієнтовані технології. Backend розробки»<br>
  на тему: <strong>«РОЗРОБКА ФУНКЦІОНАЛЬНОГО REST API. РЕЄСТРАЦІЯ ТА АВТОРИЗАЦІЯ КОРИСТУВАЧІВ. ВАЛІДАЦІЯ ДАНИХ І ОБРОБКА ПОМИЛОК»</strong><br>
  <br><br>
</div>

<div style="text-align: right;">
  <strong>Виконав:</strong> студент 3-го курсу<br>
  групи ІА-32<br>
  <strong>Феклістов Денис</strong><br>
  <br><br>
</div>

<div style="text-align: center;">
  Київ – 2026
</div>

---

## МЕТА РОБОТИ

1. Вивчення принципів побудови REST API.
2. Набуття практичних навичок розробки серверного застосунку з використанням платформи Node.js і фреймворку Express.
3. Реалізувати механізми реєстрації та авторизації користувачів.
4. Забезпечити валідацію вхідних даних та обробку помилок.
5. Організувати захищений доступ до ресурсів із використанням JWT-токенів і системи ролей користувачів.

---

## 1. ХІД ВИКОНАННЯ РОБОТИ (ПРОГРАМНА РЕАЛІЗАЦІЯ)

Під час виконання лабораторної роботи було розширено існуючий застосунок. Для забезпечення безпеки встановлено пакети `bcryptjs` (для хешування паролів), `jsonwebtoken` (для JWT-авторизації), `express-rate-limit` (для захисту від брутфорс-атак) та `google-auth-library` (для OAuth).

### 1.1. Оновлена модель користувача (`models/User.js`)
У модель було додано поля для хешу пароля, ролі користувача, refresh-токена, а також дані для скидання пароля та підтвердження email.

```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
  name: { type: DataTypes.STRING, allowNull: false },
  email: { type: DataTypes.STRING, allowNull: false, unique: true },
  password: { type: DataTypes.STRING, allowNull: false },
  role: { type: DataTypes.STRING, defaultValue: 'user' },
  refreshToken: { type: DataTypes.TEXT, allowNull: true },
  isVerified: { type: DataTypes.BOOLEAN, defaultValue: false },
  emailToken: { type: DataTypes.STRING, allowNull: true },
  resetToken: { type: DataTypes.STRING, allowNull: true },
  resetTokenExpiry: { type: DataTypes.DATE, allowNull: true }
});

module.exports = User;

```

### 1.2. Головний файл сервера (`server.js`)

Було реалізовано реєстрацію з підтвердженням email, логін із генерацією Access та Refresh токенів, CRUD операції для профілю та постів, скидання пароля та інтеграцію Google OAuth.

```javascript
const express = require('express');
const sequelize = require('./config/database');
const User = require('./models/User');
const Post = require('./models/Post');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const rateLimit = require('express-rate-limit');
const { OAuth2Client } = require('google-auth-library');

const SECRET_KEY = "super_secret_kpi_key";
const REFRESH_SECRET_KEY = "super_refresh_kpi_key";
const GOOGLE_CLIENT_ID = "123456789-testkpi.apps.googleusercontent.com";

const app = express();
const PORT = 3000;
const googleClient = new OAuth2Client(GOOGLE_CLIENT_ID);

app.use(express.json());

User.hasMany(Post, { foreignKey: 'userId', onDelete: 'CASCADE' });
Post.belongsTo(User, { foreignKey: 'userId' });

// ================== MIDDLEWARE ==================
const authenticateToken = (req, res, next) => {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1]; 
    if (!token) return res.status(401).json({ message: "Доступ заборонено" });

    jwt.verify(token, SECRET_KEY, (err, decoded) => {
        if (err) return res.status(403).json({ message: "Недійсний токен" });
        req.user = decoded; 
        next();
    });
};

const loginLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 5, message: "Забагато спроб" });

// ================== АВТЕНТИФІКАЦІЯ ==================
app.post('/register', async (req, res, next) => {
    try {
        const { name, email, password, confirmPassword } = req.body;
        if (!name || !email || !password || !confirmPassword) return res.status(400).json({ message: "Всі поля обов'язкові" });
        if (password.length < 6) return res.status(400).json({ message: "Мінімум 6 символів" });
        if (password !== confirmPassword) return res.status(400).json({ message: "Паролі не співпадають" });
        
        const userExists = await User.findOne({ where: { email } });
        if (userExists) return res.status(400).json({ message: "Користувач вже існує" });

        const hashedPassword = await bcrypt.hash(password, 10);
        const emailToken = crypto.randomBytes(32).toString('hex');

        await User.create({ name, email, password: hashedPassword, emailToken, isVerified: false });
        console.log(`Підтвердження: http://localhost:3000/verify-email?token=${emailToken}`);
        res.status(201).json({ message: "Реєстрація успішна! Перевірте консоль." });
    } catch (error) { next(error); }
});

app.post('/login', loginLimiter, async (req, res, next) => {
    try {
        const { email, password } = req.body;
        const user = await User.findOne({ where: { email } });
        if (!user) return res.status(400).json({ message: "Користувача не знайдено" });
        if (!user.isVerified) return res.status(401).json({ message: "Підтвердіть email" });

        const isMatch = await bcrypt.compare(password, user.password);
        if (!isMatch) return res.status(400).json({ message: "Невірний пароль" });

        const accessToken = jwt.sign({ id: user.id, email: user.email, role: user.role }, SECRET_KEY, { expiresIn: "15m" });
        const refreshToken = jwt.sign({ id: user.id }, REFRESH_SECRET_KEY, { expiresIn: "7d" });

        await User.update({ refreshToken }, { where: { id: user.id } });
        res.json({ message: "Успішний вхід", accessToken, refreshToken });
    } catch (error) { next(error); }
});

app.post('/refresh', async (req, res, next) => {
    try {
        const { refreshToken } = req.body;
        if (!refreshToken) return res.status(401).json({ message: "Необхідний Refresh Token" });

        const user = await User.findOne({ where: { refreshToken } });
        if (!user) return res.status(403).json({ message: "Недійсний токен" });

        jwt.verify(refreshToken, REFRESH_SECRET_KEY, (err) => {
            if (err) return res.status(403).json({ message: "Токен прострочений" });
            const accessToken = jwt.sign({ id: user.id, email: user.email, role: user.role }, SECRET_KEY, { expiresIn: "15m" });
            res.json({ accessToken });
        });
    } catch (error) { next(error); }
});

// ================== ВІДНОВЛЕННЯ ПАРОЛЯ ТА OAUTH ==================
app.get('/verify-email', async (req, res, next) => {
    try {
        const { token } = req.query;
        const user = await User.findOne({ where: { emailToken: token } });
        if (!user) return res.status(400).json({ message: "Недійсний токен" });
        await User.update({ isVerified: true, emailToken: null }, { where: { id: user.id } });
        res.send('<h1>Email підтверджено!</h1>');
    } catch (error) { next(error); }
});

app.post('/forgot-password', async (req, res, next) => {
    try {
        const { email } = req.body;
        const user = await User.findOne({ where: { email } });
        if (!user) return res.status(404).json({ message: "Користувача не знайдено" });

        const resetToken = crypto.randomBytes(32).toString('hex');
        const resetTokenExpiry = Date.now() + 3600000;
        await User.update({ resetToken, resetTokenExpiry }, { where: { id: user.id } });
        console.log(`Токен скидання: ${resetToken}`);
        res.json({ message: "Токен надіслано в консоль." });
    } catch (error) { next(error); }
});

app.post('/reset-password', async (req, res, next) => {
    try {
        const { token, newPassword } = req.body;
        const user = await User.findOne({ where: { resetToken: token, resetTokenExpiry: { [sequelize.Sequelize.Op.gt]: Date.now() } }});
        if (!user) return res.status(400).json({ message: "Токен недійсний" });

        const hashedPassword = await bcrypt.hash(newPassword, 10);
        await User.update({ password: hashedPassword, resetToken: null, resetTokenExpiry: null }, { where: { id: user.id } });
        res.json({ message: "Пароль змінено" });
    } catch (error) { next(error); }
});

app.post('/google-login', async (req, res, next) => {
    try {
        const { googleToken } = req.body;
        const ticket = await googleClient.verifyIdToken({ idToken: googleToken, audience: GOOGLE_CLIENT_ID });
        const { email, name } = ticket.getPayload();

        let user = await User.findOne({ where: { email } });
        if (!user) {
            const hashedPassword = await bcrypt.hash(crypto.randomBytes(16).toString('hex'), 10);
            user = await User.create({ name, email, password: hashedPassword, isVerified: true });
        }

        const accessToken = jwt.sign({ id: user.id, email: user.email, role: user.role }, SECRET_KEY, { expiresIn: "15m" });
        res.json({ accessToken });
    } catch (error) { res.status(401).json({ message: "Недійсний токен Google" }); }
});

// ================== ГЛОБАЛЬНА ОБРОБКА ПОМИЛОК ==================
app.use((err, req, res, next) => {
    console.error(`[ERROR] ${req.method} ${req.originalUrl} >> ${err.message}`);
    res.status(500).json({ message: "Внутрішня помилка сервера." });
});

sequelize.sync({ alter: true }).then(() => {
    app.listen(PORT, () => console.log(`🚀 Сервер працює: http://localhost:${PORT}`));
});

```
### 1.3. Результати тестування API (Скріншоти виконання)

#### 1. Успішна реєстрація нового користувача
![Рисунок 3.1 — Успішна реєстрація користувача (POST /register)](/assets/labs/lab-3/1.png)

#### 2. Валідація: спроба реєстрації з порожніми обов'язковими полями
![Рисунок 3.2 — Помилка валідації порожніх полів (400 Bad Request)](/assets/labs/lab-3/2.png)

#### 3. Валідація: незбіг пароля та поля підтвердження
![Рисунок 3.3 — Помилка валідації незбігу паролів (400 Bad Request)](/assets/labs/lab-3/3.png)

#### 4. Валідація: спроба реєстрації з email, який вже існує в системі
![Рисунок 3.4 — Помилка дублювання унікального email (400 Bad Request)](/assets/labs/lab-3/4.png)

#### 5. Симуляція відправки email: вивід токена активації в консоль сервера
![Рисунок 3.5 — Вивід унікального посилання активації в терміналі VS Code](/assets/labs/lab-3/5.png)

#### 6. Успішна верифікація та активація акаунту через посилання у браузері
![Рисунок 3.6 — Сторінка успішного підтвердження email (GET /verify-email)](/assets/labs/lab-3/6.png)

#### 7. Спроба авторизації (входу) користувача з непідтвердженою поштою
![Рисунок 3.7 — Блокування входу до моменту верифікації (401 Unauthorized)](/assets/labs/lab-3/7.png)

#### 8. Успішний вхід в систему та генерація пари токенів (Access та Refresh)
![Рисунок 3.8 — Отримання токенів при успішному логіні (POST /login)](/assets/labs/lab-3/8.png)

#### 9. Робота системи безпеки Rate Limiter (Блокування брутфорс-атак)
![Рисунок 3.9 — Перевищення ліміту спроб входу (429 Too Many Requests)](/assets/labs/lab-3/9.png)

#### 10. Механізм Refresh Token: успішне оновлення Access токена
![Рисунок 3.10 — Генерація нового короткострокового токена (POST /refresh)](/assets/labs/lab-3/10.png)

#### 11. Захищений маршрут профілю: успішне отримання даних за Bearer токеном
![Рисунок 3.11 — Отримання даних авторизованого профілю (GET /profile)](/assets/labs/lab-3/11.png)

#### 12. Редагування профілю: успішна зміна імені та email користувача
![Рисунок 3.12 — Оновлення даних користувача у захищеному маршруті (PUT /profile)](/assets/labs/lab-3/12.png)

#### 13. Безпечна зміна пароля користувача з перевіркою старого хешу
![Рисунок 3.13 — Успішна зміна пароля (PUT /profile/password)](/assets/labs/lab-3/13.png)

#### 14. Ініціація відновлення пароля: генерація та вивід токена скидання в консоль
![Рисунок 3.14 — Запит на скидання пароля та отримання інструкцій (POST /forgot-password)](/assets/labs/lab-3/14.png)

#### 15. Встановлення нового пароля за допомогою перевірки токена скидання
![Рисунок 3.15 — Успішне встановлення нового пароля за токеном (POST /reset-password)](/assets/labs/lab-3/15.png)
---

## 2. ВІДПОВІДІ НА КОНТРОЛЬНІ ПИТАННЯ

**1. Що таке REST API та які його основні принципи?** REST API — це архітектурний стиль взаємодії клієнта і сервера через HTTP-протокол. Основні принципи: клієнт-серверна архітектура, відсутність збереження стану (Stateless), використання стандартних HTTP методів, кешованість відповідей та багаторівнева система.

**2. Які HTTP-методи використовуються в REST API і для чого вони призначені?** - **GET:** отримання даних (читання).

* **POST:** створення нового ресурсу (наприклад, реєстрація користувача).
* **PUT / PATCH:** повне або часткове оновлення існуючого ресурсу.
* **DELETE:** видалення ресурсу.

**3. Що таке клієнт-серверна архітектура?** Це принцип побудови системи, де обов'язки чітко розділені: клієнтська частина відповідає виключно за відображення інтерфейсу (UI) та взаємодію з користувачем, а серверна — за бізнес-логіку, безпеку та зберігання даних у БД.

**4. Що означає принцип Stateless у REST?** Без стану (Stateless) означає, що сервер не зберігає інформацію про попередні запити клієнта. Кожен HTTP-запит є незалежним і повинен містити всю необхідну інформацію (наприклад, токен авторизації) для його успішної обробки.

**5. Для чого використовується фреймворк Express у Node.js?** Express використовується для швидкого та зручного створення веб-серверів. Він надає потужні інструменти для маршрутизації HTTP-запитів, налаштування middleware та структурування REST API.

**6. Що таке реєстрація користувача в інформаційній системі?** Це процес створення нового облікового запису. Він включає введення користувачем даних (email, пароль), валідацію цих даних сервером, безпечне хешування пароля та збереження запису в базу даних.

**7. Що таке авторизація та чим вона відрізняється від автентифікації?** Автентифікація відповідає на питання "Хто ти?" (перевірка логіна та пароля). Авторизація відповідає на питання "Що тобі дозволено робити?" (перевірка прав доступу до конкретних ресурсів або захищених маршрутів на основі ролей або токенів).

**8. Для чого використовується JWT (JSON Web Token)?** JWT використовується для безпечної передачі інформації між клієнтом та сервером у вигляді криптографічно підписаного JSON-об'єкта. У REST API він слугує "перепусткою" для доступу до захищених маршрутів без необхідності щоразу вводити пароль.

**9. З яких частин складається JWT-токен?** Токен складається з трьох частин, розділених крапками:

1. **Header** (тип токена та алгоритм шифрування),
2. **Payload** (корисні дані: ID користувача, email, роль),
3. **Signature** (цифровий підпис для перевірки цілісності токена).

**10. Для чого потрібно хешування паролів і яка бібліотека використовується в роботі?** Хешування необхідне, щоб не зберігати паролі у відкритому вигляді (plain text) в базі даних. Це захищає користувачів у разі витоку БД. У роботі використано бібліотеку `bcryptjs`.

**11. Що таке валідація даних і навіщо вона потрібна?** Це процес перевірки вхідних даних (перевірка на пусті поля, мінімальну довжину пароля, формат email) перед тим, як сервер почне їх обробляти. Вона захищає БД від сміттєвих даних та запобігає збоям чи SQL-ін'єкціям.

**12. Які основні HTTP-коди стану використовуються для обробки помилок?** - **400 (Bad Request):** Неправильні або неповні дані від клієнта.

* **401 (Unauthorized):** Користувач не пройшов автентифікацію (відсутній токен).
* **403 (Forbidden):** Токен дійсний, але не вистачає прав (або токен прострочений).
* **404 (Not Found):** Ресурс не знайдено.
* **500 (Internal Server Error):** Критична помилка на стороні сервера.

**13. Що таке middleware та яку роль він виконує в Express?** Middleware — це проміжна функція, яка має доступ до об'єктів запиту (`req`) та відповіді (`res`). Її використовують для перехоплення запитів: перевірки JWT-токенів, лімітування кількості підключень, парсингу JSON або глобального логування помилок.

**14. Для чого потрібен захищений маршрут у REST API?** Щоб обмежити доступ до приватної інформації (наприклад, даних профілю, балансу, налаштувань). Отримати відповідь від такого маршруту може лише клієнт, який надасть валідний JWT-токен у заголовках запиту.

**15. Що таке роль користувача (user/admin) і як вона впливає на доступ до ресурсів?** Роль — це атрибут облікового запису, що визначає рівень привілеїв. Вона дозволяє реалізувати Role-Based Access Control (RBAC). Наприклад, звичайний `user` може редагувати лише власні публікації, тоді як `admin` має доступ до модифікації або видалення будь-яких ресурсів у системі.

---

## ВИСНОВКИ

Під час виконання лабораторної роботи було спроектовано та реалізовано сучасну, безпечну архітектуру авторизації для REST API застосунку на базі Node.js, Express та Sequelize. Було успішно імплементовано процеси реєстрації та логіну з обов'язковим хешуванням паролів за допомогою `bcryptjs`.

На практиці було опановано роботу з технологією JWT: реалізовано систему короткострокових (Access) та довгострокових (Refresh) токенів, що дозволяє підтримувати безпечну сесію користувача без збереження стану на сервері. Розроблено механізми захисту маршрутів через спеціальні middleware-перехоплювачі. Крім того, систему посилено сучасними механізмами захисту: впроваджено rate-limiter для запобігання брутфорс-атакам, створено флоу скидання пароля та підтвердження email через унікальні згенеровані токени, а також закладено архітектуру для входу через Google OAuth 2.0. Результатом є готовий, промислового рівня бекенд із повноцінним управлінням обліковими записами.

