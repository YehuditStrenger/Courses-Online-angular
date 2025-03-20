# שינויי קוד בפרויקט

הקובץ הזה כולל את כל השינויים שביצעתי על מנת לשדרג את השרת עבור הפרויקט.

## 1. קובץ: `authMiddleware.js`

### שינוי:
- הוספתי פונקציה שמוודאת שהמשתמש מחובר על ידי קריאה למזהה `user-id` מהכותרת של הבקשה (`req.header('user-id')`).
- לאחר מכן, בוצעה קריאה לקובץ `db.json` בכדי לאחזר את פרטי המשתמש על בסיס המזהה. אם המשתמש לא נמצא, מוחזרת תשובה עם סטטוס `403` (Unauthorized).

### קוד מתוקן:
~~~js
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

export default (req, res, next) => {
    const userId = req.header('user-id');
    const db = JSON.parse(fs.readFileSync(path.join(__dirname, '../db/db.json')));

    const user = db.users.find(user => user.id == userId);
    if (!user) {
        return res.status(403).json({ message: "Unauthorized" });
    }

    req.user = user;
    next();
};
~~~

## 2. קובץ: `authRoutes.js`

### שינוי:
- הוספתי אפשרות לרשום משתמש חדש בנתיב `/register`, לבדוק אם המשתמש כבר קיים, ולוודא שכל השדות הדרושים קיימים לפני הרישום.
- הוספתי אפשרות להתחבר באמצעות כתובת דוא"ל וסיסמה בנתיב `/login`.
- עדכנתי את הנתיב `/` כך שיאפשר עדכון פרטי משתמש (שם, דוא"ל, כתובת, טלפון) רק למשתמשים שמחוברים.

### קוד מתוקן:
~~~js
import express from 'express';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';
import authMiddleware from '../middleware/authMiddleware.js';

const router = express.Router();
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const dbPath = path.join(__dirname, '../db/db.json');

// רישום משתמש חדש
router.post('/register', (req, res) => {
    console.log(req.body);

    const { email, password, name, lname, addres, phone } = req.body;
    const db = JSON.parse(fs.readFileSync(dbPath));
    if (db.users.find(user => user.email === email)) {
        return res.status(400).json({ message: "User already exists" });
    }
    if (!email || !password || !name) {
        return res.status(400).json({ message: "Email, password, and first name are required." });
    }
    
    const newUser = {
        id: Date.now(),
        email,
        password,  // במציאות – להצפין סיסמא
        name,
        lname,
        addres,
        phone
    };

    db.users.push(newUser);
    fs.writeFileSync(dbPath, JSON.stringify(db, null, 2));

    res.status(201).json({ message: "User registered successfully", userId: newUser.id });
});

// התחברות
router.post('/login', (req, res) => {
    const { email, password } = req.body;
    const db = JSON.parse(fs.readFileSync(dbPath));

    const user = db.users.find(user => user.email === email && user.password === password);

    if (!user) {
        return res.status(401).json({ message: "Invalid credentials" });
    }

    res.json({ message: "Login successful", user });
});

// עדכון פרטי משתמש
router.put('/', authMiddleware, (req, res) => {
    const { name, lname, email, addres, phone, password } = req.body;
    const id = parseInt(req.header('user-id'));

    const db = JSON.parse(fs.readFileSync(dbPath));

    const user = db.users.find(user => user.id === id);

    if (!user) {
        return res.status(404).json({ message: "User not found" });
    }

    user.name = name;
    user.lname = lname;
    user.email = email;
    user.addres = addres;
    user.phone = phone;
    user.password = password;

    fs.writeFileSync(dbPath, JSON.stringify(db, null, 2));

    res.json(user);
});

export default router;
~~~

## 3. קובץ: `recipeRoutes.js`

### שינוי:
- הוספתי את הנתיב `/` על מנת לשלוף את כל המתכונים ממסד הנתונים.
- הוספתי את הנתיב `/` גם להוספת מתכון חדש, ומבצע בדיקת אימות באמצעות middleware.
- הוספתי את האפשרות לעדכן מתכון קיים אם הוא נמצא במסד הנתונים.

### קוד מתוקן:
~~~js
import express from 'express';
import fs from 'fs';
import path from 'path';
import authMiddleware from '../middleware/authMiddleware.js';
import { fileURLToPath } from 'url';

const router = express.Router();
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const dbPath = path.join(__dirname, '../db/db.json');

// שליפת כל המתכונים
router.get('/', (req, res) => {
    const db = JSON.parse(fs.readFileSync(dbPath));
    res.json(db.recipes);
});

// הוספת מתכון (רק למשתמש מחובר)
router.post('/', authMiddleware, (req, res) => {
    const {
        title,
        description,
        products,
        ingredients,
        instructions
    } = req.body;
    const db = JSON.parse(fs.readFileSync(dbPath));

    const newRecipe = {
        id: Date.now(),
        title,
        products,
        description,
        authorId: req.header('user-id'),
        ingredients,
        instructions,
    };

    db.recipes.push(newRecipe);
    fs.writeFileSync(dbPath, JSON.stringify(db, null, 2));

    res.status(201).json({ message: "Recipe added", recipe: newRecipe });
});

// עדכון מתכון
router.put('/', authMiddleware, (req, res) => {
    const {
        title,
        description,
        products,
        ingredients,
        instructions
    } = req.body;
    const id = parseInt(req.header('recipe-id'));
    const db = JSON.parse(fs.readFileSync(dbPath));

    const recipe = db.recipes.find(recipe => recipe.id === id);

    if (!recipe) {
        return res.status(404).json({ message: "Recipe not found" });
    }

    recipe.title = title;
    recipe.description = description;
    recipe.products = products;
    recipe.ingredients = ingredients;
    recipe.instructions = instructions;
    recipe.authorId = req.header('user-id');
    fs.writeFileSync(dbPath, JSON.stringify(db, null, 2));

    res.json({ message: "Recipe updated", recipe });
});

export default router;
~~~
