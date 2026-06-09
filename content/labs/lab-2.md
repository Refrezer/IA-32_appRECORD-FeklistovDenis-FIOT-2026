<div style="text-align: center;">
  <strong>НАЦІОНАЛЬНИЙ ТЕХНІЧНИЙ УНІВЕРСИТЕТ УКРАЇНИ</strong><br>
  <strong>«КИЇВСЬКИЙ ПОЛІТЕХНІЧНИЙ ІНСТИТУТ імені ІГОРЯ СІКОРСЬКОГО»</strong><br>
  Факультет інформатики та обчислювальної техніки<br>
  Кафедра інформаційних систем та технологій<br>
  <br><br>
  <strong>Лабораторна робота № 2</strong><br>
  з дисципліни «WEB-орієнтовані технології. Backend розробки»<br>
  на тему: <strong>«СТВОРЕННЯ БАЗИ ДАНИХ. ПІДКЛЮЧЕННЯ NODE.JS ДО БАЗИ ДАНИХ. РОБОТА З ORM SEQUELIZE»</strong><br>
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

* Навчитися створювати реляційну базу даних та структурувати її компоненти.
* Освоїти виконання базових SQL-запитів та CRUD-операцій за допомогою об'єктно-реляційного відображення.
* Підключити серверну програму на Node.js до реляційної бази даних.
* Вивчити архітектуру та практичне використання ORM Sequelize для роботи з моделями даних.
* Реалізувати зв’язок типу `One-to-Many` (Один-до-багатьох) між таблицями на рівні моделей та бази даних.

---

## 1. ТЕОРЕТИЧНІ ВІДОМОСТІ

**Node.js** — це асинхронне середовище виконання JavaScript на стороні сервера, що забезпечує високу продуктивність під час обробки HTTP-запитів та взаємодії з файловими системами й базами даних.

**Sequelize** — це сучасна ORM (Object-Relational Mapping) бібліотека для Node.js, яка відображає структуру реляційної бази даних у вигляді об’єктів програмування. Завдяки цьому розробник може керувати даними, створювати таблиці та виконувати CRUD-операції через JavaScript-класи (моделі) й асинхронні методи замість написання сирих SQL-запитів.

**SQLite** — це компактна вбудована реляційна база даних, яка повністю підтримується Sequelize. На відміну від великих клієнт-серверних СУБД (як-от MySQL або PostgreSQL), SQLite зберігає всю структуру та дані в одному локальному файлі (`database.sqlite`) всередині проєкту. Це усуває потребу в розгортанні локального сервера СУБД, забезпечуючи максимальну портативність серверного застосунку, зберігаючи при цьому автентичну SQL-логіку, типізацію та механізми забезпечення цілісності даних (включаючи зовнішні ключі та транзакційність).

---

## 2. ХІД ВИКОНАННЯ РОБОТИ (ПРАКТИЧНА РЕАЛІЗАЦІЯ)

### 2.1 Організація файлової структури проєкту

У межах виконання роботи структуру проєкту було масштабовано для підтримки архітектурного патерну розділення конфігурацій та моделей даних:

```text
lab2-rest-api/
│
├── config/
│   └── database.js     # Конфігурація підключення до бази даних через Sequelize
│
├── models/
│   ├── User.js         # Модель користувача (Users)
│   └── Post.js         # Модель публікації (Posts)
│
├── database.sqlite     # Локальний файл реляційної бази даних
├── server.js           # Головний файл сервера, маршрути REST API
├── package.json        # Залежності проєкту
└── node_modules/

```

### 2.2 Налаштування підключення до бази даних

Для реалізації з'єднання з базою даних створено модуль `config/database.js`. В якості діалекту вказано `sqlite`:

```javascript
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize({
  dialect: 'sqlite',
  storage: './database.sqlite',
  logging: false 
});

module.exports = sequelize;

```

### 2.3 Опис моделей даних та встановлення зв'язків

Згідно із завданням предметної області, було створено дві пов'язані сутності: `User` (Користувач) та `Post` (Публікація).

**Модель User (`models/User.js`):**

```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
  name: {
    type: DataTypes.STRING,
    allowNull: false
  },
  email: {
    type: DataTypes.STRING,
    allowNull: false,
    unique: true
  }
});

module.exports = User;

```

**Модель Post (`models/Post.js`):**

```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Post = sequelize.define('Post', {
  title: {
    type: DataTypes.STRING,
    allowNull: false
  },
  content: {
    type: DataTypes.TEXT,
    allowNull: false
  }
});

module.exports = Post;

```

### 2.4 Реалізація One-to-Many зв'язку та REST API маршрутів

У головному файлі застосунку `server.js` було задекларовано відношення **Один-до-багатьох (One-to-Many)**.

Також реалізовано маршрути для виконання операцій додавання, читання, оновлення та видалення записів (CRUD) із застосуванням методів Sequelize.

**Фрагмент коду `server.js`:**

```javascript
const express = require('express');
const sequelize = require('./config/database');
const User = require('./models/User');
const Post = require('./models/Post');

const app = express();
app.use(express.json());

// Встановлення реляційного зв'язку One-to-Many на рівні ORM
User.hasMany(Post, { foreignKey: 'userId', onDelete: 'CASCADE' });
Post.belongsTo(User, { foreignKey: 'userId' });

// GET /users - Отримання користувачів разом із їхніми постами
app.get('/users', async (req, res) => {
    try {
        const users = await User.findAll({ include: Post });
        res.json(users);
    } catch (error) {
        res.status(500).json({ message: 'Помилка сервера', error: error.message });
    }
});

// POST /users - Створення нового користувача
app.post('/users', async (req, res) => {
    try {
        const { name, email } = req.body;
        const newUser = await User.create({ name, email });
        res.status(201).json({ message: 'Користувача створено', user: newUser });
    } catch (error) {
        res.status(400).json({ message: 'Помилка валідації даних', error: error.message });
    }
});

// POST /posts - Створення публікації, прив'язаної до автора
app.post('/posts', async (req, res) => {
    try {
        const { title, content, userId } = req.body;
        const userExists = await User.findByPk(userId);
        if (!userExists) {
            return res.status(404).json({ message: 'Вказаного користувача не знайдено' });
        }
        const newPost = await Post.create({ title, content, userId });
        res.status(201).json({ message: 'Пост опубліковано', post: newPost });
    } catch (error) {
        res.status(400).json({ message: 'Помилка створення поста', error: error.message });
    }
});

// Автоматична синхронізація моделей
sequelize.sync({ alter: true })
    .then(() => {
        console.log('✔ Все таблицы успешно синхронизированы с SQLite!');
        app.listen(3000, () => console.log(`🚀 Сервер запущен на порту 3000`));
    })
    .catch(err => console.error('❌ Критическая ошибка БД:', err));

```

---

## 3. РЕЗУЛЬТАТИ ТЕСТУВАННЯ ТА ДЕМОНСТРАЦІЯ РОБОТИ

### 3.1 Логи синхронізації та ініціалізації сховища

При запуску сервера за допомогою команди `node server.js` у вбудованому терміналі VS Code, бібліотека Sequelize успішно підключається до локального драйвера, ініціалізує структуру таблиць всередині `database.sqlite` та виводить повідомлення про успішний старт.

```
### 3.2 Тестування REST API за допомогою Postman

Для перевірки коректності зв'язків було надіслано серію запитів:

* **`POST /users`** — успішно створює сутність автора.
* **`POST /posts`** — публікує запис та автоматично пов'язує його з унікальним ідентифікатором `userId`.
* **`GET /users`** — повертає масив користувачів із вкладеними об'єктами їхніх постів, що підтверджує правильну роботу реляційного механізму `One-to-Many`.

```
---

## 4. ПОСИЛАННЯ НА РЕЗУЛЬТАТИ В GITHUB

* **Репозиторій із серверним кодом та моделями:** [Встав посилання на свій репозиторій]
* **Логічно обґрунтовані коміти (Conventional Commits):**
* `feat: встановлено пакети sqlite3 та ініціалізовано конфігурацію сховища через Sequelize`
* `feat: створено реляційні моделі User та Post з валідацією атрибутів`
* `feat: задекларовано зв'язок One-to-Many та реалізовано CRUD контролери для REST API`



---

## ВИСНОВКИ

Під час виконання лабораторної роботи №2 було успішно розширено архітектуру розробленого раніше серверного застосунку на Node.js для інтеграції з реляційним сховищем даних. На практиці опановано роботу з ORM-бібліотекою Sequelize, що дозволило абстрагуватися від написання низькорівневих SQL-інструкцій та взаємодіяти з реляційними сутностями на рівні декларативних JavaScript-класів.

Було успішно спроектовано та впроваджено моделі `User` та `Post`, реалізовано автоматичну генерацію зовнішніх ключів за допомогою зв'язку типу `One-to-Many`, а також розгорнуто REST API контролери для забезпечення повноцінного життєвого циклу даних (CRUD). Використання SQLite як вбудованого рушія підтвердило гнучкість ORM-підходу, дозволивши розгорнути повноцінну ACID-сумісну реляційну базу даних у вигляді локального файлу без втрати функціональності та потреби у встановленні сторонніх серверів.

```
