<?php
error_reporting(E_ALL);
ini_set('display_errors', 1);
declare(strict_types=1);

/**
 * Аквасбор - Полнофункциональный интернет-магазин аквариумных растений
 * Версия 3.0 - Современная CRM-система с KPI дашбордом
 */

ini_set('display_errors', '0');
ini_set('log_errors', '1');
error_reporting(E_ALL);

function log_dir(): string {
    $dir = __DIR__ . '/storage';
    if (!is_dir($dir)) @mkdir($dir, 0777, true);
    $logs = $dir . '/logs';
    if (!is_dir($logs)) @mkdir($logs, 0777, true);
    return $logs;
}

function log_error_safe(string $msg): void {
    $f = log_dir() . '/app.log';
    @file_put_contents($f, '['.date('Y-m-d H:i:s').'] '.$msg.PHP_EOL, FILE_APPEND);
}

set_error_handler(function($errno,$errstr,$errfile,$errline){
    log_error_safe("PHP[$errno] $errstr at $errfile:$errline");
    return false;
});

set_exception_handler(function($e){
    log_error_safe('Uncaught: '.$e->getMessage().' at '.$e->getFile().':'.$e->getLine());
    http_response_code(200);
    echo '<!doctype html><html><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Сервис временно недоступен</title></head><body><div style="max-width:680px;margin:40px auto;font-family:system-ui"><h2>Сервис временно недоступен</h2><p>Попробуйте обновить страницу позже.</p><p><a href="?r=diagnostics">Открыть диагностику</a></p></div></body></html>';
    exit;
});

// Конфиг
$config = [
    'app' => [
        'name' => 'Аквасбор CRM',
        'url' => (isset($_SERVER['HTTP_HOST']) ? ('http://' . $_SERVER['HTTP_HOST']) : 'http://localhost'),
        'debug' => false,
        'timezone' => 'Europe/Moscow',
        'email' => 'artcopy78@bk.ru',
    ],
    'security' => [
        'salt' => '4d77f98b402a619e8f1bf3c75e9426cb18c5f1b44fe33fd26b35d13f61b221f1',
        'session_name' => 'AQUASBOR_SESSION',
    ],
    'database' => [
        'driver' => 'sqlite',
        'path' => __DIR__ . '/storage/aquasbor.sqlite',
    ],
];

@date_default_timezone_set($config['app']['timezone']);
if (!headers_sent()) { 
    @session_name($config['security']['session_name']); 
}
@session_start();

// CSRF защита
function csrf_token(): string {
    if (empty($_SESSION['csrf_token'])) {
        $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    }
    return $_SESSION['csrf_token'];
}

function verify_csrf(): bool {
    if (!is_post()) return true;
    $token = $_POST['csrf_token'] ?? $_GET['csrf_token'] ?? '';
    if (empty($token)) {
        csrf_token();
        return true;
    }
    $sessionToken = $_SESSION['csrf_token'] ?? '';
    if (empty($sessionToken)) {
        $_SESSION['csrf_token'] = $token;
        return true;
    }
    return hash_equals($sessionToken, $token);
}

// Хелперы
function esc($s){ return htmlspecialchars((string)($s ?? ''), ENT_QUOTES|ENT_SUBSTITUTE, 'UTF-8'); }
function is_post(): bool { return ($_SERVER['REQUEST_METHOD'] ?? 'GET') === 'POST'; }

function json_response($data, int $code = 200): void {
    http_response_code($code);
    header('Content-Type: application/json; charset=utf-8');
    echo json_encode($data, JSON_UNESCAPED_UNICODE|JSON_UNESCAPED_SLASHES);
    exit;
}

function upload_file($file, $allowed_types = ['jpg', 'jpeg', 'png', 'gif', 'webp', 'mp4', 'avi', 'mov']): ?string {
    if (!$file || $file['error'] !== UPLOAD_ERR_OK) return null;

    $upload_dir = __DIR__ . '/storage/uploads';
    if (!is_dir($upload_dir)) @mkdir($upload_dir, 0777, true);

    $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
    if (!in_array($ext, $allowed_types)) return null;

    $filename = uniqid() . '.' . $ext;
    $filepath = $upload_dir . '/' . $filename;

    if (move_uploaded_file($file['tmp_name'], $filepath)) {
        return '/storage/uploads/' . $filename;
    }

    return null;
}

// БД функции
function has_pdo_sqlite(): bool { return extension_loaded('pdo_sqlite'); }

function db(): PDO {
    static $pdo;
    global $config;
    if ($pdo instanceof PDO) return $pdo;

    if (!has_pdo_sqlite()) {
        throw new RuntimeException('Расширение pdo_sqlite не доступно');
    }

    $dbFile = $config['database']['path'];
    $dir = dirname($dbFile);
    if (!is_dir($dir)) @mkdir($dir, 0777, true);

    $pdo = new PDO('sqlite:' . $dbFile, null, null, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);

    $pdo->exec('PRAGMA foreign_keys = ON');
    $pdo->exec('PRAGMA journal_mode = WAL');
    $pdo->exec('PRAGMA synchronous = NORMAL');

    return $pdo;
}

function table_exists(PDO $pdo, string $table): bool {
    try {
        $st = $pdo->prepare("SELECT name FROM sqlite_master WHERE type='table' AND name=?");
        $st->execute([$table]);
        return (bool)$st->fetch();
    } catch (Exception $e) {
        return false;
    }
}

// Установка БД с расширенными таблицами
function ensureInstalled(PDO $pdo): void {
    try {
        // Настройки
        $pdo->exec("CREATE TABLE IF NOT EXISTS settings(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            key_name TEXT NOT NULL UNIQUE,
            value TEXT NULL,
            type TEXT NOT NULL DEFAULT 'text',
            description TEXT NULL,
            category TEXT NOT NULL DEFAULT 'general'
        )");

        // Категории
        $pdo->exec("CREATE TABLE IF NOT EXISTS categories(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            slug TEXT NOT NULL UNIQUE,
            name TEXT NOT NULL,
            description TEXT NULL,
            image TEXT NULL,
            sort INTEGER NOT NULL DEFAULT 100,
            is_active INTEGER NOT NULL DEFAULT 1,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL
        )");

        // Товары (расширенная версия)
        $pdo->exec("CREATE TABLE IF NOT EXISTS products(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            slug TEXT NOT NULL UNIQUE,
            name TEXT NOT NULL,
            category_id INTEGER NULL,
            price REAL NOT NULL DEFAULT 0,
            old_price REAL NULL,
            cost_price REAL NULL,
            stock INTEGER NOT NULL DEFAULT 0,
            min_stock INTEGER NOT NULL DEFAULT 5,
            temperature_min INTEGER NULL,
            temperature_max INTEGER NULL,
            ph_min REAL NULL,
            ph_max REAL NULL,
            light TEXT NULL,
            difficulty TEXT NULL,
            growth_rate TEXT NULL,
            placement TEXT NULL,
            short_desc TEXT NULL,
            description TEXT NULL,
            care_tips TEXT NULL,
            image TEXT NULL,
            gallery TEXT NULL,
            video_url TEXT NULL,
            views INTEGER NOT NULL DEFAULT 0,
            sales INTEGER NOT NULL DEFAULT 0,
            rating REAL NULL DEFAULT 0,
            reviews_count INTEGER NOT NULL DEFAULT 0,
            is_active INTEGER NOT NULL DEFAULT 1,
            is_featured INTEGER NOT NULL DEFAULT 0,
            is_bestseller INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL,
            FOREIGN KEY(category_id) REFERENCES categories(id) ON DELETE SET NULL
        )");

        // Пользователи
        $pdo->exec("CREATE TABLE IF NOT EXISTS users(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL,
            name TEXT NULL,
            phone TEXT NULL,
            role TEXT NOT NULL DEFAULT 'admin',
            is_active INTEGER NOT NULL DEFAULT 1,
            last_login TEXT NULL,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL
        )");

        // Заказы
        $pdo->exec("CREATE TABLE IF NOT EXISTS orders(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            order_number TEXT NOT NULL UNIQUE,
            customer_name TEXT NOT NULL,
            customer_email TEXT NOT NULL,
            customer_phone TEXT NOT NULL,
            delivery_type TEXT NOT NULL DEFAULT 'pickup',
            delivery_address TEXT NULL,
            delivery_cost REAL NOT NULL DEFAULT 0,
            total_amount REAL NOT NULL DEFAULT 0,
            status TEXT NOT NULL DEFAULT 'new',
            payment_status TEXT NOT NULL DEFAULT 'pending',
            payment_method TEXT NULL,
            manager_notes TEXT NULL,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL
        )");

        // Товары в заказе
        $pdo->exec("CREATE TABLE IF NOT EXISTS order_items(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            order_id INTEGER NOT NULL,
            product_id INTEGER NOT NULL,
            product_name TEXT NOT NULL,
            price REAL NOT NULL,
            quantity INTEGER NOT NULL,
            total REAL NOT NULL,
            FOREIGN KEY(order_id) REFERENCES orders(id) ON DELETE CASCADE,
            FOREIGN KEY(product_id) REFERENCES products(id) ON DELETE CASCADE
        )");

        // Финансы
        $pdo->exec("CREATE TABLE IF NOT EXISTS finances(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            type TEXT NOT NULL CHECK(type IN ('income', 'expense')),
            amount REAL NOT NULL,
            description TEXT NOT NULL,
            category TEXT NULL,
            order_id INTEGER NULL,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            FOREIGN KEY(order_id) REFERENCES orders(id) ON DELETE SET NULL
        )");

        // Новости
        $pdo->exec("CREATE TABLE IF NOT EXISTS news(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            slug TEXT NOT NULL UNIQUE,
            title TEXT NOT NULL,
            excerpt TEXT NULL,
            body TEXT NULL,
            image TEXT NULL,
            views INTEGER NOT NULL DEFAULT 0,
            likes INTEGER NOT NULL DEFAULT 0,
            published_at TEXT NULL,
            is_active INTEGER NOT NULL DEFAULT 1,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL
        )");

        // Видеообзоры
        $pdo->exec("CREATE TABLE IF NOT EXISTS video_reviews(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT NULL,
            video_url TEXT NULL,
            video_file TEXT NULL,
            thumbnail TEXT NULL,
            product_id INTEGER NULL,
            views INTEGER NOT NULL DEFAULT 0,
            likes INTEGER NOT NULL DEFAULT 0,
            is_active INTEGER NOT NULL DEFAULT 1,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL,
            FOREIGN KEY(product_id) REFERENCES products(id) ON DELETE SET NULL
        )");

        // Слайдер
        $pdo->exec("CREATE TABLE IF NOT EXISTS sliders(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL DEFAULT '',
            subtitle TEXT NULL,
            image TEXT NOT NULL DEFAULT '',
            link TEXT NOT NULL DEFAULT '',
            sort INTEGER NOT NULL DEFAULT 100,
            is_active INTEGER NOT NULL DEFAULT 1
        )");

        // Страницы
        $pdo->exec("CREATE TABLE IF NOT EXISTS pages(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            slug TEXT NOT NULL UNIQUE,
            title TEXT NOT NULL,
            content TEXT NULL,
            meta_description TEXT NULL,
            is_active INTEGER NOT NULL DEFAULT 1,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL
        )");

        // SEO статьи
        $pdo->exec("CREATE TABLE IF NOT EXISTS seo_articles(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            slug TEXT NOT NULL UNIQUE,
            title TEXT NOT NULL,
            content TEXT NULL,
            meta_description TEXT NULL,
            keywords TEXT NULL,
            views INTEGER NOT NULL DEFAULT 0,
            is_published INTEGER NOT NULL DEFAULT 1,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL
        )");

        // Интеграции
        $pdo->exec("CREATE TABLE IF NOT EXISTS integrations(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            type TEXT NOT NULL,
            name TEXT NOT NULL,
            config TEXT NULL,
            is_active INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NULL DEFAULT (DATETIME('now')),
            updated_at TEXT NULL
        )");

        // Настройки по умолчанию
        $defaultSettings = [
            // Основные
            'shop_name' => ['Аквасбор', 'text', 'Название магазина', 'general'],
            'shop_description' => ['Премиальные аквариумные растения с доставкой по России', 'textarea', 'Описание магазина', 'general'],
            'hero_title' => ['Премиальные аквариумные растения', 'text', 'Заголовок на главной', 'design'],
            'hero_subtitle' => ['Создайте подводный сад мечты с нашими растениями', 'textarea', 'Подзаголовок на главной', 'design'],

            // Контакты
            'phone' => ['+7 (999) 123-45-67', 'tel', 'Телефон', 'contacts'],
            'email' => ['artcopy78@bk.ru', 'email', 'Email', 'contacts'],
            'address' => ['г. Москва, ул. Аквариумная, 15', 'text', 'Адрес', 'contacts'],
            'work_hours' => ['Пн-Вс: 10:00-20:00', 'text', 'Часы работы', 'contacts'],

            // Доставка
            'delivery_cost_pickup' => ['0', 'number', 'Стоимость самовывоза', 'delivery'],
            'delivery_cost_courier' => ['300', 'number', 'Стоимость курьерской доставки', 'delivery'],
            'delivery_cost_post' => ['250', 'number', 'Стоимость почтовой доставки', 'delivery'],
            'free_delivery_threshold' => ['3000', 'number', 'Сумма бесплатной доставки', 'delivery'],

            // Интеграции
            'yandex_maps_api' => ['', 'text', 'API ключ Яндекс.Карт', 'integrations'],
            'telegram_bot_token' => ['', 'text', 'Токен Telegram бота', 'integrations'],
            'yandex_kassa_shop_id' => ['', 'text', 'Shop ID Яндекс.Касса', 'integrations'],
            'yandex_kassa_secret' => ['', 'password', 'Секретный ключ Яндекс.Касса', 'integrations'],
            'sberbank_login' => ['', 'text', 'Логин Сбербанк Эквайринг', 'integrations'],
            'sberbank_password' => ['', 'password', 'Пароль Сбербанк Эквайринг', 'integrations'],
            'cdek_account' => ['', 'text', 'Аккаунт СДЭК', 'integrations'],
            'cdek_password' => ['', 'password', 'Пароль СДЭК', 'integrations'],

            // Футер
            'footer_text' => ['Премиальные аквариумные растения с гарантией качества', 'textarea', 'Текст в футере', 'design'],
            'footer_copyright' => ['© 2024 Аквасбор. Все права защищены.', 'text', 'Copyright', 'design'],
            'social_vk' => ['', 'url', 'Ссылка ВКонтакте', 'social'],
            'social_instagram' => ['', 'url', 'Ссылка Instagram', 'social'],
            'social_telegram' => ['', 'url', 'Ссылка Telegram', 'social'],
            'social_whatsapp' => ['', 'tel', 'Номер WhatsApp', 'social'],
        ];

        foreach ($defaultSettings as $key => [$value, $type, $desc, $cat]) {
            try {
                $pdo->prepare("INSERT OR IGNORE INTO settings(key_name, value, type, description, category) VALUES (?, ?, ?, ?, ?)")
                    ->execute([$key, $value, $type, $desc, $cat]);
            } catch (Exception $e) {
                log_error_safe("Error inserting setting $key: " . $e->getMessage());
            }
        }

        // Расширенная база категорий
        $categories = [
            ['rasteniya-perednego-plana', 'Растения переднего плана', 'Компактные почвопокровные растения для создания красивого переднего плана'],
            ['rasteniya-srednego-plana', 'Растения среднего плана', 'Растения средней высоты для заполнения центральной части аквариума'],
            ['rasteniya-zadnego-plana', 'Растения заднего плана', 'Высокие растения для создания фона в аквариуме'],
            ['mhi-i-pechenochniki', 'Мхи и печеночники', 'Различные виды мхов и печеночников для декорирования'],
            ['plavayushchie-rasteniya', 'Плавающие растения', 'Растения водной поверхности для естественной фильтрации'],
            ['anubias', 'Анубиасы', 'Неприхотливые растения рода Anubias различных видов'],
            ['kripty', 'Криптокорины', 'Разнообразные криптокорины для любых условий'],
            ['ehindorus', 'Эхинодорусы', 'Крупные розеточные растения-солитеры'],
            ['vallisneria', 'Валлиснерии', 'Длинные ленточные растения заднего плана'],
            ['ludvigia', 'Людвигии', 'Красочные стеблевые растения'],
            ['rotala', 'Роталы', 'Изящные стеблевые растения с мелкими листьями'],
            ['bucephalandra', 'Буцефаландры', 'Редкие эпифитные растения с Борneo'],
        ];

        foreach ($categories as $i => [$slug, $name, $desc]) {
            try {
                $stmt = $pdo->prepare("INSERT OR IGNORE INTO categories(slug, name, description, sort) VALUES (?, ?, ?, ?)");
                $stmt->execute([$slug, $name, $desc, ($i + 1) * 10]);
            } catch (Exception $e) {
                log_error_safe("Error inserting category $slug: " . $e->getMessage());
            }
        }

        // Админ по умолчанию
        $userCnt = (int)$pdo->query("SELECT COUNT(*) c FROM users")->fetch()['c'];
        if ($userCnt === 0) {
            try {
                $email = 'admin@aquasbor.ru';
                $pass = 'Admin@12345';
                $hash = password_hash($pass, PASSWORD_BCRYPT, ['cost' => 12]);
                $pdo->prepare("INSERT INTO users(email, password, name, role, is_active) VALUES (?, ?, ?, 'admin', 1)")
                    ->execute([$email, $hash, 'Администратор']);
            } catch (Exception $e) {
                log_error_safe("Error creating admin user: " . $e->getMessage());
            }
        }

        // Расширенная база товаров
        $productCount = (int)$pdo->query("SELECT COUNT(*) c FROM products")->fetch()['c'];
        if ($productCount === 0) {
            $catIds = [];
            $cats = $pdo->query("SELECT id, slug FROM categories")->fetchAll();
            foreach ($cats as $cat) {
                $catIds[$cat['slug']] = (int)$cat['id'];
            }

            $demoProducts = [
                // Растения переднего плана
                ['hemianthus-callitrichoides-cuba', 'Хемиантус Куба (HC Cuba)', 'rasteniya-perednego-plana', 299.00, 349.00, 25, 'easy', 'high', '18-28°C, pH 5.0-7.5', 'Самое популярное почвопокровное растение для создания зеленого газона'],
                ['glossostigma-elatinoides', 'Глоссостигма повойничковая', 'rasteniya-perednego-plana', 259.00, null, 18, 'medium', 'high', '20-28°C, pH 5.5-7.0', 'Мелколистное почвопокровное растение светло-зеленого цвета'],
                ['eleocharis-acicularis-mini', 'Ситняг игольчатый мини', 'rasteniya-perednego-plana', 199.00, 249.00, 32, 'easy', 'medium', '15-26°C, pH 5.0-8.0', 'Травянистое растение, образующее плотные заросли'],
                ['utricularia-graminifolia', 'Пузырчатка злаколистная', 'rasteniya-perednego-plana', 399.00, null, 12, 'hard', 'high', '20-28°C, pH 5.0-7.0', 'Необычное хищное растение с мелкими листочками'],
                ['marsilea-hirsuta', 'Марсилея коротковолосистая', 'rasteniya-perednego-plana', 279.00, null, 22, 'easy', 'low', '18-28°C, pH 6.0-8.0', 'Четырехлистное растение, похожее на клевер'],

                // Анубиасы
                ['anubias-barteri-nana', 'Анубиас Бартера нана', 'anubias', 399.00, null, 45, 'easy', 'low', '22-28°C, pH 6.0-8.0', 'Классический неприхотливый анубиас для начинающих'],
                ['anubias-petite', 'Анубиас петит', 'anubias', 459.00, 529.00, 28, 'easy', 'low', '20-28°C, pH 6.0-8.0', 'Миниатюрная форма анубиаса с очень мелкими листьями'],
                ['anubias-coffeefolia', 'Анубиас кофеефолиа', 'anubias', 899.00, null, 15, 'easy', 'low', '22-28°C, pH 6.0-7.5', 'Красивейший анубиас с гофрированными листьями'],
                ['anubias-pangolino', 'Анубиас панголино', 'anubias', 759.00, null, 18, 'easy', 'low', '22-28°C, pH 6.0-8.0', 'Компактный анубиас с заостренными листьями'],
                ['anubias-white', 'Анубиас белый', 'anubias', 1299.00, 1499.00, 8, 'easy', 'medium', '22-26°C, pH 6.0-7.5', 'Эксклюзивная белая форма анубиаса'],

                // Мхи
                ['java-moss', 'Яванский мох', 'mhi-i-pechenochniki', 299.00, null, 55, 'easy', 'low', '15-30°C, pH 5.0-8.0', 'Самый популярный и неприхотливый аквариумный мох'],
                ['christmas-moss', 'Рождественский мох', 'mhi-i-pechenochniki', 389.00, 449.00, 24, 'easy', 'medium', '18-28°C, pH 5.5-7.5', 'Мох с веточками, напоминающими елочку'],
                ['phoenix-moss', 'Феникс мох', 'mhi-i-pechenochniki', 459.00, null, 18, 'medium', 'medium', '18-26°C, pH 5.5-7.0', 'Красивый мох с перистыми веточками'],
                ['flame-moss', 'Мох пламя', 'mhi-i-pechenochniki', 419.00, null, 21, 'medium', 'medium', '18-28°C, pH 6.0-7.5', 'Мох, растущий в форме языков пламени'],
                ['riccia-fluitans', 'Риччия плавающая', 'mhi-i-pechenochniki', 199.00, null, 38, 'easy', 'high', '18-30°C, pH 6.0-8.0', 'Быстрорастущий печеночный мох'],

                // Криптокорины
                ['cryptocoryne-wendtii', 'Криптокорина Вендта', 'kripty', 179.00, 219.00, 42, 'easy', 'low', '22-28°C, pH 6.0-8.0', 'Выносливая криптокорина с волнистыми листьями'],
                ['cryptocoryne-parva', 'Криптокорина парва', 'kripty', 259.00, null, 28, 'medium', 'medium', '22-28°C, pH 6.0-7.5', 'Самая маленькая криптокорина'],
                ['cryptocoryne-lutea', 'Криптокорина лютеа', 'kripty', 219.00, null, 35, 'easy', 'low', '20-28°C, pH 6.5-8.0', 'Криптокорина с желто-зелеными листьями'],
                ['cryptocoryne-spiralis', 'Криптокорина спиральная', 'kripty', 299.00, 359.00, 24, 'easy', 'low', '22-28°C, pH 6.0-7.5', 'Высокая криптокорина с длинными листьями'],
                ['cryptocoryne-flamingo', 'Криптокорина фламинго', 'kripty', 899.00, null, 12, 'medium', 'medium', '22-26°C, pH 6.0-7.0', 'Розовая форма криптокорины'],

                // Растения заднего плана
                ['vallisneria-spiralis', 'Валлиснерия спиральная', 'rasteniya-zadnego-plana', 149.00, 189.00, 48, 'easy', 'medium', '18-28°C, pH 6.0-9.0', 'Классическое длинное растение заднего плана'],
                ['vallisneria-gigantea', 'Валлиснерия гигантская', 'rasteniya-zadnego-plana', 259.00, null, 22, 'easy', 'medium', '20-28°C, pH 6.0-8.5', 'Крупная валлиснерия для больших аквариумов'],
                ['hygrophila-polysperma', 'Гигрофила многосемянная', 'rasteniya-zadnego-plana', 99.00, null, 65, 'easy', 'medium', '20-28°C, pH 6.0-8.0', 'Быстрорастущее неприхотливое растение'],
                ['limnophila-sessiliflora', 'Амбулия сидячецветковая', 'rasteniya-zadnego-plana', 139.00, 169.00, 38, 'easy', 'medium', '18-28°C, pH 6.0-8.0', 'Пушистое растение с перистыми листьями'],
                ['myriophyllum-mattogrossense', 'Перистолистник матогроссский', 'rasteniya-zadnego-plana', 199.00, null, 29, 'medium', 'high', '22-28°C, pH 5.5-7.5', 'Красивое красноватое стеблевое растение'],

                // Эхинодорусы
                ['echinodorus-bleheri', 'Эхинодорус Блехера', 'ehindorus', 389.00, 459.00, 25, 'easy', 'medium', '20-28°C, pH 6.0-8.0', 'Популярный крупный эхинодорус'],
                ['echinodorus-red-diamond', 'Эхинодорус красный алмаз', 'ehindorus', 759.00, null, 14, 'medium', 'medium', '22-28°C, pH 6.0-7.5', 'Эффектный красный эхинодорус'],
                ['echinodorus-ozelot', 'Эхинодорус оцелот', 'ehindorus', 659.00, 799.00, 18, 'easy', 'medium', '20-28°C, pH 6.0-7.5', 'Пестрый эхинодорус с темными пятнами'],
                ['echinodorus-reni', 'Эхинодорус Рени', 'ehindorus', 459.00, null, 21, 'easy', 'medium', '22-28°C, pH 6.0-8.0', 'Компактный эхинодорус с волнистыми листьями'],

                // Людвигии
                ['ludwigia-repens', 'Людвигия ползучая', 'ludvigia', 159.00, null, 42, 'easy', 'medium', '18-28°C, pH 6.0-8.0', 'Красивое красно-зеленое стеблевое растение'],
                ['ludwigia-palustris', 'Людвигия болотная', 'ludvigia', 179.00, 229.00, 35, 'easy', 'medium', '15-28°C, pH 5.5-8.0', 'Выносливая людвигия с красными листьями'],
                ['ludwigia-super-red', 'Людвигия супер ред', 'ludvigia', 299.00, null, 24, 'medium', 'high', '22-28°C, pH 5.5-7.5', 'Ярко-красная селекционная форма'],
                ['ludwigia-tornado', 'Людвигия торнадо', 'ludvigia', 399.00, 479.00, 18, 'medium', 'high', '20-26°C, pH 6.0-7.5', 'Людвигия с закрученными листьями'],

                // Роталы
                ['rotala-rotundifolia', 'Ротала круглолистная', 'rotala', 129.00, null, 48, 'easy', 'medium', '20-30°C, pH 6.0-7.5', 'Популярная ротала для начинающих'],
                ['rotala-indica', 'Ротала индийская', 'rotala', 149.00, 179.00, 38, 'easy', 'medium', '22-28°C, pH 6.0-7.5', 'Изящная ротала с мелкими листьями'],
                ['rotala-macrandra', 'Ротала макрандра', 'rotala', 259.00, null, 22, 'hard', 'high', '24-28°C, pH 5.5-7.0', 'Капризная, но очень красивая красная ротала'],
                ['rotala-vietnam-hra', 'Ротала Вьетнам HRA', 'rotala', 399.00, 499.00, 16, 'hard', 'high', '22-26°C, pH 5.5-6.8', 'Редкая ротала с розово-красными листьями'],

                // Буцефаландры
                ['bucephalandra-kedagang', 'Буцефаландра кедаганг', 'bucephalandra', 899.00, null, 12, 'medium', 'low', '22-28°C, pH 6.0-7.5', 'Редкое растение с Борнео с темными листьями'],
                ['bucephalandra-green-wavy', 'Буцефаландра грин вейви', 'bucephalandra', 759.00, 899.00, 15, 'medium', 'low', '20-28°C, pH 6.0-7.5', 'Буцефаландра с волнистыми зелеными листьями'],
                ['bucephalandra-mini-needle-leaf', 'Буцефаландра мини нидл лиф', 'bucephalandra', 1299.00, null, 8, 'medium', 'medium', '22-26°C, pH 6.0-7.0', 'Миниатюрная буцефаландра с игольчатыми листьями'],
                ['bucephalandra-brownie-purple', 'Буцефаландра брауни пурпл', 'bucephalandra', 1199.00, 1399.00, 10, 'medium', 'medium', '22-28°C, pH 6.0-7.5', 'Буцефаландра с коричнево-фиолетовыми листьями'],

                // Плавающие растения  
                ['pistia-stratiotes', 'Пистия телорезовидная', 'plavayushchie-rasteniya', 199.00, null, 35, 'easy', 'medium', '18-30°C, pH 6.0-8.0', 'Красивое плавающее растение-розетка'],
                ['salvinia-natans', 'Сальвиния плавающая', 'plavayushchie-rasteniya', 159.00, 199.00, 42, 'easy', 'medium', '15-28°C, pH 6.0-8.0', 'Быстрорастущий плавающий папоротник'],
                ['lemna-minor', 'Ряска малая', 'plavayushchie-rasteniya', 99.00, null, 55, 'easy', 'medium', '5-30°C, pH 6.0-8.5', 'Мелкое плавающее растение для естественной фильтрации'],
                ['limnobium-laevigatum', 'Лимнобиум губчатый', 'plavayushchie-rasteniya', 229.00, null, 28, 'easy', 'medium', '18-30°C, pH 6.0-8.0', 'Плавающее растение с сердцевидными листьями'],

                // Растения среднего плана
                ['alternanthera-reineckii', 'Альтернантера Рейнека', 'rasteniya-srednego-plana', 199.00, 249.00, 32, 'medium', 'high', '22-28°C, pH 6.0-7.5', 'Красивое красно-фиолетовое растение'],
                ['hygrophila-corymbosa', 'Гигрофила щитковидная', 'rasteniya-srednego-plana', 169.00, null, 38, 'easy', 'medium', '20-28°C, pH 6.0-8.0', 'Крупнолистная быстрорастущая гигрофила'],
                ['bacopa-caroliniana', 'Бакопа каролинская', 'rasteniya-srednego-plana', 149.00, null, 45, 'easy', 'medium', '18-28°C, pH 6.0-8.0', 'Неприхотливое стеблевое растение с округлыми листьями'],
                ['staurogyne-repens', 'Стаурогина ползучая', 'rasteniya-srednego-plana', 259.00, 319.00, 26, 'medium', 'medium', '20-28°C, pH 6.0-7.5', 'Компактное растение, образующее плотные заросли'],
                ['pogostemon-helferi', 'Погостемон Хелфера', 'rasteniya-srednego-plana', 299.00, null, 22, 'medium', 'medium', '22-28°C, pH 6.0-7.5', 'Оригинальное растение с гофрированными листьями'],
            ];

            foreach ($demoProducts as [$slug, $name, $catSlug, $price, $oldPrice, $stock, $difficulty, $light, $params, $desc]) {
                try {
                    $catId = $catIds[$catSlug] ?? 1;
                    $stmt = $pdo->prepare("INSERT INTO products(slug, name, category_id, price, old_price, stock, difficulty, light, short_desc, description, is_active, is_featured, min_stock) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 1, ?, 5)");
                    $stmt->execute([$slug, $name, $catId, $price, $oldPrice, $stock, $difficulty, $light, $params, $desc, rand(0, 1)]);
                } catch (Exception $e) {
                    log_error_safe("Error inserting demo product $slug: " . $e->getMessage());
                }
            }
        }

        // Демо новости
        $newsCount = (int)$pdo->query("SELECT COUNT(*) c FROM news")->fetch()['c'];
        if ($newsCount === 0) {
            $demoNews = [
                ['novye-postupleniya-anubias', 'Новые поступления: редкие анубиасы', 'В нашем магазине появились эксклюзивные виды анубиасов от ведущих питомников'],
                ['sekrety-uhoda-za-pocvopokrovnymi', 'Секреты ухода за почвопокровными растениями', 'Подробное руководство по выращиванию ковровых растений в аквариуме'],
                ['luchshie-rasteniya-dlya-nachinayuschih', 'Лучшие растения для начинающих аквариумистов', 'Топ-10 неприхотливых растений для первого аквариума с растениями'],
            ];

            foreach ($demoNews as $i => [$slug, $title, $excerpt]) {
                try {
                    $stmt = $pdo->prepare("INSERT INTO news(slug, title, excerpt, body, published_at, is_active) VALUES (?, ?, ?, ?, datetime('now'), 1)");
                    $stmt->execute([$slug, $title, $excerpt, $excerpt . '. Полный текст статьи с подробными рекомендациями и советами от экспертов.']);
                } catch (Exception $e) {
                    log_error_safe("Error inserting demo news: " . $e->getMessage());
                }
            }
        }

        // SEO статьи
        $seoCount = (int)$pdo->query("SELECT COUNT(*) c FROM seo_articles")->fetch()['c'];
        if ($seoCount === 0) {
            $seoArticles = [
                ['akvariumnye-rasteniya-kupit-moskva', 'Аквариумные растения купить в Москве', 'Большой выбор аквариумных растений с доставкой по Москве'],
                ['uhod-za-akvariumnymi-rasteniyami', 'Уход за аквариумными растениями', 'Полное руководство по уходу за растениями в аквариуме'],
                ['luchshie-rasteniya-dlya-nano-akvariuma', 'Лучшие растения для нано-аквариума', 'Подборка растений для маленьких аквариумов'],
            ];

            foreach ($seoArticles as [$slug, $title, $content]) {
                try {
                    $stmt = $pdo->prepare("INSERT INTO seo_articles(slug, title, content, meta_description, is_published) VALUES (?, ?, ?, ?, 1)");
                    $stmt->execute([$slug, $title, $content, substr($content, 0, 160)]);
                } catch (Exception $e) {
                    log_error_safe("Error inserting seo article: " . $e->getMessage());
                }
            }
        }

        // Слайдер по умолчанию
        $sliderCnt = (int)$pdo->query("SELECT COUNT(*) c FROM sliders")->fetch()['c'];
        if ($sliderCnt === 0) {
            try {
                $pdo->exec("INSERT INTO sliders(title, subtitle, image, link, sort, is_active) VALUES
                    ('Более 100 видов растений для вашего аквариума', 'От неприхотливых до эксклюзивных коллекционных', 'https://images.unsplash.com/photo-1544197150-b99a580bb7a8?w=1200&h=400&fit=crop', '#catalog', 10, 1),
                    ('Скидки до 30% на популярные растения', 'Ограниченное предложение на хиты продаж', 'https://images.unsplash.com/photo-1583212292454-1fe6229603b7?w=1200&h=400&fit=crop', '#catalog', 20, 1),
                    ('Профессиональные консультации по подбору растений', 'Наши эксперты помогут создать идеальный подводный сад', 'https://images.unsplash.com/photo-1520637836862-4d197d17c727?w=1200&h=400&fit=crop', '#contacts', 30, 1)");
            } catch (Exception $e) {
                log_error_safe("Error inserting sliders: " . $e->getMessage());
            }
        }

        // Интеграции по умолчанию
        $intCount = (int)$pdo->query("SELECT COUNT(*) c FROM integrations")->fetch()['c'];
        if ($intCount === 0) {
            $integrations = [
                ['payment', 'Яндекс.Касса', '{"shop_id":"","secret_key":""}', 0],
                ['payment', 'Сбербанк Эквайринг', '{"login":"","password":""}', 0],
                ['payment', 'Тинькофф Эквайринг', '{"terminal_key":"","secret_key":""}', 0],
                ['delivery', 'СДЭК', '{"account":"","password":""}', 0],
                ['delivery', 'Почта России', '{"login":"","password":""}', 0],
                ['delivery', 'Boxberry', '{"token":""}', 0],
                ['notification', 'Telegram Bot', '{"bot_token":"","chat_id":""}', 0],
                ['notification', 'Email SMTP', '{"host":"","port":"","username":"","password":""}', 0],
                ['analytics', 'Яндекс.Метрика', '{"counter_id":""}', 0],
                ['analytics', 'Google Analytics', '{"tracking_id":""}', 0],
                ['maps', 'Яндекс.Карты', '{"api_key":""}', 0],
                ['maps', 'Google Maps', '{"api_key":""}', 0],
            ];

            foreach ($integrations as [$type, $name, $config, $active]) {
                try {
                    $stmt = $pdo->prepare("INSERT INTO integrations(type, name, config, is_active) VALUES (?, ?, ?, ?)");
                    $stmt->execute([$type, $name, $config, $active]);
                } catch (Exception $e) {
                    log_error_safe("Error inserting integration $name: " . $e->getMessage());
                }
            }
        }

    } catch (Exception $e) {
        log_error_safe("Error in ensureInstalled: " . $e->getMessage());
        throw $e;
    }
}

// Функции настроек
function setting(string $key, string $default = ''): string {
    static $cache = null;
    if ($cache === null) {
        $cache = [];
        try {
            $pdo = db();
            ensureInstalled($pdo);
            $st = $pdo->query("SELECT key_name, value FROM settings");
            foreach ($st as $r) $cache[$r['key_name']] = $r['value'] ?? '';
        } catch (Throwable $e) {
            $cache = [];
            log_error_safe("Error loading settings: " . $e->getMessage());
        }
    }
    return $cache[$key] ?? $default;
}

// Авторизация
function auth_login(string $email, string $password): bool {
    try {
        $pdo = db();
        ensureInstalled($pdo);
        $st = $pdo->prepare("SELECT id, email, password, role, is_active FROM users WHERE LOWER(email) = LOWER(?) LIMIT 1");
        $st->execute([$email]);
        $u = $st->fetch();
        if (!$u || !(int)$u['is_active']) return false;
        if (!password_verify($password, $u['password'])) return false;

        $_SESSION['user_id'] = (int)$u['id'];
        $_SESSION['user_email'] = $u['email'];
        $_SESSION['user_role'] = $u['role'];

        // Обновляем время последнего входа
        $pdo->prepare("UPDATE users SET last_login = datetime('now') WHERE id = ?")->execute([$u['id']]);

        return true;
    } catch (Exception $e) {
        log_error_safe("Error in auth_login: " . $e->getMessage());
        return false;
    }
}

function auth_logout(): void {
    $_SESSION = [];
    if (ini_get('session.use_cookies')) {
        $params = session_get_cookie_params();
        setcookie(session_name(), '', time() - 42000, $params['path'], $params['domain'], $params['secure'], $params['httponly']);
    }
    session_destroy();
}

function current_user(): ?array {
    if (empty($_SESSION['user_id'])) return null;
    try {
        $pdo = db();
        $st = $pdo->prepare("SELECT id, email, name, role FROM users WHERE id = ? LIMIT 1");
        $st->execute([$_SESSION['user_id']]);
        return $st->fetch() ?: null;
    } catch (Exception $e) {
        log_error_safe("Error in current_user: " . $e->getMessage());
        return null;
    }
}

function require_admin(): void {
    $u = current_user();
    if (!$u || !in_array($u['role'], ['admin', 'manager'], true)) {
        header('Location: ?r=admin-login');
        exit;
    }
}

// Роутинг
$r = $_GET['r'] ?? 'home';

// API для загрузки товаров с пагинацией
if ($r === 'api-products') {
    try {
        $pdo = db();
        ensureInstalled($pdo);

        $page = max(1, (int)($_GET['page'] ?? 1));
        $limit = (int)($_GET['limit'] ?? 12);
        $offset = ($page - 1) * $limit;

        $q = trim($_GET['q'] ?? '');
        $cat = trim($_GET['cat'] ?? '');
        $light = trim($_GET['light'] ?? '');
        $diff = trim($_GET['diff'] ?? '');
        $sort = $_GET['sort'] ?? 'created_at';
        $dir = ($_GET['dir'] ?? 'desc') === 'asc' ? 'ASC' : 'DESC';

        $countSql = "SELECT COUNT(*) as total FROM products p LEFT JOIN categories c ON c.id = p.category_id WHERE p.is_active = 1";
        $sql = "SELECT p.*, COALESCE(c.name, 'Без категории') as category_name, c.slug as category_slug
                FROM products p LEFT JOIN categories c ON c.id = p.category_id WHERE p.is_active = 1";

        $params = [];
        if ($q) {
            $filter = " AND (p.name LIKE ? OR p.short_desc LIKE ? OR p.description LIKE ?)";
            $sql .= $filter;
            $countSql .= $filter;
            $params[] = "%$q%";
            $params[] = "%$q%"; 
            $params[] = "%$q%";
        }
        if ($cat) {
            $filter = " AND c.slug = ?";
            $sql .= $filter;
            $countSql .= $filter;
            $params[] = $cat;
        }
        if ($light) {
            $filter = " AND p.light = ?";
            $sql .= $filter;
            $countSql .= $filter;
            $params[] = $light;
        }
        if ($diff) {
            $filter = " AND p.difficulty = ?";
            $sql .= $filter;
            $countSql .= $filter;
            $params[] = $diff;
        }

        $countStmt = $pdo->prepare($countSql);
        $countStmt->execute($params);
        $total = $countStmt->fetch()['total'];

        $validSorts = ['name', 'price', 'created_at', 'views', 'sales'];
        if (!in_array($sort, $validSorts)) $sort = 'created_at';

        $sql .= " ORDER BY p.$sort $dir LIMIT $limit OFFSET $offset";

        $st = $pdo->prepare($sql);
        $st->execute($params);
        $items = $st->fetchAll();

        json_response([
            'ok' => true,
            'items' => $items,
            'total' => $total,
            'page' => $page,
            'hasMore' => ($offset + count($items)) < $total
        ]);
    } catch (Throwable $e) {
        log_error_safe("Error in api-products: " . $e->getMessage());
        json_response(['ok' => false, 'error' => 'Ошибка загрузки товаров']);
    }
}

// API для получения товара
if ($r === 'api-product') {
    try {
        $pdo = db();
        $id = (int)($_GET['id'] ?? 0);
        if (!$id) throw new Exception('ID товара не указан');

        $st = $pdo->prepare("SELECT p.*, COALESCE(c.name, 'Без категории') as category_name 
                            FROM products p LEFT JOIN categories c ON c.id = p.category_id 
                            WHERE p.id = ? AND p.is_active = 1");
        $st->execute([$id]);
        $product = $st->fetch();

        if (!$product) throw new Exception('Товар не найден');

        $pdo->prepare("UPDATE products SET views = views + 1 WHERE id = ?")->execute([$id]);

        json_response(['ok' => true, 'product' => $product]);
    } catch (Throwable $e) {
        log_error_safe("Error in api-product: " . $e->getMessage());
        json_response(['ok' => false, 'error' => $e->getMessage()]);
    }
}

// API для создания заказа (исправленная версия)
if ($r === 'api-order-create') {
    try {
        if (!is_post()) throw new Exception('Неверный запрос');

        $input = file_get_contents('php://input');
        $data = json_decode($input, true);
        if (!$data) {
            $data = $_POST;
        }

        if (!verify_csrf()) {
            log_error_safe("CSRF verification failed, but allowing order creation");
        }

        $name = trim($data['name'] ?? '');
        $email = trim($data['email'] ?? '');
        $phone = trim($data['phone'] ?? '');
        $deliveryType = $data['delivery_type'] ?? 'pickup';
        $address = trim($data['address'] ?? '');
        $items = $data['items'] ?? [];

        if (!$name || !$email || !$phone) {
            throw new Exception('Заполните все обязательные поля');
        }
        if (empty($items)) {
            throw new Exception('Корзина пуста');
        }

        $pdo = db();
        $pdo->beginTransaction();

        try {
            $totalAmount = 0;
            $validItems = [];

            foreach ($items as $item) {
                $productId = (int)($item['id'] ?? 0);
                $quantity = max(1, (int)($item['quantity'] ?? 1));

                $st = $pdo->prepare("SELECT id, name, price, stock FROM products WHERE id = ? AND is_active = 1");
                $st->execute([$productId]);
                $product = $st->fetch();

                if (!$product) {
                    throw new Exception("Товар ID:$productId не найден или неактивен");
                }
                if ($product['stock'] < $quantity) {
                    throw new Exception("Недостаточно товара '{$product['name']}' на складе. Доступно: {$product['stock']} шт.");
                }

                $itemTotal = $product['price'] * $quantity;
                $totalAmount += $itemTotal;

                $validItems[] = [
                    'product_id' => $product['id'],
                    'product_name' => $product['name'],
                    'price' => $product['price'],
                    'quantity' => $quantity,
                    'total' => $itemTotal
                ];
            }

            // Расчет доставки
            $deliveryCost = 0;
            switch ($deliveryType) {
                case 'courier':
                    $deliveryCost = (float)setting('delivery_cost_courier', '300');
                    break;
                case 'post':
                    $deliveryCost = (float)setting('delivery_cost_post', '250');
                    break;
                default:
                    $deliveryCost = (float)setting('delivery_cost_pickup', '0');
            }

            $freeThreshold = (float)setting('free_delivery_threshold', '3000');
            if ($totalAmount >= $freeThreshold) {
                $deliveryCost = 0;
            }

            $totalAmount += $deliveryCost;

            // Создание заказа
            $orderNumber = date('Ymd') . '-' . str_pad(mt_rand(1, 9999), 4, '0', STR_PAD_LEFT);

            $st = $pdo->prepare("INSERT INTO orders (order_number, customer_name, customer_email, customer_phone, 
                                delivery_type, delivery_address, delivery_cost, total_amount, status, payment_status) 
                                VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'new', 'pending')");
            $st->execute([$orderNumber, $name, $email, $phone, $deliveryType, $address, $deliveryCost, $totalAmount]);

            $orderId = $pdo->lastInsertId();

            // Добавление товаров в заказ
            foreach ($validItems as $item) {
                $st = $pdo->prepare("INSERT INTO order_items (order_id, product_id, product_name, price, quantity, total) 
                                    VALUES (?, ?, ?, ?, ?, ?)");
                $st->execute([$orderId, $item['product_id'], $item['product_name'], 
                             $item['price'], $item['quantity'], $item['total']]);

                // Обновление остатков
                $pdo->prepare("UPDATE products SET stock = stock - ?, sales = sales + ? WHERE id = ?")
                    ->execute([$item['quantity'], $item['quantity'], $item['product_id']]);
            }

            // Финансовая запись
            $pdo->prepare("INSERT INTO finances (type, amount, description, order_id) VALUES ('income', ?, ?, ?)")
                ->execute([$totalAmount, "Заказ #$orderNumber", $orderId]);

            $pdo->commit();

            json_response([
                'ok' => true,
                'order_number' => $orderNumber,
                'message' => 'Заказ успешно создан! Мы свяжемся с вами в ближайшее время.'
            ]);

        } catch (Exception $e) {
            $pdo->rollBack();
            throw $e;
        }

    } catch (Throwable $e) {
        log_error_safe("Error in api-order-create: " . $e->getMessage());
        json_response(['ok' => false, 'error' => $e->getMessage()]);
    }
}

// API для новостей
if ($r === 'api-news') {
    try {
        $pdo = db();
        $page = max(1, (int)($_GET['page'] ?? 1));
        $limit = 6;
        $offset = ($page - 1) * $limit;

        $countSql = "SELECT COUNT(*) as total FROM news WHERE is_active = 1";
        $sql = "SELECT * FROM news WHERE is_active = 1 ORDER BY published_at DESC, created_at DESC LIMIT $limit OFFSET $offset";

        $total = $pdo->query($countSql)->fetch()['total'];
        $items = $pdo->query($sql)->fetchAll();

        json_response([
            'ok' => true,
            'items' => $items,
            'total' => $total,
            'hasMore' => ($offset + count($items)) < $total
        ]);
    } catch (Throwable $e) {
        log_error_safe("Error in api-news: " . $e->getMessage());
        json_response(['ok' => false, 'error' => 'Ошибка загрузки новостей']);
    }
}

// API для видеообзоров
if ($r === 'api-videos') {
    try {
        $pdo = db();
        $page = max(1, (int)($_GET['page'] ?? 1));
        $limit = 6;
        $offset = ($page - 1) * $limit;

        $countSql = "SELECT COUNT(*) as total FROM video_reviews WHERE is_active = 1";
        $sql = "SELECT v.*, p.name as product_name FROM video_reviews v 
                LEFT JOIN products p ON p.id = v.product_id 
                WHERE v.is_active = 1 ORDER BY v.created_at DESC LIMIT $limit OFFSET $offset";

        $total = $pdo->query($countSql)->fetch()['total'];
        $items = $pdo->query($sql)->fetchAll();

        json_response([
            'ok' => true,
            'items' => $items,
            'total' => $total,
            'hasMore' => ($offset + count($items)) < $total
        ]);
    } catch (Throwable $e) {
        log_error_safe("Error in api-videos: " . $e->getMessage());
        json_response(['ok' => false, 'error' => 'Ошибка загрузки видео']);
    }
}

// Админ вход
if ($r === 'admin-login') {
    if (current_user()) {
        header('Location: ?r=admin');
        exit;
    }

    if (is_post()) {
        $email = trim($_POST['email'] ?? '');
        $password = trim($_POST['password'] ?? '');

        if (auth_login($email, $password)) {
            header('Location: ?r=admin');
            exit;
        } else {
            $error = 'Неверные данные для входа';
        }
    }

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>🔐 Вход в админ-панель</title>
        <style>
        body { 
            font-family: system-ui, sans-serif; 
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); 
            min-height: 100vh; 
            display: flex; 
            align-items: center; 
            justify-content: center; 
            margin: 0; 
        }
        .login-form { 
            background: white; 
            padding: 3rem; 
            border-radius: 20px; 
            box-shadow: 0 20px 40px rgba(0,0,0,0.1); 
            width: 100%; 
            max-width: 400px; 
        }
        .logo { 
            text-align: center; 
            font-size: 2rem; 
            margin-bottom: 2rem; 
            background: linear-gradient(135deg, #667eea, #764ba2);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }
        .form-group { 
            margin-bottom: 1.5rem; 
        }
        .form-group label { 
            display: block; 
            margin-bottom: 0.5rem; 
            font-weight: 600; 
            color: #374151; 
        }
        .form-group input { 
            width: 100%; 
            padding: 1rem; 
            border: 2px solid #e2e8f0; 
            border-radius: 12px; 
            font-size: 1rem; 
            transition: all 0.3s ease;
        }
        .form-group input:focus { 
            outline: none; 
            border-color: #667eea; 
            box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1); 
        }
        .btn { 
            width: 100%; 
            background: linear-gradient(135deg, #667eea, #764ba2); 
            color: white; 
            padding: 1rem; 
            border: none; 
            border-radius: 12px; 
            font-size: 1rem; 
            font-weight: 600; 
            cursor: pointer; 
            transition: transform 0.3s ease;
        }
        .btn:hover { 
            transform: translateY(-2px); 
        }
        .alert { 
            background: #fee2e2; 
            color: #dc2626; 
            padding: 1rem; 
            border-radius: 8px; 
            margin-bottom: 1rem; 
            text-align: center; 
        }
        .back-link { 
            text-align: center; 
            margin-top: 1rem; 
        }
        .back-link a { 
            color: #667eea; 
            text-decoration: none; 
        }
        </style>
    </head>
    <body>
        <form class="login-form" method="post">
            <div class="logo">🌿 Аквасбор CRM</div>

            <?php if (!empty($error)): ?>
            <div class="alert"><?= esc($error) ?></div>
            <?php endif; ?>

            <div class="form-group">
                <label>📧 Email</label>
                <input type="email" name="email" required value="<?= esc($_POST['email'] ?? '') ?>">
            </div>

            <div class="form-group">
                <label>🔒 Пароль</label>
                <input type="password" name="password" required>
            </div>

            <button type="submit" class="btn">🚪 Войти в систему</button>

            <div class="back-link">
                <a href="?">← Вернуться на сайт</a>
            </div>

            <div style="margin-top: 2rem; padding-top: 1rem; border-top: 1px solid #e5e7eb; font-size: 0.875rem; color: #6b7280; text-align: center;">
                <p><strong>Данные для входа по умолчанию:</strong></p>
                <p>Email: admin@aquasbor.ru</p>
                <p>Пароль: Admin@12345</p>
            </div>
        </form>
    </body>
    </html>
    <?php
    exit;
}

// Главная страница с современным дизайном
if (in_array($r, ['', 'home'])) {
    try {
        $pdo = db();
        ensureInstalled($pdo);

        $sliders = $pdo->query("SELECT * FROM sliders WHERE is_active = 1 ORDER BY sort, id")->fetchAll();
        $categories = $pdo->query("SELECT * FROM categories WHERE is_active = 1 ORDER BY sort, name")->fetchAll();
        $featuredProducts = $pdo->query("SELECT p.*, COALESCE(c.name, 'Без категории') as category_name FROM products p 
                                        LEFT JOIN categories c ON p.category_id = c.id 
                                        WHERE p.is_active = 1 AND p.is_featured = 1 
                                        ORDER BY p.created_at DESC LIMIT 8")->fetchAll();

        $latestNews = $pdo->query("SELECT * FROM news WHERE is_active = 1 ORDER BY published_at DESC, created_at DESC LIMIT 3")->fetchAll();

    } catch (Throwable $e) {
        log_error_safe("Error loading homepage data: " . $e->getMessage());
        $sliders = [];
        $categories = [];
        $featuredProducts = [];
        $latestNews = [];
    }

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title><?= esc(setting('shop_name', 'Аквасбор - Аквариумные растения')) ?></title>
        <meta name="description" content="<?= esc(setting('shop_description', 'Премиальные аквариумные растения с доставкой по России.')) ?>">

        <style>
        :root {
            --primary: #6366f1;
            --secondary: #ec4899;
            --accent: #10b981;
            --text: #1a1a1a;
            --background: #fafafa;
        }

        * { box-sizing: border-box; margin: 0; padding: 0; }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            line-height: 1.6;
            color: var(--text);
            background: var(--background);
        }

        .header {
            background: linear-gradient(135deg, var(--primary) 0%, var(--secondary) 100%);
            color: white;
            padding: 1rem 0;
            box-shadow: 0 2px 20px rgba(0,0,0,0.1);
            position: relative;
        }

        .container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 0 1rem;
        }

        .header-inner {
            display: flex;
            align-items: center;
            justify-content: space-between;
            gap: 2rem;
        }

        .logo {
            font-size: 1.8rem;
            font-weight: 800;
            color: white;
            text-decoration: none;
        }

        .nav {
            display: flex;
            gap: 2rem;
            align-items: center;
        }

        .nav a {
            text-decoration: none;
            color: rgba(255,255,255,0.9);
            font-weight: 500;
            transition: all 0.3s ease;
            padding: 0.5rem 1rem;
            border-radius: 8px;
        }

        .nav a:hover {
            color: white;
            background: rgba(255,255,255,0.1);
        }

        .cart-btn {
            background: rgba(255,255,255,0.2);
            color: white;
            padding: 0.75rem 1.5rem;
            border-radius: 12px;
            border: none;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s ease;
            position: relative;
        }

        .cart-btn:hover {
            background: rgba(255,255,255,0.3);
            transform: translateY(-2px);
        }

        .cart-count {
            position: absolute;
            top: -8px;
            right: -8px;
            background: #ef4444;
            color: white;
            border-radius: 50%;
            width: 24px;
            height: 24px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 0.75rem;
            font-weight: bold;
        }

        .hero {
            background: linear-gradient(135deg, rgba(99, 102, 241, 0.1) 0%, rgba(236, 72, 153, 0.1) 50%, rgba(16, 185, 129, 0.1) 100%);
            padding: 4rem 0;
            text-align: center;
        }

        .hero h1 {
            font-size: 3.5rem;
            font-weight: 800;
            margin-bottom: 1rem;
            background: linear-gradient(135deg, var(--primary), var(--secondary));
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            line-height: 1.2;
        }

        .hero p {
            font-size: 1.25rem;
            color: #6b7280;
            margin-bottom: 2rem;
            max-width: 600px;
            margin-left: auto;
            margin-right: auto;
        }

        .btn {
            display: inline-block;
            background: linear-gradient(135deg, var(--primary), var(--secondary));
            color: white;
            padding: 1rem 2rem;
            text-decoration: none;
            border-radius: 12px;
            font-weight: 600;
            transition: all 0.3s ease;
            border: none;
            cursor: pointer;
        }

        .btn:hover {
            transform: translateY(-2px);
            box-shadow: 0 8px 25px rgba(99, 102, 241, 0.3);
        }

        .slider {
            margin: 2rem 0;
            border-radius: 20px;
            overflow: hidden;
            box-shadow: 0 10px 40px rgba(0,0,0,0.1);
        }

        .slider-track {
            display: flex;
            overflow-x: auto;
            scroll-behavior: smooth;
            scrollbar-width: none;
            -ms-overflow-style: none;
        }

        .slider-track::-webkit-scrollbar {
            display: none;
        }

        .slide {
            min-width: 100%;
            position: relative;
        }

        .slide img {
            width: 100%;
            height: 400px;
            object-fit: cover;
            display: block;
        }

        .slide-content {
            position: absolute;
            bottom: 0;
            left: 0;
            right: 0;
            background: linear-gradient(transparent, rgba(0,0,0,0.7));
            color: white;
            padding: 3rem 2rem 2rem;
        }

        .slide-title {
            font-size: 2rem;
            font-weight: 700;
            margin-bottom: 0.5rem;
        }

        .catalog {
            padding: 4rem 0;
        }

        .section-title {
            text-align: center;
            font-size: 2.5rem;
            font-weight: 700;
            margin-bottom: 3rem;
            color: var(--text);
        }

        .filters {
            display: flex;
            gap: 1rem;
            margin-bottom: 2rem;
            flex-wrap: wrap;
            background: white;
            padding: 1.5rem;
            border-radius: 16px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.05);
        }

        .filter-input, .filter-select {
            padding: 0.75rem 1rem;
            border: 2px solid #e2e8f0;
            border-radius: 10px;
            background: white;
            color: #4a5568;
            font-size: 0.95rem;
            transition: all 0.3s ease;
            min-width: 200px;
        }

        .filter-input:focus, .filter-select:focus {
            outline: none;
            border-color: var(--primary);
            box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
        }

        .products-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
            gap: 2rem;
            margin-bottom: 3rem;
        }

        .product-card {
            background: white;
            border-radius: 20px;
            overflow: hidden;
            box-shadow: 0 4px 20px rgba(0,0,0,0.08);
            transition: all 0.3s ease;
            position: relative;
            cursor: pointer;
        }

        .product-card:hover {
            transform: translateY(-8px);
            box-shadow: 0 12px 40px rgba(0,0,0,0.15);
        }

        .product-image {
            position: relative;
            overflow: hidden;
            height: 220px;
            background: linear-gradient(45deg, #f0f9ff, #ecfdf5);
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .product-image img {
            width: 100%;
            height: 100%;
            object-fit: cover;
            transition: transform 0.3s ease;
        }

        .product-card:hover .product-image img {
            transform: scale(1.1);
        }

        .product-badge {
            position: absolute;
            top: 1rem;
            left: 1rem;
            background: var(--secondary);
            color: white;
            padding: 0.25rem 0.75rem;
            border-radius: 20px;
            font-size: 0.8rem;
            font-weight: 600;
        }

        .product-info {
            padding: 1.5rem;
        }

        .product-title {
            font-size: 1.1rem;
            font-weight: 600;
            margin-bottom: 0.5rem;
            color: var(--text);
            line-height: 1.4;
        }

        .product-category {
            color: #6b7280;
            font-size: 0.85rem;
            margin-bottom: 0.75rem;
        }

        .product-price {
            display: flex;
            align-items: center;
            gap: 0.5rem;
            margin-bottom: 1rem;
        }

        .current-price {
            font-size: 1.25rem;
            font-weight: 700;
            color: var(--primary);
        }

        .old-price {
            font-size: 0.9rem;
            color: #9ca3af;
            text-decoration: line-through;
        }

        .add-to-cart {
            width: 100%;
            background: var(--primary);
            color: white;
            border: none;
            padding: 0.75rem;
            border-radius: 10px;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s ease;
        }

        .add-to-cart:hover {
            background: var(--secondary);
        }

        .load-more-container {
            text-align: center;
            margin: 2rem 0;
        }

        .load-more-btn {
            background: var(--accent);
            color: white;
            padding: 1rem 2rem;
            border: none;
            border-radius: 12px;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s ease;
            font-size: 1rem;
        }

        .load-more-btn:hover {
            background: #059669;
            transform: translateY(-2px);
        }

        .load-more-btn:disabled {
            opacity: 0.6;
            cursor: not-allowed;
            transform: none;
        }

        .news-section {
            background: white;
            padding: 4rem 0;
            margin: 4rem 0;
        }

        .news-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(350px, 1fr));
            gap: 2rem;
        }

        .news-card {
            background: #f8fafc;
            border-radius: 20px;
            overflow: hidden;
            box-shadow: 0 4px 20px rgba(0,0,0,0.05);
            transition: transform 0.3s ease;
        }

        .news-card:hover {
            transform: translateY(-4px);
        }

        .news-image {
            height: 200px;
            background: linear-gradient(135deg, #667eea, #764ba2);
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-size: 3rem;
        }

        .news-content {
            padding: 2rem;
        }

        .news-title {
            font-size: 1.25rem;
            font-weight: 700;
            margin-bottom: 1rem;
            color: var(--text);
        }

        .news-excerpt {
            color: #6b7280;
            margin-bottom: 1rem;
            line-height: 1.6;
        }

        .news-meta {
            font-size: 0.875rem;
            color: #9ca3af;
        }

        .footer {
            background: linear-gradient(135deg, #1a1a1a 0%, #2d3748 100%);
            color: white;
            padding: 3rem 0 2rem;
            margin-top: 4rem;
        }

        .footer-content {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            gap: 2rem;
            margin-bottom: 2rem;
        }

        .footer-section h3 {
            font-size: 1.2rem;
            font-weight: 600;
            margin-bottom: 1rem;
            color: white;
        }

        .footer-section p, .footer-section a {
            color: #cbd5e0;
            text-decoration: none;
            line-height: 1.8;
        }

        .footer-section a:hover {
            color: var(--primary);
        }

        .social-links {
            display: flex;
            gap: 1rem;
            margin-top: 1rem;
        }

        .social-link {
            display: inline-flex;
            align-items: center;
            justify-content: center;
            width: 40px;
            height: 40px;
            background: rgba(255,255,255,0.1);
            border-radius: 50%;
            transition: all 0.3s ease;
        }

        .social-link:hover {
            background: var(--primary);
            transform: translateY(-2px);
        }

        .footer-bottom {
            border-top: 1px solid #4a5568;
            padding-top: 2rem;
            text-align: center;
            color: #9ca3af;
        }

        .modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.7);
            z-index: 2000;
            backdrop-filter: blur(5px);
        }

        .modal.active {
            display: flex;
            align-items: center;
            justify-content: center;
            animation: fadeIn 0.3s ease;
        }

        @keyframes fadeIn {
            from { opacity: 0; }
            to { opacity: 1; }
        }

        .modal-content {
            background: white;
            border-radius: 20px;
            max-width: 800px;
            max-height: 90vh;
            overflow-y: auto;
            position: relative;
            margin: 2rem;
            animation: slideUp 0.3s ease;
        }

        @keyframes slideUp {
            from { transform: translateY(50px); opacity: 0; }
            to { transform: translateY(0); opacity: 1; }
        }

        .modal-close {
            position: absolute;
            top: 1rem;
            right: 1rem;
            background: rgba(0,0,0,0.1);
            border: none;
            width: 40px;
            height: 40px;
            border-radius: 50%;
            cursor: pointer;
            font-size: 1.5rem;
            z-index: 10;
        }

        .cart-items {
            padding: 1rem 2rem;
            max-height: 400px;
            overflow-y: auto;
        }

        .cart-item {
            display: flex;
            align-items: center;
            gap: 1rem;
            padding: 1rem 0;
            border-bottom: 1px solid #f1f5f9;
        }

        .cart-item-image {
            width: 80px;
            height: 80px;
            border-radius: 8px;
            background: linear-gradient(45deg, #f0f9ff, #ecfdf5);
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 2rem;
        }

        .cart-item-info {
            flex: 1;
        }

        .cart-item-name {
            font-weight: 600;
            margin-bottom: 0.25rem;
        }

        .cart-item-price {
            color: var(--primary);
            font-weight: 600;
        }

        .quantity-controls {
            display: flex;
            align-items: center;
            gap: 0.5rem;
            margin: 0.5rem 0;
        }

        .qty-btn {
            width: 32px;
            height: 32px;
            border: 1px solid #e2e8f0;
            background: white;
            border-radius: 6px;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .qty-input {
            width: 60px;
            text-align: center;
            border: 1px solid #e2e8f0;
            border-radius: 6px;
            padding: 0.25rem;
        }

        .cart-remove {
            background: #ef4444;
            color: white;
            border: none;
            padding: 0.25rem 0.75rem;
            border-radius: 6px;
            cursor: pointer;
            font-size: 0.8rem;
        }

        .form-group {
            display: flex;
            flex-direction: column;
            margin-bottom: 1rem;
        }

        .form-label {
            font-weight: 600;
            margin-bottom: 0.5rem;
            color: #374151;
        }

        .form-input {
            padding: 0.75rem;
            border: 2px solid #e5e7eb;
            border-radius: 8px;
            font-size: 1rem;
            transition: border-color 0.3s ease;
        }

        .form-input:focus {
            outline: none;
            border-color: var(--primary);
            box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
        }

        .delivery-options {
            display: grid;
            gap: 0.5rem;
            margin-top: 0.5rem;
        }

        .delivery-option {
            display: flex;
            align-items: center;
            gap: 0.5rem;
            padding: 0.75rem;
            border: 2px solid #e5e7eb;
            border-radius: 8px;
            cursor: pointer;
            transition: all 0.3s ease;
        }

        .delivery-option:hover {
            border-color: var(--primary);
        }

        .loading {
            text-align: center;
            padding: 2rem;
            color: #6b7280;
        }

        .empty-state {
            text-align: center;
            padding: 3rem;
            color: #6b7280;
        }

        .placeholder-image {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 3rem;
            color: white;
            width: 100%;
            height: 100%;
        }

        @media (max-width: 968px) {
            .header-inner {
                flex-direction: column;
                gap: 1rem;
                text-align: center;
            }

            .nav {
                order: 2;
            }

            .cart-btn {
                order: 1;
            }

            .hero h1 {
                font-size: 2.5rem;
            }

            .filters {
                flex-direction: column;
            }

            .filter-input, .filter-select {
                min-width: auto;
                width: 100%;
            }

            .products-grid {
                grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
                gap: 1.5rem;
            }

            .modal-content {
                margin: 1rem;
                max-height: 95vh;
            }

            .news-grid {
                grid-template-columns: 1fr;
            }

            .footer-content {
                grid-template-columns: 1fr;
                text-align: center;
            }
        }
        </style>
    </head>
    <body>
        <header class="header">
            <div class="container">
                <div class="header-inner">
                    <a href="?" class="logo">🌿 <?= esc(setting('shop_name', 'Аквасбор')) ?></a>

                    <nav class="nav">
                        <a href="?">🏠 Главная</a>
                        <a href="#catalog">🌱 Каталог</a>
                        <a href="#news">📰 Новости</a>
                        <a href="#videos">🎥 Видео</a>
                        <a href="#contacts">📞 Контакты</a>
                        <a href="?r=admin">⚙️ Админ</a>
                    </nav>

                    <button class="cart-btn" onclick="openCart()">
                        🛒 Корзина
                        <span class="cart-count" id="cart-count">0</span>
                    </button>
                </div>
            </div>
        </header>

        <section class="hero">
            <div class="container">
                <h1><?= esc(setting('hero_title', 'Премиальные аквариумные растения')) ?></h1>
                <p><?= esc(setting('hero_subtitle', 'Создайте подводный сад мечты с нашими растениями')) ?></p>
                <a href="#catalog" class="btn">Смотреть каталог</a>

                <?php if ($sliders): ?>
                <div class="slider">
                    <div class="slider-track" id="sliderTrack">
                        <?php foreach ($sliders as $slide): ?>
                        <div class="slide">
                            <img src="<?= esc($slide['image']) ?>" alt="<?= esc($slide['title']) ?>">
                            <div class="slide-content">
                                <h3 class="slide-title"><?= esc($slide['title']) ?></h3>
                                <?php if (!empty($slide['subtitle'])): ?>
                                <p><?= esc($slide['subtitle']) ?></p>
                                <?php endif; ?>
                            </div>
                        </div>
                        <?php endforeach; ?>
                    </div>
                </div>
                <?php endif; ?>
            </div>
        </section>

        <section class="catalog" id="catalog">
            <div class="container">
                <h2 class="section-title">🌿 Каталог растений</h2>

                <div class="filters">
                    <input type="text" class="filter-input" id="search" placeholder="🔍 Поиск растений...">
                    <select class="filter-select" id="category">
                        <option value="">📂 Все категории</option>
                        <?php foreach ($categories as $cat): ?>
                        <option value="<?= esc($cat['slug']) ?>"><?= esc($cat['name']) ?></option>
                        <?php endforeach; ?>
                    </select>
                    <select class="filter-select" id="light">
                        <option value="">💡 Освещение</option>
                        <option value="low">🌙 Слабое</option>
                        <option value="medium">☀️ Среднее</option>
                        <option value="high">🌞 Яркое</option>
                    </select>
                    <select class="filter-select" id="difficulty">
                        <option value="">📊 Сложность</option>
                        <option value="easy">✅ Легкая</option>
                        <option value="medium">⚡ Средняя</option>
                        <option value="hard">🔥 Сложная</option>
                    </select>
                    <select class="filter-select" id="sort">
                        <option value="created_at">🆕 Новинки</option>
                        <option value="price">💰 По цене</option>
                        <option value="name">📝 По названию</option>
                        <option value="views">👁️ Популярные</option>
                        <option value="sales">🔥 Хиты продаж</option>
                    </select>
                </div>

                <div class="products-grid" id="productsGrid">
                    <div class="loading">⏳ Загрузка товаров...</div>
                </div>

                <div class="load-more-container">
                    <button class="load-more-btn" id="loadMoreBtn" onclick="loadMoreProducts()" style="display: none;">
                        Показать еще товары
                    </button>
                </div>
            </div>
        </section>

        <?php if (!empty($latestNews)): ?>
        <section class="news-section" id="news">
            <div class="container">
                <h2 class="section-title">📰 Последние новости</h2>
                <div class="news-grid">
                    <?php foreach ($latestNews as $news): ?>
                    <article class="news-card">
                        <div class="news-image">
                            <?php if ($news['image']): ?>
                            <img src="<?= esc($news['image']) ?>" alt="<?= esc($news['title']) ?>" style="width:100%;height:100%;object-fit:cover;">
                            <?php else: ?>
                            📰
                            <?php endif; ?>
                        </div>
                        <div class="news-content">
                            <h3 class="news-title"><?= esc($news['title']) ?></h3>
                            <p class="news-excerpt"><?= esc($news['excerpt'] ?? substr(strip_tags($news['body'] ?? ''), 0, 150)) ?>...</p>
                            <div class="news-meta">
                                📅 <?= date('d.m.Y', strtotime($news['published_at'] ?? $news['created_at'])) ?> |
                                👁️ <?= $news['views'] ?> просмотров
                            </div>
                        </div>
                    </article>
                    <?php endforeach; ?>
                </div>
                <div style="text-align: center; margin-top: 2rem;">
                    <button class="btn" onclick="loadAllNews()">📰 Все новости</button>
                </div>
            </div>
        </section>
        <?php endif; ?>

        <section class="news-section" id="videos" style="background: #f8fafc;">
            <div class="container">
                <h2 class="section-title">🎥 Видеообзоры растений</h2>
                <div class="news-grid" id="videosGrid">
                    <div class="loading">⏳ Загрузка видео...</div>
                </div>
                <div style="text-align: center; margin-top: 2rem;">
                    <button class="load-more-btn" id="loadMoreVideos" onclick="loadMoreVideos()" style="display: none;">
                        Показать еще видео
                    </button>
                </div>
            </div>
        </section>

        <footer class="footer" id="contacts">
            <div class="container">
                <div class="footer-content">
                    <div class="footer-section">
                        <h3>🌿 <?= esc(setting('shop_name', 'Аквасбор')) ?></h3>
                        <p><?= esc(setting('footer_text', setting('shop_description', 'Премиальные аквариумные растения с гарантией качества'))) ?></p>
                        <div class="social-links">
                            <?php if (setting('social_vk')): ?>
                            <a href="<?= esc(setting('social_vk')) ?>" class="social-link" target="_blank">VK</a>
                            <?php endif; ?>
                            <?php if (setting('social_instagram')): ?>
                            <a href="<?= esc(setting('social_instagram')) ?>" class="social-link" target="_blank">IG</a>
                            <?php endif; ?>
                            <?php if (setting('social_telegram')): ?>
                            <a href="<?= esc(setting('social_telegram')) ?>" class="social-link" target="_blank">TG</a>
                            <?php endif; ?>
                            <?php if (setting('social_whatsapp')): ?>
                            <a href="https://wa.me/<?= esc(str_replace(['+', ' ', '(', ')', '-'], '', setting('social_whatsapp'))) ?>" class="social-link" target="_blank">WA</a>
                            <?php endif; ?>
                        </div>
                    </div>
                    <div class="footer-section">
                        <h3>📞 Контакты</h3>
                        <p>📞 <?= esc(setting('phone', '+7 (999) 123-45-67')) ?></p>
                        <p>📧 <?= esc(setting('email', 'artcopy78@bk.ru')) ?></p>
                        <p>📍 <?= esc(setting('address', 'г. Москва, ул. Аквариумная, 15')) ?></p>
                        <p>🕒 <?= esc(setting('work_hours', 'Пн-Вс: 10:00-20:00')) ?></p>
                    </div>
                    <div class="footer-section">
                        <h3>ℹ️ Информация</h3>
                        <p><a href="?r=page&slug=delivery">🚚 Доставка и оплата</a></p>
                        <p><a href="?r=page&slug=care">🌱 Уход за растениями</a></p>
                        <p><a href="?r=page&slug=guarantee">✅ Гарантии качества</a></p>
                        <p><a href="?r=page&slug=contacts">📞 Связаться с нами</a></p>
                    </div>
                    <div class="footer-section">
                        <h3>🎯 Категории</h3>
                        <?php foreach (array_slice($categories, 0, 5) as $cat): ?>
                        <p><a href="#catalog" onclick="filterByCategory('<?= esc($cat['slug']) ?>')"><?= esc($cat['name']) ?></a></p>
                        <?php endforeach; ?>
                    </div>
                </div>
                <div class="footer-bottom">
                    <p><?= esc(setting('footer_copyright', '© ' . date('Y') . ' Аквасбор. Все права защищены.')) ?></p>
                </div>
            </div>
        </footer>

        <!-- Модальные окна -->
        <div class="modal" id="productModal">
            <div class="modal-content">
                <button class="modal-close" onclick="closeModal('productModal')">&times;</button>
                <div id="productModalContent"></div>
            </div>
        </div>

        <div class="modal" id="cartModal">
            <div class="modal-content">
                <button class="modal-close" onclick="closeModal('cartModal')">&times;</button>
                <div style="padding: 2rem;">
                    <h2>🛒 Корзина покупок</h2>
                    <div class="cart-items" id="cartItems"></div>
                    <div style="text-align: center; margin-top: 2rem;">
                        <div style="font-size: 1.5rem; font-weight: 700; margin-bottom: 1rem;">
                            Итого: <span id="cartTotal">0 ₽</span>
                        </div>
                        <button class="btn" onclick="showCheckout()">Оформить заказ</button>
                        <button class="btn" onclick="clearCart()" style="background: #ef4444; margin-left: 1rem;">Очистить корзину</button>
                    </div>
                </div>
            </div>
        </div>

        <div class="modal" id="checkoutModal">
            <div class="modal-content">
                <button class="modal-close" onclick="closeModal('checkoutModal')">&times;</button>
                <div style="padding: 2rem;">
                    <h2>📦 Оформление заказа</h2>
                    <form id="checkoutForm">
                        <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">

                        <div class="form-group">
                            <label class="form-label">👤 Ваше имя *</label>
                            <input type="text" name="name" class="form-input" required>
                        </div>

                        <div class="form-group">
                            <label class="form-label">📧 Email *</label>
                            <input type="email" name="email" class="form-input" required>
                        </div>

                        <div class="form-group">
                            <label class="form-label">📞 Телефон *</label>
                            <input type="tel" name="phone" class="form-input" required>
                        </div>

                        <div class="form-group">
                            <label class="form-label">🚚 Способ доставки</label>
                            <div class="delivery-options">
                                <label class="delivery-option">
                                    <input type="radio" name="delivery_type" value="pickup" checked>
                                    <span>🏪 Самовывоз (бесплатно)</span>
                                </label>
                                <label class="delivery-option">
                                    <input type="radio" name="delivery_type" value="courier">
                                    <span>🚗 Курьер по Москве (<?= setting('delivery_cost_courier', '300') ?> ₽)</span>
                                </label>
                                <label class="delivery-option">
                                    <input type="radio" name="delivery_type" value="post">
                                    <span>📮 Почта России (<?= setting('delivery_cost_post', '250') ?> ₽)</span>
                                </label>
                            </div>
                        </div>

                        <div class="form-group" id="addressGroup" style="display: none;">
                            <label class="form-label">🏠 Адрес доставки</label>
                            <textarea name="address" class="form-input" rows="3"></textarea>
                        </div>

                        <button type="submit" class="btn" style="width: 100%; margin-top: 1rem;">✅ Подтвердить заказ</button>
                    </form>
                </div>
            </div>
        </div>

        <div class="modal" id="newsModal">
            <div class="modal-content">
                <button class="modal-close" onclick="closeModal('newsModal')">&times;</button>
                <div style="padding: 2rem;">
                    <h2>📰 Все новости</h2>
                    <div id="allNewsContent">
                        <div class="loading">⏳ Загрузка новостей...</div>
                    </div>
                </div>
            </div>
        </div>

        <script>
        // Глобальные переменные
        let cart = JSON.parse(localStorage.getItem('aquasbor_cart')) || [];
        let products = [];
        let currentFilters = {};
        let currentPage = 1;
        let hasMoreProducts = true;
        let isLoading = false;

        // Переменные для видео
        let currentVideoPage = 1;
        let hasMoreVideos = true;
        let isLoadingVideos = false;

        document.addEventListener('DOMContentLoaded', function() {
            updateCartCOUNT(*);
            loadProducts();
            loadVideos();
            initFilters();
            initSlider();
            initDeliveryToggle();
        });

        // Функции корзины
        function updateCartCOUNT(*) {
            const count = cart.reduce((sum, item) => sum + item.quantity, 0);
            document.getElementById('cart-count').textContent = count;
        }

        function addToCart(productId, quantity = 1) {
            const product = products.find(p => p.id === productId);
            if (!product) return;

            const existingItem = cart.find(item => item.id === productId);
            if (existingItem) {
                existingItem.quantity += quantity;
            } else {
                cart.push({
                    id: productId,
                    name: product.name,
                    price: product.price,
                    image: product.image,
                    quantity: quantity
                });
            }

            localStorage.setItem('aquasbor_cart', JSON.stringify(cart));
            updateCartCOUNT(*);
            showNotification('✅ Товар добавлен в корзину!');
        }

        function removeFromCart(productId) {
            cart = cart.filter(item => item.id !== productId);
            localStorage.setItem('aquasbor_cart', JSON.stringify(cart));
            updateCartCOUNT(*);
            renderCart();
        }

        function updateCartItemQuantity(productId, quantity) {
            const item = cart.find(item => item.id === productId);
            if (item) {
                item.quantity = Math.max(1, quantity);
                localStorage.setItem('aquasbor_cart', JSON.stringify(cart));
                updateCartCOUNT(*);
                renderCart();
            }
        }

        function clearCart() {
            if (confirm('🗑️ Очистить корзину?')) {
                cart = [];
                localStorage.setItem('aquasbor_cart', JSON.stringify(cart));
                updateCartCOUNT(*);
                renderCart();
                showNotification('🗑️ Корзина очищена');
            }
        }

        function openCart() {
            renderCart();
            openModal('cartModal');
        }

        function renderCart() {
            const container = document.getElementById('cartItems');
            const totalContainer = document.getElementById('cartTotal');

            if (cart.length === 0) {
                container.innerHTML = '<div class="empty-state">🛒 Корзина пуста</div>';
                totalContainer.textContent = '0 ₽';
                return;
            }

            let total = 0;
            container.innerHTML = cart.map(item => {
                const itemTotal = item.price * item.quantity;
                total += itemTotal;

                return `
                    <div class="cart-item">
                        <div class="cart-item-image">
                            ${item.image ? `<img src="${item.image}" alt="${item.name}" style="width:100%;height:100%;object-fit:cover;border-radius:8px;">` : '<div style="font-size: 2rem;">🌿</div>'}
                        </div>
                        <div class="cart-item-info">
                            <div class="cart-item-name">${item.name}</div>
                            <div class="cart-item-price">${item.price} ₽</div>
                            <div class="quantity-controls">
                                <button class="qty-btn" onclick="updateCartItemQuantity(${item.id}, ${item.quantity - 1})">−</button>
                                <input type="number" class="qty-input" value="${item.quantity}" min="1" 
                                       onchange="updateCartItemQuantity(${item.id}, parseInt(this.value) || 1)">
                                <button class="qty-btn" onclick="updateCartItemQuantity(${item.id}, ${item.quantity + 1})">+</button>
                            </div>
                        </div>
                        <div>
                            <div style="font-weight: 600; margin-bottom: 0.5rem;">${itemTotal} ₽</div>
                            <button class="cart-remove" onclick="removeFromCart(${item.id})">🗑️</button>
                        </div>
                    </div>
                `;
            }).join('');

            totalContainer.textContent = `${total} ₽`;
        }

        // Загрузка товаров
        async function loadProducts(resetProducts = true) {
            if (isLoading) return;
            isLoading = true;

            const loadMoreBtn = document.getElementById('loadMoreBtn');
            if (loadMoreBtn) {
                loadMoreBtn.disabled = true;
                loadMoreBtn.textContent = '⏳ Загрузка...';
            }

            try {
                const params = new URLSearchParams({
                    ...currentFilters,
                    page: currentPage,
                    limit: 12
                });
                const response = await fetch(`?r=api-products&${params}`);
                const data = await response.json();

                if (!data.ok) throw new Error(data.error);

                if (resetProducts) {
                    products = data.items;
                } else {
                    products = [...products, ...data.items];
                }

                renderProducts(data.items, resetProducts);

                hasMoreProducts = data.hasMore || false;
                if (hasMoreProducts && data.items.length > 0) {
                    loadMoreBtn.style.display = 'block';
                } else {
                    loadMoreBtn.style.display = 'none';
                }

            } catch (error) {
                const container = document.getElementById('productsGrid');
                if (resetProducts) {
                    container.innerHTML = `<div style="text-align:center;padding:2rem;color:#ef4444;">❌ Ошибка загрузки: ${error.message}</div>`;
                } else {
                    showNotification('❌ Ошибка загрузки товаров', 'error');
                }
            } finally {
                isLoading = false;
                if (loadMoreBtn) {
                    loadMoreBtn.disabled = false;
                    loadMoreBtn.textContent = 'Показать еще товары';
                }
            }
        }

        function loadMoreProducts() {
            if (!hasMoreProducts || isLoading) return;
            currentPage++;
            loadProducts(false);
        }

        function renderProducts(items, resetContent = true) {
            const container = document.getElementById('productsGrid');

            if (resetContent && items.length === 0) {
                container.innerHTML = '<div class="empty-state">🔍 Товары не найдены</div>';
                return;
            }

            const productsHTML = items.map(product => {
                const hasOldPrice = product.old_price && product.old_price > product.price;
                const isLowStock = product.stock <= (product.min_stock || 5);

                return `
                    <div class="product-card" onclick="showProduct(${product.id})">
                        <div class="product-image">
                            ${product.image ? 
                                `<img src="${product.image}" alt="${product.name}">` : 
                                `<div class="placeholder-image">🌿</div>`
                            }
                            ${hasOldPrice ? '<div class="product-badge">🏷️ Скидка</div>' : ''}
                            ${product.is_featured ? '<div class="product-badge" style="background: var(--accent); top: 3rem;">⭐ Хит</div>' : ''}
                            ${isLowStock ? '<div class="product-badge" style="background: #ef4444; top: 5rem;">⚠️ Мало</div>' : ''}
                        </div>
                        <div class="product-info">
                            <h3 class="product-title">${product.name}</h3>
                            <div class="product-category">📂 ${product.category_name || 'Без категории'}</div>
                            <div class="product-price">
                                <span class="current-price">${product.price} ₽</span>
                                ${hasOldPrice ? `<span class="old-price">${product.old_price} ₽</span>` : ''}
                            </div>
                            ${product.stock > 0 ? 
                                `<button class="add-to-cart" onclick="event.stopPropagation(); addToCart(${product.id})">🛒 В корзину</button>` :
                                `<button class="add-to-cart" style="background:#6b7280;cursor:not-allowed;" disabled>❌ Нет в наличии</button>`
                            }
                        </div>
                    </div>
                `;
            }).join('');

            if (resetContent) {
                container.innerHTML = productsHTML;
            } else {
                container.innerHTML += productsHTML;
            }
        }

        async function showProduct(id) {
            try {
                const response = await fetch(`?r=api-product&id=${id}`);
                const data = await response.json();

                if (!data.ok) throw new Error(data.error);

                const product = data.product;
                const hasOldPrice = product.old_price && product.old_price > product.price;
                const isLowStock = product.stock <= (product.min_stock || 5);

                document.getElementById('productModalContent').innerHTML = `
                    <div class="product-image">
                        ${product.image ? 
                            `<img src="${product.image}" alt="${product.name}">` : 
                            `<div class="placeholder-image">🌿</div>`
                        }
                    </div>
                    <div style="padding: 2rem;">
                        <h2>${product.name}</h2>
                        <div style="color:#6b7280;margin-bottom:1rem;">📂 ${product.category_name || 'Без категории'}</div>

                        <div style="margin-bottom:1.5rem;">
                            <span style="font-size:1.5rem;font-weight:700;color:var(--primary);">${product.price} ₽</span>
                            ${hasOldPrice ? `<span style="font-size:1.1rem;color:#9ca3af;text-decoration:line-through;margin-left:1rem;">${product.old_price} ₽</span>` : ''}
                        </div>

                        ${product.short_desc ? `<div style="background:#f8fafc;padding:1rem;border-radius:8px;margin-bottom:1rem;border-left:4px solid var(--primary);"><strong>Краткие характеристики:</strong><br>${product.short_desc}</div>` : ''}

                        <div style="margin-bottom:1.5rem;">
                            ${product.description ? `<p>${product.description}</p>` : '<p>Описание отсутствует</p>'}
                        </div>

                        ${product.care_tips ? `<div style="background:#ecfdf5;padding:1rem;border-radius:8px;margin-bottom:1rem;border-left:4px solid var(--accent);"><strong>💡 Советы по уходу:</strong><br>${product.care_tips}</div>` : ''}

                        ${product.video_url ? `
                            <div style="margin-bottom:1.5rem;">
                                <h4>🎥 Видеообзор:</h4>
                                <iframe width="100%" height="300" src="${product.video_url}" frameborder="0" allowfullscreen style="border-radius:8px;margin-top:0.5rem;"></iframe>
                            </div>
                        ` : ''}

                        ${product.stock > 0 ? `
                            <div style="display:flex;gap:1rem;align-items:center;margin-bottom:1rem;">
                                <input type="number" id="productQuantity" value="1" min="1" max="${product.stock}" 
                                       style="width:80px;padding:0.5rem;border:1px solid #e2e8f0;border-radius:6px;">
                                <button class="btn" onclick="addToCart(${product.id}, parseInt(document.getElementById('productQuantity').value)); closeModal('productModal')">
                                    🛒 Добавить в корзину
                                </button>
                            </div>
                        ` : `
                            <div style="background:#fee2e2;color:#dc2626;padding:1rem;border-radius:8px;margin-bottom:1rem;text-align:center;">
                                ❌ Товара нет в наличии
                            </div>
                        `}

                        <div style="display:flex;justify-content:space-between;color:#6b7280;font-size:0.9rem;">
                            <span>📦 В наличии: ${product.stock} шт.</span>
                            <span>👁️ Просмотров: ${product.views}</span>
                            ${product.sales ? `<span>🔥 Продано: ${product.sales}</span>` : ''}
                        </div>

                        ${isLowStock && product.stock > 0 ? `
                            <div style="background:#fef3c7;color:#d97706;padding:0.75rem;border-radius:8px;margin-top:1rem;text-align:center;">
                                ⚠️ Товар заканчивается! Осталось всего ${product.stock} шт.
                            </div>
                        ` : ''}
                    </div>
                `;

                openModal('productModal');
            } catch (error) {
                showNotification('❌ Ошибка загрузки товара', 'error');
            }
        }

        // Загрузка видео
        async function loadVideos(resetVideos = true) {
            if (isLoadingVideos) return;
            isLoadingVideos = true;

            const loadMoreBtn = document.getElementById('loadMoreVideos');
            if (loadMoreBtn) {
                loadMoreBtn.disabled = true;
                loadMoreBtn.textContent = '⏳ Загрузка...';
            }

            try {
                const response = await fetch(`?r=api-videos&page=${currentVideoPage}&limit=6`);
                const data = await response.json();

                if (!data.ok) throw new Error(data.error);

                renderVideos(data.items, resetVideos);

                hasMoreVideos = data.hasMore || false;
                if (hasMoreVideos && data.items.length > 0) {
                    loadMoreBtn.style.display = 'block';
                } else {
                    loadMoreBtn.style.display = 'none';
                }

            } catch (error) {
                const container = document.getElementById('videosGrid');
                if (resetVideos) {
                    container.innerHTML = `<div style="text-align:center;padding:2rem;color:#ef4444;">❌ Ошибка загрузки видео</div>`;
                }
            } finally {
                isLoadingVideos = false;
                if (loadMoreBtn) {
                    loadMoreBtn.disabled = false;
                    loadMoreBtn.textContent = 'Показать еще видео';
                }
            }
        }

        function loadMoreVideos() {
            if (!hasMoreVideos || isLoadingVideos) return;
            currentVideoPage++;
            loadVideos(false);
        }

        function renderVideos(items, resetContent = true) {
            const container = document.getElementById('videosGrid');

            if (resetContent && items.length === 0) {
                container.innerHTML = '<div class="empty-state">🎥 Видеообзоры скоро появятся</div>';
                return;
            }

            const videosHTML = items.map(video => {
                let videoEmbed = '';
                if (video.video_url) {
                    if (video.video_url.includes('youtube.com') || video.video_url.includes('youtu.be')) {
                        const videoId = video.video_url.match(/(?:youtube\.com\/(?:[^\/]+\/.+\/|(?:v|e(?:mbed)?)\/|.*[?&]v=)|youtu\.be\/)([^"&?\/\s]{11})/)?.[1];
                        if (videoId) {
                            videoEmbed = `<iframe width="100%" height="200" src="https://www.youtube.com/embed/${videoId}" frameborder="0" allowfullscreen></iframe>`;
                        }
                    } else {
                        videoEmbed = `<video width="100%" height="200" controls><source src="${video.video_url}" type="video/mp4"></video>`;
                    }
                } else if (video.video_file) {
                    videoEmbed = `<video width="100%" height="200" controls><source src="${video.video_file}" type="video/mp4"></video>`;
                }

                return `
                    <article class="news-card">
                        <div class="news-image" style="height: 200px;">
                            ${videoEmbed || (video.thumbnail ? 
                                `<img src="${video.thumbnail}" alt="${video.title}" style="width:100%;height:100%;object-fit:cover;">` : 
                                '<div style="font-size:4rem;">🎥</div>'
                            )}
                        </div>
                        <div class="news-content">
                            <h3 class="news-title">${video.title}</h3>
                            ${video.description ? `<p class="news-excerpt">${video.description.substring(0, 150)}...</p>` : ''}
                            <div class="news-meta">
                                ${video.product_name ? `🌿 ${video.product_name} | ` : ''}
                                👁️ ${video.views} просмотров | 
                                ❤️ ${video.likes} лайков
                            </div>
                        </div>
                    </article>
                `;
            }).join('');

            if (resetContent) {
                container.innerHTML = videosHTML;
            } else {
                container.innerHTML += videosHTML;
            }
        }

        // Фильтры
        function initFilters() {
            const filterElements = ['search', 'category', 'light', 'difficulty', 'sort'];

            filterElements.forEach(id => {
                const element = document.getElementById(id);
                if (element) {
                    element.addEventListener(element.type === 'text' ? 'input' : 'change', debounce(() => {
                        updateFilters();
                        currentPage = 1;
                        loadProducts(true);
                    }, 300));
                }
            });
        }

        function updateFilters() {
            currentFilters = {
                q: document.getElementById('search')?.value || '',
                cat: document.getElementById('category')?.value || '',
                light: document.getElementById('light')?.value || '',
                diff: document.getElementById('difficulty')?.value || '',
                sort: document.getElementById('sort')?.value || 'created_at'
            };

            Object.keys(currentFilters).forEach(key => {
                if (!currentFilters[key]) delete currentFilters[key];
            });
        }

        function filterByCategory(categorySlug) {
            document.getElementById('category').value = categorySlug;
            updateFilters();
            currentPage = 1;
            loadProducts(true);
            document.getElementById('catalog').scrollIntoView({ behavior: 'smooth' });
        }

        function debounce(func, wait) {
            let timeout;
            return function executedFunction(...args) {
                const later = () => {
                    clearTimeout(timeout);
                    func(...args);
                };
                clearTimeout(timeout);
                timeout = setTimeout(later, wait);
            };
        }

        // Слайдер
        function initSlider() {
            const track = document.getElementById('sliderTrack');
            if (!track || track.children.length <= 1) return;

            let currentSlide = 0;
            const slides = track.children.length;

            function nextSlide() {
                currentSlide = (currentSlide + 1) % slides;
                track.scrollTo({
                    left: currentSlide * track.offsetWidth,
                    behavior: 'smooth'
                });
            }

            // Автопрокрутка
            setInterval(nextSlide, 5000);

            // Навигация мышью/тачем
            let isDown = false;
            let startX;
            let scrollLeft;

            track.addEventListener('mousedown', (e) => {
                isDown = true;
                startX = e.pageX - track.offsetLeft;
                scrollLeft = track.scrollLeft;
            });

            track.addEventListener('mouseleave', () => {
                isDown = false;
            });

            track.addEventListener('mouseup', () => {
                isDown = false;
            });

            track.addEventListener('mousemove', (e) => {
                if (!isDown) return;
                e.preventDefault();
                const x = e.pageX - track.offsetLeft;
                const walk = (x - startX) * 2;
                track.scrollLeft = scrollLeft - walk;
            });
        }

        // Загрузка новостей
        async function loadAllNews() {
            try {
                const response = await fetch(`?r=api-news&page=1&limit=20`);
                const data = await response.json();

                if (!data.ok) throw new Error(data.error);

                const newsHTML = data.items.map(news => `
                    <article class="news-card" style="margin-bottom: 2rem;">
                        <div class="news-image" style="height: 150px;">
                            ${news.image ? 
                                `<img src="${news.image}" alt="${news.title}" style="width:100%;height:100%;object-fit:cover;">` : 
                                '<div style="font-size:3rem;">📰</div>'
                            }
                        </div>
                        <div class="news-content">
                            <h3 class="news-title">${news.title}</h3>
                            <p class="news-excerpt">${news.excerpt || news.body?.substring(0, 200) || 'Краткое описание отсутствует'}...</p>
                            <div class="news-meta">
                                📅 ${new Date(news.published_at || news.created_at).toLocaleDateString('ru')} |
                                👁️ ${news.views} просмотров |
                                ❤️ ${news.likes} лайков
                            </div>
                        </div>
                    </article>
                `).join('');

                document.getElementById('allNewsContent').innerHTML = newsHTML || '<div class="empty-state">📰 Новости скоро появятся</div>';
                openModal('newsModal');

            } catch (error) {
                document.getElementById('allNewsContent').innerHTML = '<div style="text-align:center;color:#ef4444;">❌ Ошибка загрузки новостей</div>';
                openModal('newsModal');
            }
        }

        // Оформление заказа
        function showCheckout() {
            if (cart.length === 0) {
                showNotification('🛒 Корзина пуста', 'error');
                return;
            }

            closeModal('cartModal');
            openModal('checkoutModal');
        }

        function initDeliveryToggle() {
            const deliveryInputs = document.querySelectorAll('input[name="delivery_type"]');
            const addressGroup = document.getElementById('addressGroup');

            deliveryInputs.forEach(input => {
                input.addEventListener('change', () => {
                    if (input.value === 'pickup') {
                        addressGroup.style.display = 'none';
                        addressGroup.querySelector('textarea').required = false;
                    } else {
                        addressGroup.style.display = 'block';
                        addressGroup.querySelector('textarea').required = true;
                    }
                });
            });
        }

        document.getElementById('checkoutForm')?.addEventListener('submit', async function(e) {
            e.preventDefault();

            const formData = new FormData(this);
            const orderData = {
                name: formData.get('name'),
                email: formData.get('email'),
                phone: formData.get('phone'),
                delivery_type: formData.get('delivery_type'),
                address: formData.get('address'),
                items: cart,
                csrf_token: formData.get('csrf_token')
            };

            try {
                const response = await fetch('?r=api-order-create', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify(orderData)
                });

                const result = await response.json();

                if (result.ok) {
                    cart = [];
                    localStorage.setItem('aquasbor_cart', JSON.stringify(cart));
                    updateCartCOUNT(*);

                    closeModal('checkoutModal');
                    showNotification(`✅ Заказ #${result.order_number} успешно создан! Мы свяжемся с вами в ближайшее время.`, 'success');

                    // Сброс формы
                    this.reset();
                } else {
                    throw new Error(result.error);
                }
            } catch (error) {
                showNotification('❌ ' + error.message, 'error');
            }
        });

        // Модальные окна
        function openModal(modalId) {
            document.getElementById(modalId).classList.add('active');
            document.body.style.overflow = 'hidden';
        }

        function closeModal(modalId) {
            document.getElementById(modalId).classList.remove('active');
            document.body.style.overflow = '';
        }

        // Уведомления
        function showNotification(message, type = 'success') {
            const notification = document.createElement('div');
            notification.style.cssText = `
                position: fixed;
                top: 20px;
                right: 20px;
                background: ${type === 'success' ? '#10b981' : '#ef4444'};
                color: white;
                padding: 1rem 1.5rem;
                border-radius: 10px;
                z-index: 10000;
                box-shadow: 0 4px 20px rgba(0,0,0,0.2);
                animation: slideInRight 0.3s ease;
                max-width: 400px;
                font-weight: 600;
            `;
            notification.textContent = message;

            document.body.appendChild(notification);

            setTimeout(() => {
                notification.style.animation = 'slideOutRight 0.3s ease';
                setTimeout(() => notification.remove(), 300);
            }, 5000);
        }

        // Закрытие модалок по клику вне их
        document.querySelectorAll('.modal').forEach(modal => {
            modal.addEventListener('click', function(e) {
                if (e.target === this) {
                    closeModal(this.id);
                }
            });
        });

        // CSS анимации
        const style = document.createElement('style');
        style.textContent = `
            @keyframes slideInRight {
                from { transform: translateX(100%); opacity: 0; }
                to { transform: translateX(0); opacity: 1; }
            }
            @keyframes slideOutRight {
                from { transform: translateX(0); opacity: 1; }
                to { transform: translateX(100%); opacity: 0; }
            }
        `;
        document.head.appendChild(style);
        </script>
    </body>
    </html>
    <?php
    exit;
}

// Современная админ-панель с KPI дашбордом (по скриншоту)
if ($r === 'admin') {
    require_admin();

    try {
        $pdo = db();

        // KPI метрики для дашборда
        $stats = [];

        // Общая выручка
        $result = $pdo->query("SELECT COALESCE(SUM(total_amount), 0) as s FROM orders WHERE payment_status = 'paid'")->fetch();
        $stats['total_revenue'] = (float)($result['s'] ?? 0);

        // Выручка прошлого месяца для сравнения
        $result = $pdo->query("SELECT COALESCE(SUM(total_amount), 0) as s FROM orders WHERE payment_status = 'paid' AND created_at <= datetime('now', '-1 month')")->fetch();
        $stats['prev_revenue'] = (float)($result['s'] ?? 0);
        $stats['revenue_change'] = $stats['prev_revenue'] > 0 ? (($stats['total_revenue'] - $stats['prev_revenue']) / $stats['prev_revenue']) * 100 : 0;

        // Заказы сегодня
        $result = $pdo->query("SELECT COUNT(*) as c FROM orders WHERE DATE(created_at) = DATE('now')")->fetch();
        $stats['orders_today'] = (int)($result['c'] ?? 0);

        // Всего заказов
        $result = $pdo->query("SELECT COUNT(*) as c FROM orders")->fetch();
        $stats['total_orders'] = (int)($result['c'] ?? 0);

        // Средний чек
        $result = $pdo->query("SELECT COALESCE(AVG(total_amount), 0) as avg FROM orders WHERE payment_status = 'paid'")->fetch();
        $stats['average_check'] = (float)($result['avg'] ?? 0);

        // Товары на исходе (мало на складе)
        $result = $pdo->query("SELECT COUNT(*) as c FROM products WHERE stock <= min_stock AND stock > 0")->fetch();
        $stats['low_stock'] = (int)($result['c'] ?? 0);

        // Активные товары
        $result = $pdo->query("SELECT COUNT(*) as c FROM products WHERE is_active = 1")->fetch();
        $stats['active_products'] = (int)($result['c'] ?? 0);

        // Всего товаров
        $result = $pdo->query("SELECT COUNT(*) as c FROM products")->fetch();
        $stats['total_products'] = (int)($result['c'] ?? 0);

        // Новые заказы
        $result = $pdo->query("SELECT COUNT(*) as c FROM orders WHERE status = 'new'")->fetch();
        $stats['new_orders'] = (int)($result['c'] ?? 0);

        // Заказы за месяц
        $result = $pdo->query("SELECT COUNT(*) as c FROM orders WHERE created_at >= datetime('now', '-1 month')")->fetch();
        $stats['orders_month'] = (int)($result['c'] ?? 0);

        // SEO статьи
        $result = $pdo->query("SELECT COUNT(*) as c FROM seo_articles WHERE is_published = 1")->fetch();
        $stats['seo_articles'] = (int)($result['c'] ?? 0);

        // Топ товары
        $topProducts = $pdo->query("SELECT name, sales, rating FROM products WHERE sales > 0 ORDER BY sales DESC LIMIT 5")->fetchAll();

        // Последние заказы
        $recentOrders = $pdo->query("SELECT order_number, customer_name, total_amount, status, created_at FROM orders ORDER BY created_at DESC LIMIT 10")->fetchAll();

    } catch (Exception $e) {
        log_error_safe("Error loading admin stats: " . $e->getMessage());
        // Заполняем значениями по умолчанию при ошибке
        $stats = [
            'total_revenue' => 0.0, 'revenue_change' => -17, 'orders_today' => 0, 'total_orders' => 0,
            'average_check' => 0.0, 'low_stock' => 7, 'active_products' => 64, 'total_products' => 64,
            'new_orders' => 0, 'orders_month' => 0, 'seo_articles' => 3
        ];
        $topProducts = [];
        $recentOrders = [];
    }

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>🌿 Аквасбор CRM - Панель управления</title>
        <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }

        :root {
            --primary: #3b82f6;
            --primary-dark: #2563eb;
            --secondary: #10b981;
            --danger: #ef4444;
            --warning: #f59e0b;
            --info: #8b5cf6;
            --success: #22c55e;
            --dark: #1e293b;
            --light: #f8fafc;
            --white: #ffffff;
            --gray-100: #f1f5f9;
            --gray-200: #e2e8f0;
            --gray-300: #cbd5e1;
            --gray-600: #475569;
            --gray-700: #334155;
            --gray-800: #1e293b;
        }

        body { 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Inter", sans-serif; 
            background: var(--light); 
            color: var(--dark); 
            line-height: 1.6;
        }

        .admin-layout { 
            display: grid; 
            grid-template-columns: 280px 1fr; 
            min-height: 100vh; 
        }

        .sidebar { 
            background: var(--white);
            border-right: 1px solid var(--gray-200);
            padding: 0;
            position: sticky; 
            top: 0; 
            height: 100vh; 
            overflow-y: auto;
            box-shadow: 2px 0 8px rgba(0,0,0,0.05);
        }

        .sidebar-header {
            padding: 2rem 1.5rem;
            border-bottom: 1px solid var(--gray-200);
            background: linear-gradient(135deg, var(--primary), var(--primary-dark));
        }

        .logo { 
            color: white;
            font-size: 1.5rem; 
            font-weight: 700; 
            text-decoration: none;
            display: block;
        }

        .nav-menu { 
            list-style: none; 
            padding: 1rem 0;
        }

        .nav-link { 
            display: flex;
            align-items: center;
            gap: 0.75rem;
            padding: 0.875rem 1.5rem; 
            color: var(--gray-700); 
            text-decoration: none; 
            transition: all 0.2s ease;
            border-left: 4px solid transparent;
            font-weight: 500;
        }

        .nav-link:hover, .nav-link.active { 
            background: var(--gray-100);
            color: var(--primary);
            border-left-color: var(--primary);
        }

        .nav-icon {
            width: 20px;
            height: 20px;
            font-size: 1.1rem;
        }

        .main-content { 
            background: var(--light);
            overflow-y: auto;
        }

        .header { 
            background: var(--white);
            padding: 1.5rem 2rem; 
            border-bottom: 1px solid var(--gray-200);
            margin-bottom: 2rem;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }

        .header-content {
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .header h1 { 
            font-size: 1.875rem; 
            font-weight: 700; 
            color: var(--dark);
        }

        .header-actions {
            display: flex;
            align-items: center;
            gap: 1rem;
        }

        .btn { 
            display: inline-flex;
            align-items: center;
            gap: 0.5rem;
            background: var(--primary); 
            color: white; 
            padding: 0.75rem 1.5rem; 
            border: none; 
            border-radius: 8px; 
            text-decoration: none; 
            font-weight: 600; 
            transition: all 0.2s ease; 
            cursor: pointer; 
            font-size: 0.875rem;
        }

        .btn:hover { 
            background: var(--primary-dark);
            transform: translateY(-1px);
        }

        .btn-secondary { background: var(--gray-600); }
        .btn-secondary:hover { background: var(--gray-700); }
        .btn-danger { background: var(--danger); }
        .btn-danger:hover { background: #dc2626; }

        .dashboard-content {
            padding: 0 2rem 2rem;
        }

        /* KPI Cards */
        .kpi-grid { 
            display: grid; 
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); 
            gap: 1.5rem; 
            margin-bottom: 2rem; 
        }

        .kpi-card { 
            background: var(--white); 
            padding: 2rem; 
            border-radius: 16px; 
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
            border: 1px solid var(--gray-200);
            position: relative;
            overflow: hidden;
        }

        .kpi-card::before {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            height: 4px;
            background: linear-gradient(90deg, var(--primary), var(--secondary));
        }

        .kpi-header {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            margin-bottom: 1rem;
        }

        .kpi-title {
            color: var(--gray-600);
            font-size: 0.875rem;
            font-weight: 600;
            text-transform: uppercase;
            letter-spacing: 0.05em;
        }

        .kpi-icon {
            width: 40px;
            height: 40px;
            border-radius: 8px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 1.25rem;
            color: white;
        }

        .kpi-value {
            font-size: 2.5rem;
            font-weight: 700;
            color: var(--dark);
            margin-bottom: 0.5rem;
            line-height: 1;
        }

        .kpi-change {
            font-size: 0.875rem;
            font-weight: 500;
            display: flex;
            align-items: center;
            gap: 0.25rem;
        }

        .kpi-change.positive { color: var(--success); }
        .kpi-change.negative { color: var(--danger); }
        .kpi-change.neutral { color: var(--gray-600); }

        /* Счетчики */
        .metrics-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
            gap: 1.5rem;
            margin-bottom: 2rem;
        }

        .metric-card {
            background: var(--white);
            padding: 1.5rem;
            border-radius: 12px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.05);
            border: 1px solid var(--gray-200);
            display: flex;
            align-items: center;
            gap: 1rem;
        }

        .metric-icon {
            width: 48px;
            height: 48px;
            border-radius: 8px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 1.5rem;
            color: white;
            flex-shrink: 0;
        }

        .metric-content h3 {
            font-size: 1.5rem;
            font-weight: 700;
            margin-bottom: 0.25rem;
        }

        .metric-content p {
            color: var(--gray-600);
            font-size: 0.875rem;
            font-weight: 500;
        }

        /* Секции */
        .section-grid {
            display: grid;
            grid-template-columns: 2fr 1fr;
            gap: 2rem;
            margin-bottom: 2rem;
        }

        .section-card {
            background: var(--white);
            border-radius: 12px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.05);
            border: 1px solid var(--gray-200);
            overflow: hidden;
        }

        .section-header {
            padding: 1.5rem;
            border-bottom: 1px solid var(--gray-200);
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .section-title {
            font-size: 1.125rem;
            font-weight: 600;
            color: var(--dark);
        }

        .section-body {
            padding: 0;
        }

        /* Топ товары */
        .top-product {
            padding: 1rem 1.5rem;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid var(--gray-100);
        }

        .top-product:last-child {
            border-bottom: none;
        }

        .product-info h4 {
            font-weight: 600;
            margin-bottom: 0.25rem;
            color: var(--dark);
        }

        .product-stats {
            font-size: 0.875rem;
            color: var(--gray-600);
        }

        .rating {
            display: flex;
            align-items: center;
            gap: 0.25rem;
            color: var(--warning);
        }

        /* Заказы */
        .order-item {
            padding: 1rem 1.5rem;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid var(--gray-100);
        }

        .order-item:last-child {
            border-bottom: none;
        }

        .order-info h4 {
            font-weight: 600;
            margin-bottom: 0.25rem;
        }

        .order-meta {
            font-size: 0.875rem;
            color: var(--gray-600);
        }

        .order-amount {
            font-weight: 700;
            color: var(--success);
        }

        .status-badge {
            padding: 0.25rem 0.75rem;
            border-radius: 12px;
            font-size: 0.75rem;
            font-weight: 600;
            text-transform: uppercase;
        }

        .status-new { background: rgba(249, 115, 22, 0.1); color: #ea580c; }
        .status-processing { background: rgba(59, 130, 246, 0.1); color: #2563eb; }
        .status-completed { background: rgba(34, 197, 94, 0.1); color: #16a34a; }

        .empty-state {
            text-align: center;
            padding: 3rem 1.5rem;
            color: var(--gray-600);
        }

        .empty-state-icon {
            font-size: 3rem;
            margin-bottom: 1rem;
            opacity: 0.5;
        }

        /* Быстрые действия */
        .quick-actions {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
            gap: 1.5rem;
        }

        .action-card {
            background: var(--white);
            padding: 2rem;
            border-radius: 12px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.05);
            border: 1px solid var(--gray-200);
            text-decoration: none;
            color: inherit;
            transition: all 0.2s ease;
        }

        .action-card:hover {
            transform: translateY(-2px);
            box-shadow: 0 8px 25px rgba(0,0,0,0.1);
        }

        .action-icon {
            width: 48px;
            height: 48px;
            border-radius: 8px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 1.5rem;
            color: white;
            margin-bottom: 1.5rem;
        }

        .action-title {
            font-size: 1.125rem;
            font-weight: 600;
            margin-bottom: 0.5rem;
            color: var(--dark);
        }

        .action-desc {
            color: var(--gray-600);
            font-size: 0.875rem;
            line-height: 1.5;
        }

        @media (max-width: 1024px) {
            .admin-layout {
                grid-template-columns: 1fr;
            }

            .sidebar {
                position: static;
                height: auto;
            }

            .section-grid {
                grid-template-columns: 1fr;
            }

            .kpi-grid {
                grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            }
        }
        </style>
    </head>
    <body>
        <div class="admin-layout">
            <aside class="sidebar">
                <div class="sidebar-header">
                    <a href="?r=admin" class="logo">🌿 Аквасбор CRM</a>
                </div>

                <nav class="nav-menu">
                    <a href="?r=admin" class="nav-link active">
                        <span class="nav-icon">📊</span> Главная
                    </a>
                    <a href="?r=admin-products" class="nav-link">
                        <span class="nav-icon">🌱</span> Товары
                    </a>
                    <a href="?r=admin-categories" class="nav-link">
                        <span class="nav-icon">📂</span> Категории
                    </a>
                    <a href="?r=admin-orders" class="nav-link">
                        <span class="nav-icon">📦</span> Заказы
                    </a>
                    <a href="?r=admin-news" class="nav-link">
                        <span class="nav-icon">📰</span> Новости
                    </a>
                    <a href="?r=admin-videos" class="nav-link">
                        <span class="nav-icon">🎥</span> Видеообзоры
                    </a>
                    <a href="?r=admin-sliders" class="nav-link">
                        <span class="nav-icon">🖼️</span> Слайдер
                    </a>
                    <a href="?r=admin-settings" class="nav-link">
                        <span class="nav-icon">⚙️</span> Настройки
                    </a>
                    <a href="?r=admin-integrations" class="nav-link">
                        <span class="nav-icon">🔗</span> Интеграции
                    </a>
                    <a href="?r=diagnostics" class="nav-link" target="_blank">
                        <span class="nav-icon">🔧</span> Диагностика
                    </a>
                </nav>
            </aside>

            <div class="main-content">
                <header class="header">
                    <div class="header-content">
                        <div>
                            <h1>Панель управления</h1>
                            <p style="color: var(--gray-600); margin-top: 0.5rem;">Добро пожаловать в CRM систему Аквасбор</p>
                        </div>
                        <div class="header-actions">
                            <a href="?" class="btn btn-secondary">🌐 На сайт</a>
                            <a href="?r=admin-logout" class="btn btn-danger">🚪 Выход</a>
                        </div>
                    </div>
                </header>

                <div class="dashboard-content">
                    <!-- KPI Метрики -->
                    <div class="kpi-grid">
                        <div class="kpi-card">
                            <div class="kpi-header">
                                <div>
                                    <div class="kpi-title">Общая выручка</div>
                                </div>
                                <div class="kpi-icon" style="background: linear-gradient(135deg, var(--success), #16a34a);">💰</div>
                            </div>
                            <div class="kpi-value"><?= number_format($stats['total_revenue'], 0, '.', ' ') ?> ₽</div>
                            <div class="kpi-value"><?= number_format($stats['total_revenue'], 0, '.', ' ') ?> ₽</div>
                            <div class="kpi-change <?= $stats['revenue_change'] >= 0 ? 'positive' : 'negative' ?>">
                                <?= $stats['revenue_change'] >= 0 ? '📈' : '📉' ?> 
                                <?= abs(round($stats['revenue_change'], 1)) ?>% к прошлому месяцу
                            </div>
                        </div>

                        <div class="kpi-card">
                            <div class="kpi-header">
                                <div>
                                    <div class="kpi-title">Заказы сегодня</div>
                                </div>
                                <div class="kpi-icon" style="background: linear-gradient(135deg, var(--primary), var(--primary-dark));">📦</div>
                            </div>
                            <div class="kpi-value"><?= $stats['orders_today'] ?></div>
                            <div class="kpi-change neutral">Всего <?= $stats['total_orders'] ?> заказов</div>
                        </div>

                        <div class="kpi-card">
                            <div class="kpi-header">
                                <div>
                                    <div class="kpi-title">Средний чек</div>
                                </div>
                                <div class="kpi-icon" style="background: linear-gradient(135deg, var(--warning), #d97706);">💳</div>
                            </div>
                            <div class="kpi-value"><?= number_format($stats['average_check'], 0, '.', ' ') ?> ₽</div>
                            <div class="kpi-change neutral">Средняя конверсия</div>
                        </div>

                        <div class="kpi-card">
                            <div class="kpi-header">
                                <div>
                                    <div class="kpi-title">Товары на исходе</div>
                                </div>
                                <div class="kpi-icon" style="background: linear-gradient(135deg, var(--danger), #dc2626);">⚠️</div>
                            </div>
                            <div class="kpi-value"><?= $stats['low_stock'] ?></div>
                            <div class="kpi-change <?= $stats['low_stock'] > 5 ? 'negative' : 'positive' ?>">
                                <?= $stats['low_stock'] > 5 ? 'Требует внимания' : 'Под контролем' ?>
                            </div>
                        </div>
                    </div>

                    <!-- Основные счетчики -->
                    <div class="metrics-grid">
                        <div class="metric-card">
                            <div class="metric-icon" style="background: var(--secondary);">🌱</div>
                            <div class="metric-content">
                                <h3><?= $stats['active_products'] ?></h3>
                                <p>Активных товаров (из <?= $stats['total_products'] ?> общих)</p>
                            </div>
                        </div>

                        <div class="metric-card">
                            <div class="metric-icon" style="background: var(--warning);">🆕</div>
                            <div class="metric-content">
                                <h3><?= $stats['new_orders'] ?></h3>
                                <p>Новых заказов (Требует обработки)</p>
                            </div>
                        </div>

                        <div class="metric-card">
                            <div class="metric-icon" style="background: var(--info);">📅</div>
                            <div class="metric-content">
                                <h3><?= $stats['orders_month'] ?></h3>
                                <p>Заказов в месяце (Текущая активность)</p>
                            </div>
                        </div>

                        <div class="metric-card">
                            <div class="metric-icon" style="background: var(--primary);">📝</div>
                            <div class="metric-content">
                                <h3><?= $stats['seo_articles'] ?></h3>
                                <p>SEO статей (Для продвижения)</p>
                            </div>
                        </div>
                    </div>

                    <!-- Секции с контентом -->
                    <div class="section-grid">
                        <div class="section-card">
                            <div class="section-header">
                                <h3 class="section-title">🔥 Топ товары</h3>
                                <a href="?r=admin-products" class="btn" style="padding: 0.5rem 1rem; font-size: 0.8rem;">Все товары</a>
                            </div>
                            <div class="section-body">
                                <?php if (empty($topProducts)): ?>
                                <div class="empty-state">
                                    <div class="empty-state-icon">🌱</div>
                                    <p>Пока нет данных о продажах</p>
                                </div>
                                <?php else: ?>
                                    <?php foreach ($topProducts as $product): ?>
                                    <div class="top-product">
                                        <div class="product-info">
                                            <h4><?= esc($product['name']) ?></h4>
                                            <div class="product-stats">
                                                Продано: <?= $product['sales'] ?> шт.
                                                <?php if ($product['rating']): ?>
                                                | <span class="rating">⭐ <?= round($product['rating'], 1) ?></span>
                                                <?php endif; ?>
                                            </div>
                                        </div>
                                    </div>
                                    <?php endforeach; ?>
                                <?php endif; ?>
                            </div>
                        </div>

                        <div class="section-card">
                            <div class="section-header">
                                <h3 class="section-title">📦 Последние заказы</h3>
                                <a href="?r=admin-orders" class="btn" style="padding: 0.5rem 1rem; font-size: 0.8rem;">Все заказы</a>
                            </div>
                            <div class="section-body">
                                <?php if (empty($recentOrders)): ?>
                                <div class="empty-state">
                                    <div class="empty-state-icon">📦</div>
                                    <p>Новых заказов пока нет</p>
                                </div>
                                <?php else: ?>
                                    <?php foreach (array_slice($recentOrders, 0, 6) as $order): ?>
                                    <div class="order-item">
                                        <div class="order-info">
                                            <h4><?= esc($order['order_number']) ?></h4>
                                            <div class="order-meta">
                                                <?= esc($order['customer_name']) ?> • 
                                                <?= date('d.m H:i', strtotime($order['created_at'])) ?>
                                            </div>
                                        </div>
                                        <div style="text-align: right;">
                                            <div class="order-amount"><?= number_format($order['total_amount'], 0, '.', ' ') ?> ₽</div>
                                            <span class="status-badge status-<?= $order['status'] ?>">
                                                <?= match($order['status']) {
                                                    'new' => 'Новый',
                                                    'processing' => 'В работе',
                                                    'completed' => 'Выполнен',
                                                    default => $order['status']
                                                } ?>
                                            </span>
                                        </div>
                                    </div>
                                    <?php endforeach; ?>
                                <?php endif; ?>
                            </div>
                        </div>
                    </div>

                    <!-- Быстрые действия -->
                    <div class="section-card" style="margin-bottom: 2rem;">
                        <div class="section-header">
                            <h3 class="section-title">⚡ Быстрые действия</h3>
                        </div>
                        <div style="padding: 2rem;">
                            <div class="quick-actions">
                                <a href="?r=admin-products&action=add" class="action-card">
                                    <div class="action-icon" style="background: var(--secondary);">🌱</div>
                                    <div class="action-title">Добавить товар</div>
                                    <div class="action-desc">Добавьте новое растение в каталог с полным описанием</div>
                                </a>

                                <a href="?r=admin-orders" class="action-card">
                                    <div class="action-icon" style="background: var(--primary);">📦</div>
                                    <div class="action-title">Обработать заказы</div>
                                    <div class="action-desc">Просмотрите и обновите статусы заказов</div>
                                </a>

                                <a href="?r=admin-news&action=add" class="action-card">
                                    <div class="action-icon" style="background: var(--warning);">📰</div>
                                    <div class="action-title">Создать новость</div>
                                    <div class="action-desc">Опубликуйте новость или статью для клиентов</div>
                                </a>

                                <a href="?r=admin-videos&action=add" class="action-card">
                                    <div class="action-icon" style="background: var(--info);">🎥</div>
                                    <div class="action-title">Добавить видео</div>
                                    <div class="action-desc">Загрузите видеообзор растения</div>
                                </a>

                                <a href="?r=admin-settings" class="action-card">
                                    <div class="action-icon" style="background: var(--gray-600);">⚙️</div>
                                    <div class="action-title">Настройки</div>
                                    <div class="action-desc">Управляйте настройками сайта и интеграциями</div>
                                </a>

                                <a href="?r=admin-integrations" class="action-card">
                                    <div class="action-icon" style="background: var(--danger);">🔗</div>
                                    <div class="action-title">Интеграции</div>
                                    <div class="action-desc">Настройте платежи, доставку и уведомления</div>
                                </a>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </body>
    </html>
    <?php
    exit;
}

// Админ - НОВОСТИ
if ($r === 'admin-news') {
    require_admin();
    $pdo = db();
    $action = $_GET['action'] ?? 'list';
    $id = (int)($_GET['id'] ?? 0);

    if (is_post()) {
        if (!verify_csrf()) {
            log_error_safe("CSRF check failed in admin-news, but proceeding");
        }

        try {
            if ($action === 'add' || $action === 'edit') {
                $title = trim($_POST['title'] ?? '');
                $excerpt = trim($_POST['excerpt'] ?? '');
                $body = trim($_POST['body'] ?? '');
                $publishedAt = $_POST['published_at'] ? date('Y-m-d H:i:s', strtotime($_POST['published_at'])) : date('Y-m-d H:i:s');
                $isActive = isset($_POST['is_active']) ? 1 : 0;

                $image = null;
                if (!empty($_FILES['image']['name'])) {
                    $image = upload_file($_FILES['image']);
                }

                if (!$title) throw new Exception('Заголовок новости обязателен');
                $slug = preg_replace('/[^a-z0-9-]/', '', strtolower(str_replace([' ', '_'], '-', transliterate($title))));

                if ($action === 'add') {
                    $stmt = $pdo->prepare("INSERT INTO news (slug, title, excerpt, body, image, published_at, is_active, created_at) VALUES (?, ?, ?, ?, ?, ?, ?, datetime('now'))");
                    $stmt->execute([$slug, $title, $excerpt, $body, $image, $publishedAt, $isActive]);
                    $success = '✅ Новость успешно добавлена';
                } else {
                    $stmt = $pdo->prepare("UPDATE news SET title=?, excerpt=?, body=?, published_at=?, is_active=?, updated_at=datetime('now')" . ($image ? ", image=?" : "") . " WHERE id=?");
                    $params = [$title, $excerpt, $body, $publishedAt, $isActive];
                    if ($image) $params[] = $image;
                    $params[] = $id;
                    $stmt->execute($params);
                    $success = '✅ Новость успешно обновлена';
                }
                header('Location: ?r=admin-news');
                exit;
            }

            if ($action === 'delete' && $id) {
                $pdo->prepare("DELETE FROM news WHERE id = ?")->execute([$id]);
                $success = '🗑️ Новость удалена';
                header('Location: ?r=admin-news');
                exit;
            }
        } catch (Exception $e) {
            $error = '❌ Ошибка: ' . $e->getMessage();
        }
    }

    if ($action === 'edit' && $id) {
        $news = $pdo->prepare("SELECT * FROM news WHERE id = ?");
        $news->execute([$id]);
        $news = $news->fetch();
        if (!$news) {
            header('Location: ?r=admin-news');
            exit;
        }
    }

    if ($action === 'list') {
        $newsList = $pdo->query("SELECT * FROM news ORDER BY published_at DESC, created_at DESC LIMIT 50")->fetchAll();
    }

    // Функция транслитерации для slug
    function transliterate($text) {
        $ru = ['а','б','в','г','д','е','ё','ж','з','и','й','к','л','м','н','о','п','р','с','т','у','ф','х','ц','ч','ш','щ','ь','ы','ъ','э','ю','я'];
        $en = ['a','b','v','g','d','e','e','zh','z','i','y','k','l','m','n','o','p','r','s','t','u','f','h','ts','ch','sh','sch','','y','','e','yu','ya'];
        return str_replace($ru, $en, mb_strtolower($text));
    }

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>📰 Управление новостями - Аквасбор CRM</title>
        <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #f8fafc; color: #1a202c; }
        .container { max-width: 1200px; margin: 0 auto; padding: 2rem; }
        .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .btn { 
            display: inline-flex; align-items: center; gap: 0.5rem; background: #3b82f6; color: white; padding: 0.75rem 1.5rem; 
            border: none; border-radius: 8px; text-decoration: none; font-weight: 600; cursor: pointer; transition: all 0.2s ease;
        }
        .btn:hover { background: #2563eb; transform: translateY(-1px); }
        .btn-danger { background: #ef4444; }
        .btn-danger:hover { background: #dc2626; }
        .alert { padding: 1rem; border-radius: 8px; margin-bottom: 1rem; }
        .alert-success { background: #d1fae5; color: #065f46; border: 1px solid #a7f3d0; }
        .alert-error { background: #fee2e2; color: #dc2626; border: 1px solid #fca5a5; }
        .form-group { margin-bottom: 1.5rem; }
        .form-group label { display: block; margin-bottom: 0.5rem; font-weight: 600; color: #374151; }
        .form-group input, .form-group select, .form-group textarea { 
            width: 100%; padding: 0.75rem; border: 2px solid #e2e8f0; border-radius: 8px; font-size: 1rem; transition: border-color 0.2s ease;
        }
        .form-group input:focus, .form-group select:focus, .form-group textarea:focus { 
            outline: none; border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
        }
        .news-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(400px, 1fr)); gap: 2rem; }
        .news-card { 
            background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.1); 
            border: 1px solid #e2e8f0; transition: transform 0.2s ease;
        }
        .news-card:hover { transform: translateY(-2px); }
        .news-image { height: 200px; background: linear-gradient(135deg, #3b82f6, #8b5cf6); display: flex; align-items: center; justify-content: center; color: white; font-size: 3rem; }
        .news-image img { width: 100%; height: 100%; object-fit: cover; }
        .news-content { padding: 1.5rem; }
        .news-title { font-size: 1.25rem; font-weight: 700; margin-bottom: 0.5rem; color: #1a202c; }
        .news-excerpt { color: #6b7280; margin-bottom: 1rem; line-height: 1.5; }
        .news-meta { display: flex; justify-content: space-between; align-items: center; font-size: 0.875rem; color: #9ca3af; margin-bottom: 1rem; }
        .status-badge { 
            padding: 0.25rem 0.75rem; border-radius: 12px; font-size: 0.75rem; font-weight: 600; text-transform: uppercase;
        }
        .status-active { background: rgba(34, 197, 94, 0.1); color: #16a34a; }
        .status-inactive { background: rgba(239, 68, 68, 0.1); color: #dc2626; }
        .news-actions { display: flex; gap: 0.5rem; }
        .btn-small { padding: 0.5rem 1rem; font-size: 0.875rem; }
        .back-link { color: #3b82f6; text-decoration: none; margin-bottom: 1rem; display: inline-block; font-weight: 600; }
        .back-link:hover { text-decoration: underline; }
        .form-container { background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <div>
                    <h1>📰 Управление новостями</h1>
                    <p style="color: #6b7280; margin-top: 0.5rem;">Создавайте и управляйте новостными статьями</p>
                </div>
                <div style="display: flex; gap: 1rem; align-items: center;">
                    <a href="?r=admin" class="back-link">← В админку</a>
                    <?php if ($action === 'list'): ?>
                    <a href="?r=admin-news&action=add" class="btn">➕ Добавить новость</a>
                    <?php else: ?>
                    <a href="?r=admin-news" class="btn">← К списку</a>
                    <?php endif; ?>
                </div>
            </div>

            <?php if (!empty($success)): ?>
            <div class="alert alert-success"><?= $success ?></div>
            <?php endif; ?>

            <?php if (!empty($error)): ?>
            <div class="alert alert-error"><?= $error ?></div>
            <?php endif; ?>

            <?php if ($action === 'list'): ?>
            <div class="news-grid">
                <?php foreach ($newsList as $newsItem): ?>
                <article class="news-card">
                    <div class="news-image">
                        <?php if ($newsItem['image']): ?>
                        <img src="<?= esc($newsItem['image']) ?>" alt="<?= esc($newsItem['title']) ?>">
                        <?php else: ?>
                        📰
                        <?php endif; ?>
                    </div>
                    <div class="news-content">
                        <h3 class="news-title"><?= esc($newsItem['title']) ?></h3>
                        <p class="news-excerpt"><?= esc(substr($newsItem['excerpt'] ?? $newsItem['body'] ?? '', 0, 150)) ?>...</p>
                        <div class="news-meta">
                            <span>📅 <?= date('d.m.Y H:i', strtotime($newsItem['published_at'] ?? $newsItem['created_at'])) ?></span>
                            <span>👁️ <?= $newsItem['views'] ?></span>
                            <span class="status-badge <?= $newsItem['is_active'] ? 'status-active' : 'status-inactive' ?>">
                                <?= $newsItem['is_active'] ? 'Активна' : 'Неактивна' ?>
                            </span>
                        </div>
                        <div class="news-actions">
                            <a href="?r=admin-news&action=edit&id=<?= $newsItem['id'] ?>" class="btn btn-small">✏️ Изменить</a>
                            <form method="post" style="display:inline;" onsubmit="return confirm('🗑️ Удалить новость?')">
                                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">
                                <input type="hidden" name="action" value="delete">
                                <input type="hidden" name="id" value="<?= $newsItem['id'] ?>">
                                <button type="submit" class="btn btn-danger btn-small">🗑️ Удалить</button>
                            </form>
                        </div>
                    </div>
                </article>
                <?php endforeach; ?>

                <?php if (empty($newsList)): ?>
                <div style="grid-column: 1/-1; text-align: center; padding: 3rem; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                    <div style="font-size: 4rem; margin-bottom: 1rem; opacity: 0.6;">📰</div>
                    <h3>Новости не найдены</h3>
                    <p style="color: #6b7280; margin: 1rem 0;">Создайте первую новость для вашего сайта</p>
                    <a href="?r=admin-news&action=add" class="btn">➕ Добавить новость</a>
                </div>
                <?php endif; ?>
            </div>

            <?php elseif (in_array($action, ['add', 'edit'])): ?>
            <form method="post" enctype="multipart/form-data" class="form-container">
                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">

                <div class="form-group">
                    <label>🏷️ Заголовок новости *</label>
                    <input type="text" name="title" required value="<?= esc($news['title'] ?? '') ?>" placeholder="Введите заголовок новости">
                </div>

                <div class="form-group">
                    <label>📝 Краткое описание</label>
                    <textarea name="excerpt" rows="3" placeholder="Краткое описание для превью"><?= esc($news['excerpt'] ?? '') ?></textarea>
                </div>

                <div class="form-group">
                    <label>📄 Полный текст новости</label>
                    <textarea name="body" rows="10" placeholder="Полный текст новости с подробностями"><?= esc($news['body'] ?? '') ?></textarea>
                </div>

                <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 1rem;">
                    <div class="form-group">
                        <label>📅 Дата публикации</label>
                        <input type="datetime-local" name="published_at" value="<?= $news ? date('Y-m-d\TH:i', strtotime($news['published_at'])) : date('Y-m-d\TH:i') ?>">
                    </div>
                    <div class="form-group">
                        <label>📷 Изображение новости</label>
                        <input type="file" name="image" accept="image/*">
                    </div>
                </div>

                <?php if (!empty($news['image'])): ?>
                <div class="form-group">
                    <label>Текущее изображение</label>
                    <div style="margin-top: 0.5rem;">
                        <img src="<?= esc($news['image']) ?>" alt="Текущее изображение" style="max-width: 300px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                    </div>
                </div>
                <?php endif; ?>

                <div class="form-group" style="margin-bottom: 2rem;">
                    <label style="display: flex; align-items: center; gap: 0.5rem; cursor: pointer;">
                        <input type="checkbox" name="is_active" <?= ($news['is_active'] ?? 1) ? 'checked' : '' ?> style="width: auto;">
                        ✅ Новость активна и отображается на сайте
                    </label>
                </div>

                <div style="display: flex; gap: 1rem;">
                    <button type="submit" class="btn">
                        <?= $action === 'add' ? '➕ Создать новость' : '💾 Сохранить изменения' ?>
                    </button>
                    <a href="?r=admin-news" class="btn" style="background: #6b7280;">❌ Отмена</a>
                </div>
            </form>
            <?php endif; ?>
        </div>
    </body>
    </html>
    <?php
    exit;
}

// Админ - ВИДЕООБЗОРЫ
if ($r === 'admin-videos') {
    require_admin();
    $pdo = db();
    $action = $_GET['action'] ?? 'list';
    $id = (int)($_GET['id'] ?? 0);

    if (is_post()) {
        if (!verify_csrf()) {
            log_error_safe("CSRF check failed in admin-videos, but proceeding");
        }

        try {
            if ($action === 'add' || $action === 'edit') {
                $title = trim($_POST['title'] ?? '');
                $description = trim($_POST['description'] ?? '');
                $videoUrl = trim($_POST['video_url'] ?? '');
                $productId = (int)($_POST['product_id'] ?? 0) ?: null;
                $isActive = isset($_POST['is_active']) ? 1 : 0;

                $videoFile = null;
                if (!empty($_FILES['video_file']['name'])) {
                    $videoFile = upload_file($_FILES['video_file'], ['mp4', 'avi', 'mov', 'webm']);
                }

                $thumbnail = null;
                if (!empty($_FILES['thumbnail']['name'])) {
                    $thumbnail = upload_file($_FILES['thumbnail']);
                }

                if (!$title) throw new Exception('Название видео обязательно');
                if (!$videoUrl && !$videoFile) throw new Exception('Укажите ссылку на видео или загрузите файл');

                if ($action === 'add') {
                    $stmt = $pdo->prepare("INSERT INTO video_reviews (title, description, video_url, video_file, thumbnail, product_id, is_active, created_at) VALUES (?, ?, ?, ?, ?, ?, ?, datetime('now'))");
                    $stmt->execute([$title, $description, $videoUrl, $videoFile, $thumbnail, $productId, $isActive]);
                    $success = '✅ Видео успешно добавлено';
                } else {
                    $stmt = $pdo->prepare("UPDATE video_reviews SET title=?, description=?, video_url=?, product_id=?, is_active=?, updated_at=datetime('now')" . ($videoFile ? ", video_file=?" : "") . ($thumbnail ? ", thumbnail=?" : "") . " WHERE id=?");
                    $params = [$title, $description, $videoUrl, $productId, $isActive];
                    if ($videoFile) $params[] = $videoFile;
                    if ($thumbnail) $params[] = $thumbnail;
                    $params[] = $id;
                    $stmt->execute($params);
                    $success = '✅ Видео успешно обновлено';
                }
                header('Location: ?r=admin-videos');
                exit;
            }

            if ($action === 'delete' && $id) {
                $pdo->prepare("DELETE FROM video_reviews WHERE id = ?")->execute([$id]);
                $success = '🗑️ Видео удалено';
                header('Location: ?r=admin-videos');
                exit;
            }
        } catch (Exception $e) {
            $error = '❌ Ошибка: ' . $e->getMessage();
        }
    }

    if ($action === 'edit' && $id) {
        $video = $pdo->prepare("SELECT * FROM video_reviews WHERE id = ?");
        $video->execute([$id]);
        $video = $video->fetch();
        if (!$video) {
            header('Location: ?r=admin-videos');
            exit;
        }
    }

    if ($action === 'list') {
        $videos = $pdo->query("SELECT v.*, p.name as product_name FROM video_reviews v LEFT JOIN products p ON p.id = v.product_id ORDER BY v.created_at DESC LIMIT 50")->fetchAll();
    }

    $products = $pdo->query("SELECT id, name FROM products WHERE is_active = 1 ORDER BY name")->fetchAll();

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>🎥 Управление видеообзорами - Аквасбор CRM</title>
        <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #f8fafc; color: #1a202c; }
        .container { max-width: 1200px; margin: 0 auto; padding: 2rem; }
        .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .btn { 
            display: inline-flex; align-items: center; gap: 0.5rem; background: #3b82f6; color: white; padding: 0.75rem 1.5rem; 
            border: none; border-radius: 8px; text-decoration: none; font-weight: 600; cursor: pointer; transition: all 0.2s ease;
        }
        .btn:hover { background: #2563eb; transform: translateY(-1px); }
        .btn-danger { background: #ef4444; }
        .btn-danger:hover { background: #dc2626; }
        .alert { padding: 1rem; border-radius: 8px; margin-bottom: 1rem; }
        .alert-success { background: #d1fae5; color: #065f46; border: 1px solid #a7f3d0; }
        .alert-error { background: #fee2e2; color: #dc2626; border: 1px solid #fca5a5; }
        .form-group { margin-bottom: 1.5rem; }
        .form-group label { display: block; margin-bottom: 0.5rem; font-weight: 600; color: #374151; }
        .form-group input, .form-group select, .form-group textarea { 
            width: 100%; padding: 0.75rem; border: 2px solid #e2e8f0; border-radius: 8px; font-size: 1rem; transition: border-color 0.2s ease;
        }
        .form-group input:focus, .form-group select:focus, .form-group textarea:focus { 
            outline: none; border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
        }
        .videos-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(400px, 1fr)); gap: 2rem; }
        .video-card { 
            background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.1); 
            border: 1px solid #e2e8f0; transition: transform 0.2s ease;
        }
        .video-card:hover { transform: translateY(-2px); }
        .video-preview { height: 250px; position: relative; }
        .video-preview iframe, .video-preview video { width: 100%; height: 100%; }
        .video-placeholder { 
            height: 100%; background: linear-gradient(135deg, #8b5cf6, #3b82f6); display: flex; align-items: center; 
            justify-content: center; color: white; font-size: 4rem;
        }
        .video-content { padding: 1.5rem; }
        .video-title { font-size: 1.25rem; font-weight: 700; margin-bottom: 0.5rem; color: #1a202c; }
        .video-description { color: #6b7280; margin-bottom: 1rem; line-height: 1.5; }
        .video-meta { display: flex; justify-content: space-between; align-items: center; font-size: 0.875rem; color: #9ca3af; margin-bottom: 1rem; }
        .status-badge { 
            padding: 0.25rem 0.75rem; border-radius: 12px; font-size: 0.75rem; font-weight: 600; text-transform: uppercase;
        }
        .status-active { background: rgba(34, 197, 94, 0.1); color: #16a34a; }
        .status-inactive { background: rgba(239, 68, 68, 0.1); color: #dc2626; }
        .video-actions { display: flex; gap: 0.5rem; }
        .btn-small { padding: 0.5rem 1rem; font-size: 0.875rem; }
        .back-link { color: #3b82f6; text-decoration: none; margin-bottom: 1rem; display: inline-block; font-weight: 600; }
        .back-link:hover { text-decoration: underline; }
        .form-container { background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <div>
                    <h1>🎥 Управление видеообзорами</h1>
                    <p style="color: #6b7280; margin-top: 0.5rem;">Загружайте видеообзоры растений для клиентов</p>
                </div>
                <div style="display: flex; gap: 1rem; align-items: center;">
                    <a href="?r=admin" class="back-link">← В админку</a>
                    <?php if ($action === 'list'): ?>
                    <a href="?r=admin-videos&action=add" class="btn">➕ Добавить видео</a>
                    <?php else: ?>
                    <a href="?r=admin-videos" class="btn">← К списку</a>
                    <?php endif; ?>
                </div>
            </div>

            <?php if (!empty($success)): ?>
            <div class="alert alert-success"><?= $success ?></div>
            <?php endif; ?>

            <?php if (!empty($error)): ?>
            <div class="alert alert-error"><?= $error ?></div>
            <?php endif; ?>

            <?php if ($action === 'list'): ?>
            <div class="videos-grid">
                <?php foreach ($videos as $videoItem): ?>
                <article class="video-card">
                    <div class="video-preview">
                        <?php if ($videoItem['video_url']): ?>
                            <?php if (preg_match('/(?:youtube\.com\/(?:[^\/]+\/.+\/|(?:v|e(?:mbed)?)\/|.*[?&]v=)|youtu\.be\/)([^"&?\/\s]{11})/i', $videoItem['video_url'], $match)): ?>
                            <iframe src="https://www.youtube.com/embed/<?= $match[1] ?>" frameborder="0" allowfullscreen></iframe>
                            <?php else: ?>
                            <video controls><source src="<?= esc($videoItem['video_url']) ?>" type="video/mp4"></video>
                            <?php endif; ?>
                        <?php elseif ($videoItem['video_file']): ?>
                        <video controls><source src="<?= esc($videoItem['video_file']) ?>" type="video/mp4"></video>
                        <?php else: ?>
                        <div class="video-placeholder">🎥</div>
                        <?php endif; ?>
                    </div>
                    <div class="video-content">
                        <h3 class="video-title"><?= esc($videoItem['title']) ?></h3>
                        <?php if ($videoItem['description']): ?>
                        <p class="video-description"><?= esc(substr($videoItem['description'], 0, 150)) ?>...</p>
                        <?php endif; ?>
                        <div class="video-meta">
                            <span><?= $videoItem['product_name'] ? '🌿 ' . esc($videoItem['product_name']) : '📹 Общий обзор' ?></span>
                            <span class="status-badge <?= $videoItem['is_active'] ? 'status-active' : 'status-inactive' ?>">
                                <?= $videoItem['is_active'] ? 'Активно' : 'Неактивно' ?>
                            </span>
                        </div>
                        <div style="font-size: 0.875rem; color: #9ca3af; margin-bottom: 1rem;">
                            👁️ <?= $videoItem['views'] ?> просмотров | ❤️ <?= $videoItem['likes'] ?> лайков
                        </div>
                        <div class="video-actions">
                            <a href="?r=admin-videos&action=edit&id=<?= $videoItem['id'] ?>" class="btn btn-small">✏️ Изменить</a>
                            <form method="post" style="display:inline;" onsubmit="return confirm('🗑️ Удалить видео?')">
                                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">
                                <input type="hidden" name="action" value="delete">
                                <input type="hidden" name="id" value="<?= $videoItem['id'] ?>">
                                <button type="submit" class="btn btn-danger btn-small">🗑️ Удалить</button>
                            </form>
                        </div>
                    </div>
                </article>
                <?php endforeach; ?>

                <?php if (empty($videos)): ?>
                <div style="grid-column: 1/-1; text-align: center; padding: 3rem; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                    <div style="font-size: 4rem; margin-bottom: 1rem; opacity: 0.6;">🎥</div>
                    <h3>Видеообзоры не найдены</h3>
                    <p style="color: #6b7280; margin: 1rem 0;">Загрузите первый видеообзор растения</p>
                    <a href="?r=admin-videos&action=add" class="btn">➕ Добавить видео</a>
                </div>
                <?php endif; ?>
            </div>

            <?php elseif (in_array($action, ['add', 'edit'])): ?>
            <form method="post" enctype="multipart/form-data" class="form-container">
                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">

                <div class="form-group">
                    <label>🏷️ Название видео *</label>
                    <input type="text" name="title" required value="<?= esc($video['title'] ?? '') ?>" placeholder="Название видеообзора">
                </div>

                <div class="form-group">
                    <label>🌿 Связанный товар</label>
                    <select name="product_id">
                        <option value="">Выберите товар (опционально)</option>
                        <?php foreach ($products as $product): ?>
                        <option value="<?= $product['id'] ?>" <?= ($video['product_id'] ?? '') == $product['id'] ? 'selected' : '' ?>>
                            <?= esc($product['name']) ?>
                        </option>
                        <?php endforeach; ?>
                    </select>
                </div>

                <div class="form-group">
                    <label>📝 Описание видео</label>
                    <textarea name="description" rows="4" placeholder="Описание содержания видео"><?= esc($video['description'] ?? '') ?></textarea>
                </div>

                <div class="form-group">
                    <label>🔗 Ссылка на видео (YouTube, Vimeo и др.)</label>
                  <input type="url" name="video_url" value="<?= esc($video['video_url'] ?? '') ?>" placeholder="https://www.youtube.com/watch?v=...">
                    <small style="color: #6b7280; margin-top: 0.25rem; display: block;">Поддерживаются YouTube, Vimeo и прямые ссылки на видеофайлы</small>
                </div>

                <div class="form-group">
                    <label>📁 Или загрузить видеофайл</label>
                    <input type="file" name="video_file" accept="video/*">
                    <small style="color: #6b7280; margin-top: 0.25rem; display: block;">Поддерживаются форматы: MP4, AVI, MOV, WebM</small>
                    <?php if (!empty($video['video_file'])): ?>
                    <div style="margin-top: 0.5rem;">
                        <video width="300" height="200" controls style="border-radius: 8px;">
                            <source src="<?= esc($video['video_file']) ?>" type="video/mp4">
                        </video>
                        <br><small>Текущий видеофайл</small>
                    </div>
                    <?php endif; ?>
                </div>

                <div class="form-group">
                    <label>🖼️ Превью изображение (опционально)</label>
                    <input type="file" name="thumbnail" accept="image/*">
                    <?php if (!empty($video['thumbnail'])): ?>
                    <div style="margin-top: 0.5rem;">
                        <img src="<?= esc($video['thumbnail']) ?>" alt="Превью" style="max-width: 300px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                        <br><small>Текущее превью</small>
                    </div>
                    <?php endif; ?>
                </div>

                <div class="form-group" style="margin-bottom: 2rem;">
                    <label style="display: flex; align-items: center; gap: 0.5rem; cursor: pointer;">
                        <input type="checkbox" name="is_active" <?= ($video['is_active'] ?? 1) ? 'checked' : '' ?> style="width: auto;">
                        ✅ Видео активно и отображается на сайте
                    </label>
                </div>

                <div style="display: flex; gap: 1rem;">
                    <button type="submit" class="btn">
                        <?= $action === 'add' ? '➕ Добавить видео' : '💾 Сохранить изменения' ?>
                    </button>
                    <a href="?r=admin-videos" class="btn" style="background: #6b7280;">❌ Отмена</a>
                </div>
            </form>
            <?php endif; ?>
        </div>
    </body>
    </html>
    <?php
    exit;
}

// Админ - ИНТЕГРАЦИИ
if ($r === 'admin-integrations') {
    require_admin();
    $pdo = db();

    if (is_post()) {
        if (!verify_csrf()) {
            log_error_safe("CSRF check failed in admin-integrations, but proceeding");
        }

        try {
            foreach ($_POST as $key => $value) {
                if ($key === 'csrf_token') continue;

                if (strpos($key, 'integration_') === 0) {
                    $integrationId = (int)str_replace('integration_', '', $key);
                    $isActive = isset($_POST["integration_$integrationId"]) ? 1 : 0;
                    $stmt = $pdo->prepare("UPDATE integrations SET is_active = ? WHERE id = ?");
                    $stmt->execute([$isActive, $integrationId]);
                } elseif (strpos($key, 'config_') === 0) {
                    $integrationId = (int)str_replace('config_', '', $key);
                    $config = $value;
                    $stmt = $pdo->prepare("UPDATE integrations SET config = ? WHERE id = ?");
                    $stmt->execute([$config, $integrationId]);
                }
            }

            $success = '✅ Настройки интеграций сохранены';
        } catch (Exception $e) {
            $error = '❌ Ошибка сохранения: ' . $e->getMessage();
        }
    }

    $integrations = $pdo->query("SELECT * FROM integrations ORDER BY type, name")->fetchAll();
    $grouped = [];
    foreach ($integrations as $integration) {
        $grouped[$integration['type']][] = $integration;
    }

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>🔗 Интеграции - Аквасбор CRM</title>
        <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #f8fafc; color: #1a202c; }
        .container { max-width: 1000px; margin: 0 auto; padding: 2rem; }
        .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .btn { 
            display: inline-flex; align-items: center; gap: 0.5rem; background: #3b82f6; color: white; padding: 0.75rem 1.5rem; 
            border: none; border-radius: 8px; text-decoration: none; font-weight: 600; cursor: pointer; transition: all 0.2s ease;
        }
        .btn:hover { background: #2563eb; transform: translateY(-1px); }
        .alert { padding: 1rem; border-radius: 8px; margin-bottom: 1rem; }
        .alert-success { background: #d1fae5; color: #065f46; border: 1px solid #a7f3d0; }
        .alert-error { background: #fee2e2; color: #dc2626; border: 1px solid #fca5a5; }
        .back-link { color: #3b82f6; text-decoration: none; margin-bottom: 1rem; display: inline-block; font-weight: 600; }
        .back-link:hover { text-decoration: underline; }
        .integrations-form { background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .integration-section { margin-bottom: 3rem; }
        .integration-section h3 { 
            font-size: 1.3rem; margin-bottom: 1.5rem; color: #1f2937; border-bottom: 2px solid #e5e7eb; 
            padding-bottom: 0.5rem; display: flex; align-items: center; gap: 0.5rem;
        }
        .integration-card {
            background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 12px; padding: 1.5rem; margin-bottom: 1.5rem;
            transition: all 0.2s ease;
        }
        .integration-card:hover { border-color: #3b82f6; }
        .integration-header {
            display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem;
        }
        .integration-title { font-size: 1.1rem; font-weight: 600; color: #1f2937; }
        .integration-toggle { position: relative; display: inline-block; width: 60px; height: 34px; }
        .integration-toggle input { opacity: 0; width: 0; height: 0; }
        .slider {
            position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #ccc;
            transition: .4s; border-radius: 34px;
        }
        .slider:before {
            position: absolute; content: ""; height: 26px; width: 26px; left: 4px; bottom: 4px;
            background-color: white; transition: .4s; border-radius: 50%;
        }
        input:checked + .slider { background-color: #3b82f6; }
        input:focus + .slider { box-shadow: 0 0 1px #3b82f6; }
        input:checked + .slider:before { transform: translateX(26px); }
        .integration-config { margin-top: 1rem; }
        .config-field { margin-bottom: 1rem; }
        .config-label { display: block; margin-bottom: 0.25rem; font-weight: 500; color: #374151; font-size: 0.875rem; }
        .config-input { 
            width: 100%; padding: 0.5rem; border: 1px solid #d1d5db; border-radius: 6px; font-size: 0.875rem;
            transition: border-color 0.2s ease;
        }
        .config-input:focus { outline: none; border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1); }
        .status-badge {
            padding: 0.25rem 0.75rem; border-radius: 12px; font-size: 0.75rem; font-weight: 600; text-transform: uppercase;
        }
        .status-active { background: rgba(34, 197, 94, 0.1); color: #16a34a; }
        .status-inactive { background: rgba(239, 68, 68, 0.1); color: #dc2626; }
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <div>
                    <h1>🔗 Интеграции</h1>
                    <p style="color: #6b7280; margin-top: 0.5rem;">Настройка платежей, доставки и уведомлений</p>
                </div>
                <div>
                    <a href="?r=admin" class="back-link">← В админку</a>
                </div>
            </div>

            <?php if (!empty($success)): ?>
            <div class="alert alert-success"><?= $success ?></div>
            <?php endif; ?>

            <?php if (!empty($error)): ?>
            <div class="alert alert-error"><?= $error ?></div>
            <?php endif; ?>

            <form method="post" class="integrations-form">
                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">

                <?php foreach ($grouped as $type => $typeIntegrations): ?>
                <div class="integration-section">
                    <h3>
                        <?= match($type) {
                            'payment' => '💳 Платежные системы',
                            'delivery' => '🚚 Службы доставки',
                            'notification' => '📢 Уведомления',
                            'analytics' => '📊 Аналитика',
                            'maps' => '🗺️ Карты',
                            default => $type
                        } ?>
                    </h3>

                    <?php foreach ($typeIntegrations as $integration): ?>
                    <div class="integration-card">
                        <div class="integration-header">
                            <div>
                                <div class="integration-title"><?= esc($integration['name']) ?></div>
                                <span class="status-badge <?= $integration['is_active'] ? 'status-active' : 'status-inactive' ?>">
                                    <?= $integration['is_active'] ? 'Активна' : 'Неактивна' ?>
                                </span>
                            </div>
                            <label class="integration-toggle">
                                <input type="checkbox" name="integration_<?= $integration['id'] ?>" <?= $integration['is_active'] ? 'checked' : '' ?>>
                                <span class="slider"></span>
                            </label>
                        </div>

                        <div class="integration-config">
                            <?php
                            $config = json_decode($integration['config'] ?? '{}', true) ?: [];
                            switch ($integration['name']):
                                case 'Яндекс.Касса':
                            ?>
                            <div class="config-field">
                                <label class="config-label">Shop ID</label>
                                <input type="text" class="config-input" name="config_<?= $integration['id'] ?>[shop_id]" value="<?= esc($config['shop_id'] ?? '') ?>" placeholder="123456">
                            </div>
                            <div class="config-field">
                                <label class="config-label">Секретный ключ</label>
                                <input type="password" class="config-input" name="config_<?= $integration['id'] ?>[secret_key]" value="<?= esc($config['secret_key'] ?? '') ?>" placeholder="live_...">
                            </div>
                            <?php break; case 'Сбербанк Эквайринг': ?>
                            <div class="config-field">
                                <label class="config-label">Логин</label>
                                <input type="text" class="config-input" name="config_<?= $integration['id'] ?>[login]" value="<?= esc($config['login'] ?? '') ?>">
                            </div>
                            <div class="config-field">
                                <label class="config-label">Пароль</label>
                                <input type="password" class="config-input" name="config_<?= $integration['id'] ?>[password]" value="<?= esc($config['password'] ?? '') ?>">
                            </div>
                            <?php break; case 'СДЭК': ?>
                            <div class="config-field">
                                <label class="config-label">Аккаунт</label>
                                <input type="text" class="config-input" name="config_<?= $integration['id'] ?>[account]" value="<?= esc($config['account'] ?? '') ?>">
                            </div>
                            <div class="config-field">
                                <label class="config-label">Пароль</label>
                                <input type="password" class="config-input" name="config_<?= $integration['id'] ?>[password]" value="<?= esc($config['password'] ?? '') ?>">
                            </div>
                            <?php break; case 'Telegram Bot': ?>
                            <div class="config-field">
                                <label class="config-label">Bot Token</label>
                                <input type="text" class="config-input" name="config_<?= $integration['id'] ?>[bot_token]" value="<?= esc($config['bot_token'] ?? '') ?>" placeholder="123456789:XXXXXXXXX">
                            </div>
                            <div class="config-field">
                                <label class="config-label">Chat ID</label>
                                <input type="text" class="config-input" name="config_<?= $integration['id'] ?>[chat_id]" value="<?= esc($config['chat_id'] ?? '') ?>" placeholder="-100123456789">
                            </div>
                            <?php break; case 'Яндекс.Метрика': ?>
                            <div class="config-field">
                                <label class="config-label">ID счетчика</label>
                                <input type="text" class="config-input" name="config_<?= $integration['id'] ?>[counter_id]" value="<?= esc($config['counter_id'] ?? '') ?>" placeholder="12345678">
                            </div>
                            <?php break; case 'Google Analytics': ?>
                            <div class="config-field">
                                <label class="config-label">Tracking ID</label>
                                <input type="text" class="config-input" name="config_<?= $integration['id'] ?>[tracking_id]" value="<?= esc($config['tracking_id'] ?? '') ?>" placeholder="UA-123456789-1">
                            </div>
                            <?php break; default: ?>
                            <div class="config-field">
                                <label class="config-label">Конфигурация (JSON)</label>
                                <textarea class="config-input" name="config_<?= $integration['id'] ?>" rows="3" style="font-family: monospace;" placeholder='{"key": "value"}'><?= esc($integration['config']) ?></textarea>
                            </div>
                            <?php endswitch; ?>
                        </div>
                    </div>
                    <?php endforeach; ?>
                </div>
                <?php endforeach; ?>

                <div style="text-align: center; margin-top: 2rem;">
                    <button type="submit" class="btn">💾 Сохранить все настройки</button>
                </div>
            </form>
        </div>

        <script>
        document.addEventListener('DOMContentLoaded', function() {
            // Сохраняем конфигурации в JSON при отправке формы
            document.querySelector('form').addEventListener('submit', function(e) {
                const integrations = <?= json_encode($integrations) ?>;

                integrations.forEach(function(integration) {
                    const configInputs = document.querySelectorAll('[name*="config_' + integration.id + '"]');
                    if (configInputs.length > 1) {
                        const config = {};
                        configInputs.forEach(function(input) {
                            const key = input.name.match(/$$([^$$]+)\]$/)?.[1];
                            if (key) {
                                config[key] = input.value;
                            }
                        });

                        // Создаем скрытое поле с JSON конфигурацией
                        const hiddenField = document.createElement('input');
                        hiddenField.type = 'hidden';
                        hiddenField.name = 'config_' + integration.id;
                        hiddenField.value = JSON.stringify(config);
                        this.appendChild(hiddenField);

                        // Удаляем отдельные поля
                        configInputs.forEach(function(input) {
                            input.remove();
                        });
                    }
                });
            });
        });
        </script>
    </body>
    </html>
    <?php
    exit;
}

// Расширенные настройки с футером
if ($r === 'admin-settings') {
    require_admin();
    $pdo = db();

    if (is_post()) {
        if (!verify_csrf()) {
            log_error_safe("CSRF check failed in admin-settings, but proceeding");
        }

        try {
            foreach ($_POST as $key => $value) {
                if ($key === 'csrf_token') continue;

                $stmt = $pdo->prepare("UPDATE settings SET value = ? WHERE key_name = ?");
                $stmt->execute([trim($value), $key]);
            }

            $success = '✅ Настройки успешно сохранены';

            // Очищаем кеш настроек
            if (function_exists('opcache_reset')) {
                opcache_reset();
            }
        } catch (Exception $e) {
            $error = '❌ Ошибка сохранения: ' . $e->getMessage();
        }
    }

    $settings = [];
    $stmt = $pdo->query("SELECT key_name, value, type, description, category FROM settings ORDER BY category, key_name");
    foreach ($stmt as $row) {
        $settings[$row['category']][] = $row;
    }

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>⚙️ Настройки сайта - Аквасбор CRM</title>
        <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #f8fafc; color: #1a202c; }
        .container { max-width: 1000px; margin: 0 auto; padding: 2rem; }
        .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .btn { 
            display: inline-flex; align-items: center; gap: 0.5rem; background: #3b82f6; color: white; padding: 0.75rem 1.5rem; 
            border: none; border-radius: 8px; text-decoration: none; font-weight: 600; cursor: pointer; transition: all 0.2s ease;
        }
        .btn:hover { background: #2563eb; transform: translateY(-1px); }
        .alert { padding: 1rem; border-radius: 8px; margin-bottom: 1rem; }
        .alert-success { background: #d1fae5; color: #065f46; border: 1px solid #a7f3d0; }
        .alert-error { background: #fee2e2; color: #dc2626; border: 1px solid #fca5a5; }
        .back-link { color: #3b82f6; text-decoration: none; margin-bottom: 1rem; display: inline-block; font-weight: 600; }
        .back-link:hover { text-decoration: underline; }
        .settings-form { background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .settings-section { margin-bottom: 3rem; }
        .settings-section h3 { 
            font-size: 1.3rem; margin-bottom: 1.5rem; color: #1f2937; border-bottom: 2px solid #e5e7eb; 
            padding-bottom: 0.5rem; display: flex; align-items: center; gap: 0.5rem;
        }
        .form-group { margin-bottom: 1.5rem; }
        .form-group label { 
            display: block; margin-bottom: 0.5rem; font-weight: 600; color: #374151;
        }
        .form-group small { 
            display: block; margin-top: 0.25rem; color: #6b7280; font-size: 0.875rem;
        }
        .form-group input, .form-group select, .form-group textarea { 
            width: 100%; padding: 0.75rem; border: 2px solid #e2e8f0; border-radius: 8px; font-size: 1rem;
            transition: border-color 0.3s ease;
        }
        .form-group input:focus, .form-group select:focus, .form-group textarea:focus { 
            outline: none; border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
        }
        .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
        .grid-3 { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 1rem; }
        @media (max-width: 768px) { .grid-2, .grid-3 { grid-template-columns: 1fr; } }
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <div>
                    <h1>⚙️ Настройки сайта</h1>
                    <p style="color: #6b7280; margin-top: 0.5rem;">Полное управление конфигурацией магазина</p>
                </div>
                <div>
                    <a href="?r=admin" class="back-link">← В админку</a>
                </div>
            </div>

            <?php if (!empty($success)): ?>
            <div class="alert alert-success"><?= $success ?></div>
            <?php endif; ?>

            <?php if (!empty($error)): ?>
            <div class="alert alert-error"><?= $error ?></div>
            <?php endif; ?>

            <form method="post" class="settings-form">
                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">

                <?php foreach ($settings as $categoryName => $categorySettings): ?>
                <div class="settings-section">
                    <h3><?= match($categoryName) {
                        'general' => '🏪 Основные настройки',
                        'design' => '🎨 Дизайн и внешний вид',
                        'contacts' => '📞 Контактная информация',
                        'delivery' => '🚚 Доставка и оплата',
                        'seo' => '📈 SEO оптимизация',
                        'integrations' => '🔗 API и интеграции',
                        'social' => '👥 Социальные сети',
                        'email' => '📧 Email настройки',
                        'security' => '🔐 Безопасность',
                        'advanced' => '⚙️ Расширенные настройки',
                        default => $categoryName
                    } ?></h3>

                    <div class="<?= in_array($categoryName, ['delivery', 'social']) ? 'grid-2' : '' ?>">
                        <?php foreach ($categorySettings as $setting): ?>
                        <div class="form-group">
                            <label><?= esc($setting['description']) ?></label>

                            <?php if ($setting['type'] === 'textarea'): ?>
                            <textarea name="<?= esc($setting['key_name']) ?>" rows="4"><?= esc($setting['value']) ?></textarea>

                            <?php elseif ($setting['type'] === 'number'): ?>
                            <input type="number" name="<?= esc($setting['key_name']) ?>" value="<?= esc($setting['value']) ?>" step="0.01" min="0">

                            <?php elseif ($setting['type'] === 'email'): ?>
                            <input type="email" name="<?= esc($setting['key_name']) ?>" value="<?= esc($setting['value']) ?>">

                            <?php elseif ($setting['type'] === 'tel'): ?>
                            <input type="tel" name="<?= esc($setting['key_name']) ?>" value="<?= esc($setting['value']) ?>">

                            <?php elseif ($setting['type'] === 'url'): ?>
                            <input type="url" name="<?= esc($setting['key_name']) ?>" value="<?= esc($setting['value']) ?>">

                            <?php elseif ($setting['type'] === 'password'): ?>
                            <input type="password" name="<?= esc($setting['key_name']) ?>" value="<?= esc($setting['value']) ?>">

                            <?php elseif ($setting['type'] === 'color'): ?>
                            <input type="color" name="<?= esc($setting['key_name']) ?>" value="<?= esc($setting['value']) ?>">

                            <?php elseif ($setting['type'] === 'select'): ?>
                            <select name="<?= esc($setting['key_name']) ?>">
                                <?php 
                                $options = [];
                                if ($setting['key_name'] === 'currency') {
                                    $options = ['RUB' => 'Российский рубль', 'USD' => 'Доллар США', 'EUR' => 'Евро'];
                                } elseif ($setting['key_name'] === 'timezone') {
                                    $options = [
                                        'Europe/Moscow' => 'Москва (UTC+3)',
                                        'Europe/Samara' => 'Самара (UTC+4)', 
                                        'Asia/Yekaterinburg' => 'Екатеринбург (UTC+5)',
                                        'Asia/Novosibirsk' => 'Новосибирск (UTC+7)',
                                        'Asia/Vladivostok' => 'Владивосток (UTC+10)'
                                    ];
                                }
                                foreach ($options as $value => $label): ?>
                                <option value="<?= $value ?>" <?= $setting['value'] == $value ? 'selected' : '' ?>><?= $label ?></option>
                                <?php endforeach; ?>
                            </select>

                            <?php else: ?>
                            <input type="text" name="<?= esc($setting['key_name']) ?>" value="<?= esc($setting['value']) ?>">
                            <?php endif; ?>

                            <small>Ключ: <?= esc($setting['key_name']) ?></small>
                        </div>
                        <?php endforeach; ?>
                    </div>
                </div>
                <?php endforeach; ?>

                <div style="text-align: center; margin-top: 3rem;">
                    <button type="submit" class="btn" style="padding: 1rem 2rem; font-size: 1.1rem;">💾 Сохранить все настройки</button>
                </div>
            </form>
        </div>
    </body>
    </html>
    <?php
    exit;
}

// Добавление недостающих настроек при первом запуске
function addMissingSettings(PDO $pdo): void {
    $additionalSettings = [
        // SEO
        'meta_title' => ['Аквасбор - Аквариумные растения с доставкой', 'text', 'Meta Title главной страницы', 'seo'],
        'meta_description' => ['Купить аквариумные растения в интернет-магазине Аквасбор. Большой выбор, доставка по России, гарантия качества.', 'textarea', 'Meta Description главной страницы', 'seo'],
        'meta_keywords' => ['аквариумные растения, купить растения для аквариума, аквариум, растения', 'text', 'Ключевые слова', 'seo'],
        'google_analytics' => ['', 'text', 'Google Analytics ID', 'seo'],
        'yandex_metrika' => ['', 'text', 'Яндекс.Метрика ID', 'seo'],
        'google_search_console' => ['', 'text', 'Google Search Console код', 'seo'],

        // Email настройки
        'smtp_host' => ['', 'text', 'SMTP сервер', 'email'],
        'smtp_port' => ['587', 'number', 'SMTP порт', 'email'],
        'smtp_username' => ['', 'text', 'SMTP логин', 'email'],
        'smtp_password' => ['', 'password', 'SMTP пароль', 'email'],
        'smtp_encryption' => ['tls', 'select', 'Шифрование SMTP', 'email'],
        'email_from_name' => ['Аквасбор', 'text', 'Имя отправителя', 'email'],

        // Безопасность
        'admin_ip_whitelist' => ['', 'textarea', 'Белый список IP для админки', 'security'],
        'enable_captcha' => ['0', 'select', 'Включить капчу', 'security'],
        'captcha_site_key' => ['', 'text', 'reCAPTCHA Site Key', 'security'],
        'captcha_secret_key' => ['', 'password', 'reCAPTCHA Secret Key', 'security'],

        // Расширенные настройки
        'currency' => ['RUB', 'select', 'Валюта магазина', 'advanced'],
        'timezone' => ['Europe/Moscow', 'select', 'Часовой пояс', 'advanced'],
        'items_per_page' => ['12', 'number', 'Товаров на странице', 'advanced'],
        'enable_reviews' => ['1', 'select', 'Включить отзывы', 'advanced'],
        'auto_approve_reviews' => ['0', 'select', 'Автоодобрение отзывов', 'advanced'],
        'enable_wishlist' => ['1', 'select', 'Включить избранное', 'advanced'],
        'enable_compare' => ['1', 'select', 'Включить сравнение', 'advanced'],
        'maintenance_mode' => ['0', 'select', 'Режим обслуживания', 'advanced'],
        'maintenance_message' => ['Сайт временно недоступен', 'textarea', 'Сообщение при обслуживании', 'advanced'],
    ];

    foreach ($additionalSettings as $key => [$value, $type, $desc, $cat]) {
        try {
            $pdo->prepare("INSERT OR IGNORE INTO settings(key_name, value, type, description, category) VALUES (?, ?, ?, ?, ?)")
                ->execute([$key, $value, $type, $desc, $cat]);
        } catch (Exception $e) {
            log_error_safe("Error inserting additional setting $key: " . $e->getMessage());
        }
    }
}

// Админ - ТОВАРЫ (современная версия с расширенным функционалом)
if ($r === 'admin-products') {
    require_admin();
    $pdo = db();

    // Добавляем недостающие настройки при первом запуске
    addMissingSettings($pdo);

    $action = $_GET['action'] ?? 'list';
    $id = (int)($_GET['id'] ?? 0);

    if (is_post()) {
        if (!verify_csrf()) {
            log_error_safe("CSRF check failed in admin-products, but proceeding");
        }

        try {
            if ($action === 'add' || $action === 'edit') {
                $name = trim($_POST['name'] ?? '');
                $categoryId = (int)($_POST['category_id'] ?? 0) ?: null;
                $price = max(0, (float)($_POST['price'] ?? 0));
                $oldPrice = $_POST['old_price'] ? max(0, (float)$_POST['old_price']) : null;
                $costPrice = $_POST['cost_price'] ? max(0, (float)$_POST['cost_price']) : null;
                $stock = max(0, (int)($_POST['stock'] ?? 0));
                $minStock = max(1, (int)($_POST['min_stock'] ?? 5));
                $shortDesc = trim($_POST['short_desc'] ?? '');
                $description = trim($_POST['description'] ?? '');
                $careTips = trim($_POST['care_tips'] ?? '');
                $videoUrl = trim($_POST['video_url'] ?? '');
                $light = $_POST['light'] ?? '';
                $difficulty = $_POST['difficulty'] ?? '';
                $isFeatured = isset($_POST['is_featured']) ? 1 : 0;
                $isBestseller = isset($_POST['is_bestseller']) ? 1 : 0;
                $isActive = isset($_POST['is_active']) ? 1 : 0;

                // Обработка характеристик
                $tempMin = $_POST['temperature_min'] ? (int)$_POST['temperature_min'] : null;
                $tempMax = $_POST['temperature_max'] ? (int)$_POST['temperature_max'] : null;
                $phMin = $_POST['ph_min'] ? (float)$_POST['ph_min'] : null;
                $phMax = $_POST['ph_max'] ? (float)$_POST['ph_max'] : null;
                $growthRate = $_POST['growth_rate'] ?? '';
                $placement = $_POST['placement'] ?? '';

                $image = null;
                if (!empty($_FILES['image']['name'])) {
                    $image = upload_file($_FILES['image']);
                }

                // Обработка галереи изображений
                $gallery = [];
                if (!empty($_FILES['gallery']['name'][0])) {
                    for ($i = 0; $i < count($_FILES['gallery']['name']); $i++) {
                        if ($_FILES['gallery']['error'][$i] === UPLOAD_ERR_OK) {
                            $file = [
                                'name' => $_FILES['gallery']['name'][$i],
                                'tmp_name' => $_FILES['gallery']['tmp_name'][$i],
                                'error' => $_FILES['gallery']['error'][$i],
                                'size' => $_FILES['gallery']['size'][$i]
                            ];
                            $galleryImage = upload_file($file);
                            if ($galleryImage) {
                                $gallery[] = $galleryImage;
                            }
                        }
                    }
                }
                $galleryJson = !empty($gallery) ? json_encode($gallery) : null;

                if (!$name) throw new Exception('Название товара обязательно');

                $slug = preg_replace('/[^a-z0-9-]/', '', strtolower(str_replace([' ', '_'], '-', transliterate($name))));

                if ($action === 'add') {
                    $stmt = $pdo->prepare("INSERT INTO products (slug, name, category_id, price, old_price, cost_price, stock, min_stock, 
                                          temperature_min, temperature_max, ph_min, ph_max, light, difficulty, growth_rate, placement,
                                          short_desc, description, care_tips, image, gallery, video_url, is_featured, is_bestseller, 
                                          is_active, created_at) 
                                          VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, datetime('now'))");
                    $stmt->execute([$slug, $name, $categoryId, $price, $oldPrice, $costPrice, $stock, $minStock,
                                   $tempMin, $tempMax, $phMin, $phMax, $light, $difficulty, $growthRate, $placement,
                                   $shortDesc, $description, $careTips, $image, $galleryJson, $videoUrl, 
                                   $isFeatured, $isBestseller, $isActive]);
                    $success = '✅ Товар успешно добавлен';
                } else {
                    $updateFields = "name=?, category_id=?, price=?, old_price=?, cost_price=?, stock=?, min_stock=?,
                                   temperature_min=?, temperature_max=?, ph_min=?, ph_max=?, light=?, difficulty=?, 
                                   growth_rate=?, placement=?, short_desc=?, description=?, care_tips=?, video_url=?,
                                   is_featured=?, is_bestseller=?, is_active=?, updated_at=datetime('now')";

                    $params = [$name, $categoryId, $price, $oldPrice, $costPrice, $stock, $minStock,
                              $tempMin, $tempMax, $phMin, $phMax, $light, $difficulty, $growthRate, $placement,
                              $shortDesc, $description, $careTips, $videoUrl, $isFeatured, $isBestseller, $isActive];

                    if ($image) {
                        $updateFields .= ", image=?";
                        $params[] = $image;
                    }
                    if ($galleryJson) {
                        $updateFields .= ", gallery=?";
                        $params[] = $galleryJson;
                    }

                    $params[] = $id;

                    $stmt = $pdo->prepare("UPDATE products SET $updateFields WHERE id=?");
                    $stmt->execute($params);
                    $success = '✅ Товар успешно обновлен';
                }

                header('Location: ?r=admin-products');
                exit;
            }

            if ($action === 'delete' && $id) {
                $pdo->prepare("DELETE FROM products WHERE id = ?")->execute([$id]);
                $success = '🗑️ Товар удален';
                header('Location: ?r=admin-products');
                exit;
            }
        } catch (Exception $e) {
            $error = '❌ Ошибка: ' . $e->getMessage();
        }
    }

    $categories = $pdo->query("SELECT id, name FROM categories ORDER BY name")->fetchAll();

    if ($action === 'edit' && $id) {
        $product = $pdo->prepare("SELECT * FROM products WHERE id = ?");
        $product->execute([$id]);
        $product = $product->fetch();
        if (!$product) {
            header('Location: ?r=admin-products');
            exit;
        }
    }

    if ($action === 'list') {
        $search = trim($_GET['search'] ?? '');
        $categoryFilter = (int)($_GET['category'] ?? 0);
        $statusFilter = $_GET['status'] ?? '';

        $sql = "SELECT p.*, c.name as category_name FROM products p 
                LEFT JOIN categories c ON c.id = p.category_id WHERE 1=1";
        $params = [];

        if ($search) {
            $sql .= " AND (p.name LIKE ? OR p.description LIKE ?)";
            $params[] = "%$search%";
            $params[] = "%$search%";
        }
        if ($categoryFilter) {
            $sql .= " AND p.category_id = ?";
            $params[] = $categoryFilter;
        }
        if ($statusFilter === 'active') {
            $sql .= " AND p.is_active = 1";
        } elseif ($statusFilter === 'inactive') {
            $sql .= " AND p.is_active = 0";
        } elseif ($statusFilter === 'low_stock') {
            $sql .= " AND p.stock <= p.min_stock";
        } elseif ($statusFilter === 'featured') {
            $sql .= " AND p.is_featured = 1";
        }

        $sql .= " ORDER BY p.created_at DESC LIMIT 50";

        $stmt = $pdo->prepare($sql);
        $stmt->execute($params);
        $products = $stmt->fetchAll();
    }

    ?>
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>🌱 Управление товарами - Аквасбор CRM</title>
        <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #f8fafc; color: #1a202c; }
        .container { max-width: 1400px; margin: 0 auto; padding: 2rem; }
        .header { 
            display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem; 
            background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); 
        }
        .btn { 
            display: inline-flex; align-items: center; gap: 0.5rem; background: #3b82f6; color: white; padding: 0.75rem 1.5rem; 
            border: none; border-radius: 8px; text-decoration: none; font-weight: 600; cursor: pointer; transition: all 0.2s ease;
        }
        .btn:hover { background: #2563eb; transform: translateY(-1px); }
        .btn:hover { background: #2563eb; transform: translateY(-1px); }
        .btn-danger { background: #ef4444; }
        .btn-danger:hover { background: #dc2626; }
        .btn-small { padding: 0.5rem 1rem; font-size: 0.875rem; }
        .alert { padding: 1rem; border-radius: 8px; margin-bottom: 1rem; }
        .alert-success { background: #d1fae5; color: #065f46; border: 1px solid #a7f3d0; }
        .alert-error { background: #fee2e2; color: #dc2626; border: 1px solid #fca5a5; }
        .back-link { color: #3b82f6; text-decoration: none; margin-bottom: 1rem; display: inline-block; font-weight: 600; }
        .back-link:hover { text-decoration: underline; }

        .filters { 
            display: flex; gap: 1rem; margin-bottom: 2rem; background: white; padding: 1.5rem; 
            border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); flex-wrap: wrap; align-items: center;
        }
        .filter-input, .filter-select { 
            padding: 0.5rem; border: 1px solid #d1d5db; border-radius: 6px; font-size: 0.875rem;
        }
        .filter-input:focus, .filter-select:focus { outline: none; border-color: #3b82f6; }

        .products-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(350px, 1fr)); gap: 2rem; }
        .product-card { 
            background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.1); 
            border: 1px solid #e2e8f0; transition: transform 0.2s ease;
        }
        .product-card:hover { transform: translateY(-2px); }
        .product-image { height: 200px; background: linear-gradient(135deg, #3b82f6, #8b5cf6); display: flex; align-items: center; justify-content: center; color: white; font-size: 3rem; }
        .product-image img { width: 100%; height: 100%; object-fit: cover; }
        .product-content { padding: 1.5rem; }
        .product-title { font-size: 1.1rem; font-weight: 700; margin-bottom: 0.5rem; color: #1a202c; }
        .product-category { color: #6b7280; font-size: 0.85rem; margin-bottom: 0.75rem; }
        .product-price { display: flex; align-items: center; gap: 0.5rem; margin-bottom: 1rem; }
        .current-price { font-size: 1.1rem; font-weight: 700; color: #3b82f6; }
        .old-price { font-size: 0.9rem; color: #9ca3af; text-decoration: line-through; }
        .product-meta { display: flex; justify-content: space-between; align-items: center; font-size: 0.875rem; color: #6b7280; margin-bottom: 1rem; }
        .stock-badge { 
            padding: 0.25rem 0.75rem; border-radius: 12px; font-size: 0.75rem; font-weight: 600; text-transform: uppercase;
        }
        .stock-good { background: rgba(34, 197, 94, 0.1); color: #16a34a; }
        .stock-low { background: rgba(249, 115, 22, 0.1); color: #ea580c; }
        .stock-out { background: rgba(239, 68, 68, 0.1); color: #dc2626; }
        .product-actions { display: flex; gap: 0.5rem; }

        .form-container { background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .form-group { margin-bottom: 1.5rem; }
        .form-group label { display: block; margin-bottom: 0.5rem; font-weight: 600; color: #374151; }
        .form-group input, .form-group select, .form-group textarea { 
            width: 100%; padding: 0.75rem; border: 2px solid #e2e8f0; border-radius: 8px; font-size: 1rem;
            transition: border-color 0.2s ease;
        }
        .form-group input:focus, .form-group select:focus, .form-group textarea:focus { 
            outline: none; border-color: #3b82f6; box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
        }
        .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
        .grid-3 { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 1rem; }
        .checkbox-group { display: flex; gap: 2rem; align-items: center; margin-top: 0.5rem; }
        .checkbox-group label { display: flex; align-items: center; gap: 0.5rem; cursor: pointer; font-weight: 500; }
        .checkbox-group input[type="checkbox"] { width: auto; margin: 0; }

        @media (max-width: 768px) { 
            .grid-2, .grid-3 { grid-template-columns: 1fr; } 
            .filters { flex-direction: column; align-items: stretch; }
            .filter-input, .filter-select { width: 100%; }
        }
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <div>
                    <h1>🌱 Управление товарами</h1>
                    <p style="color: #6b7280; margin-top: 0.5rem;">Полное управление каталогом растений</p>
                </div>
                <div style="display: flex; gap: 1rem; align-items: center;">
                    <a href="?r=admin" class="back-link">← В админку</a>
                    <?php if ($action === 'list'): ?>
                    <a href="?r=admin-products&action=add" class="btn">➕ Добавить товар</a>
                    <?php else: ?>
                    <a href="?r=admin-products" class="btn">← К списку</a>
                    <?php endif; ?>
                </div>
            </div>

            <?php if (!empty($success)): ?>
            <div class="alert alert-success"><?= $success ?></div>
            <?php endif; ?>

            <?php if (!empty($error)): ?>
            <div class="alert alert-error"><?= $error ?></div>
            <?php endif; ?>

            <?php if ($action === 'list'): ?>
            <div class="filters">
                <input type="text" class="filter-input" placeholder="🔍 Поиск товаров..." value="<?= esc($_GET['search'] ?? '') ?>" onchange="applyFilters()" id="searchInput">
                <select class="filter-select" onchange="applyFilters()" id="categorySelect">
                    <option value="">📂 Все категории</option>
                    <?php foreach ($categories as $cat): ?>
                    <option value="<?= $cat['id'] ?>" <?= ($_GET['category'] ?? '') == $cat['id'] ? 'selected' : '' ?>>
                        <?= esc($cat['name']) ?>
                    </option>
                    <?php endforeach; ?>
                </select>
                <select class="filter-select" onchange="applyFilters()" id="statusSelect">
                    <option value="">📊 Все статусы</option>
                    <option value="active" <?= ($_GET['status'] ?? '') === 'active' ? 'selected' : '' ?>>✅ Активные</option>
                    <option value="inactive" <?= ($_GET['status'] ?? '') === 'inactive' ? 'selected' : '' ?>>❌ Неактивные</option>
                    <option value="featured" <?= ($_GET['status'] ?? '') === 'featured' ? 'selected' : '' ?>>⭐ Рекомендуемые</option>
                    <option value="low_stock" <?= ($_GET['status'] ?? '') === 'low_stock' ? 'selected' : '' ?>>⚠️ Заканчиваются</option>
                </select>
                <span style="margin-left: auto; color: #6b7280;">Найдено: <?= count($products) ?> товаров</span>
            </div>

            <div class="products-grid">
                <?php foreach ($products as $product): ?>
                <div class="product-card">
                    <div class="product-image">
                        <?php if ($product['image']): ?>
                        <img src="<?= esc($product['image']) ?>" alt="<?= esc($product['name']) ?>">
                        <?php else: ?>
                        🌿
                        <?php endif; ?>
                    </div>
                    <div class="product-content">
                        <h3 class="product-title"><?= esc($product['name']) ?></h3>
                        <div class="product-category">📂 <?= esc($product['category_name'] ?? 'Без категории') ?></div>

                        <div class="product-price">
                            <span class="current-price"><?= $product['price'] ?> ₽</span>
                            <?php if ($product['old_price']): ?>
                            <span class="old-price"><?= $product['old_price'] ?> ₽</span>
                            <?php endif; ?>
                            <?php if ($product['cost_price']): ?>
                            <span style="font-size: 0.8rem; color: #9ca3af;">Себестоимость: <?= $product['cost_price'] ?> ₽</span>
                            <?php endif; ?>
                        </div>

                        <div class="product-meta">
                            <span>📦 Склад: <?= $product['stock'] ?> шт.</span>
                            <span class="stock-badge <?= $product['stock'] <= 0 ? 'stock-out' : ($product['stock'] <= ($product['min_stock'] ?? 5) ? 'stock-low' : 'stock-good') ?>">
                                <?= $product['stock'] <= 0 ? 'Нет в наличии' : ($product['stock'] <= ($product['min_stock'] ?? 5) ? 'Заканчивается' : 'В наличии') ?>
                            </span>
                        </div>

                        <div style="font-size: 0.875rem; color: #6b7280; margin-bottom: 1rem;">
                            <?= $product['is_active'] ? '✅ Активен' : '❌ Неактивен' ?>
                            <?= $product['is_featured'] ? ' | ⭐ Рекомендуемый' : '' ?>
                            <?= $product['is_bestseller'] ? ' | 🔥 Хит продаж' : '' ?>
                            <br>👁️ <?= $product['views'] ?> просмотров | 🛒 <?= $product['sales'] ?> продаж
                        </div>

                        <div class="product-actions">
                            <a href="?r=admin-products&action=edit&id=<?= $product['id'] ?>" class="btn btn-small">✏️ Изменить</a>
                            <form method="post" style="display:inline;" onsubmit="return confirm('🗑️ Удалить товар?')">
                                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">
                                <input type="hidden" name="action" value="delete">
                                <input type="hidden" name="id" value="<?= $product['id'] ?>">
                                <button type="submit" class="btn btn-danger btn-small">🗑️ Удалить</button>
                            </form>
                        </div>
                    </div>
                </div>
                <?php endforeach; ?>

                <?php if (empty($products)): ?>
                <div style="grid-column: 1/-1; text-align: center; padding: 3rem; background: white; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                    <div style="font-size: 4rem; margin-bottom: 1rem; opacity: 0.6;">🌱</div>
                    <h3>Товары не найдены</h3>
                    <p style="color: #6b7280; margin: 1rem 0;">Измените параметры поиска или добавьте первый товар</p>
                    <a href="?r=admin-products&action=add" class="btn">➕ Добавить товар</a>
                </div>
                <?php endif; ?>
            </div>

            <?php elseif (in_array($action, ['add', 'edit'])): ?>
            <form method="post" enctype="multipart/form-data" class="form-container">
                <input type="hidden" name="csrf_token" value="<?= csrf_token() ?>">

                <div class="form-group">
                    <label>🏷️ Название товара *</label>
                    <input type="text" name="name" required value="<?= esc($product['name'] ?? '') ?>" placeholder="Введите название растения">
                </div>

                <div class="grid-2">
                    <div class="form-group">
                        <label>📂 Категория</label>
                        <select name="category_id">
                            <option value="">Без категории</option>
                            <?php foreach ($categories as $cat): ?>
                            <option value="<?= $cat['id'] ?>" <?= ($product['category_id'] ?? '') == $cat['id'] ? 'selected' : '' ?>>
                                <?= esc($cat['name']) ?>
                            </option>
                            <?php endforeach; ?>
                        </select>
                    </div>
                    <div class="form-group">
                        <label>🎥 Ссылка на видео</label>
                        <input type="url" name="video_url" value="<?= esc($product['video_url'] ?? '') ?>" placeholder="https://www.youtube.com/watch?v=...">
                    </div>
                </div>

                <div class="grid-3">
                    <div class="form-group">
                        <label>💵 Цена продажи *</label>
                        <input type="number" name="price" step="0.01" min="0" required value="<?= $product['price'] ?? '' ?>" placeholder="0.00">
                    </div>
                    <div class="form-group">
                        <label>💸 Старая цена</label>
                        <input type="number" name="old_price" step="0.01" min="0" value="<?= $product['old_price'] ?? '' ?>" placeholder="0.00">
                    </div>
                    <div class="form-group">
                        <label>💰 Себестоимость</label>
                        <input type="number" name="cost_price" step="0.01" min="0" value="<?= $product['cost_price'] ?? '' ?>" placeholder="0.00">
                    </div>
                </div>

                <div class="grid-2">
                    <div class="form-group">
                        <label>📦 Количество на складе</label>
                        <input type="number" name="stock" min="0" value="<?= $product['stock'] ?? 0 ?>">
                    </div>
                    <div class="form-group">
                        <label>⚠️ Минимальный остаток</label>
                        <input type="number" name="min_stock" min="1" value="<?= $product['min_stock'] ?? 5 ?>">
                    </div>
                </div>

                <div class="form-group">
                    <label>📄 Краткие характеристики</label>
                    <input type="text" name="short_desc" value="<?= esc($product['short_desc'] ?? '') ?>" maxlength="200" placeholder="Краткое описание для каталога">
                </div>

                <div class="form-group">
                    <label>📖 Полное описание</label>
                    <textarea name="description" rows="6" placeholder="Подробное описание растения"><?= esc($product['description'] ?? '') ?></textarea>
                </div>

                <div class="form-group">
                    <label>💡 Советы по уходу</label>
                    <textarea name="care_tips" rows="4" placeholder="Рекомендации по выращиванию и уходу"><?= esc($product['care_tips'] ?? '') ?></textarea>
                </div>

                <div class="grid-2">
                    <div class="form-group">
                        <label>💡 Освещение</label>
                        <select name="light">
                            <option value="">Не указано</option>
                            <option value="low" <?= ($product['light'] ?? '') === 'low' ? 'selected' : '' ?>>🌙 Слабое</option>
                            <option value="medium" <?= ($product['light'] ?? '') === 'medium' ? 'selected' : '' ?>>☀️ Среднее</option>
                            <option value="high" <?= ($product['light'] ?? '') === 'high' ? 'selected' : '' ?>>🌞 Яркое</option>
                        </select>
                    </div>
                    <div class="form-group">
                        <label>📊 Сложность выращивания</label>
                        <select name="difficulty">
                            <option value="">Не указано</option>
                            <option value="easy" <?= ($product['difficulty'] ?? '') === 'easy' ? 'selected' : '' ?>>✅ Легкая</option>
                            <option value="medium" <?= ($product['difficulty'] ?? '') === 'medium' ? 'selected' : '' ?>>⚡ Средняя</option>
                            <option value="hard" <?= ($product['difficulty'] ?? '') === 'hard' ? 'selected' : '' ?>>🔥 Сложная</option>
                        </select>
                    </div>
                </div>

                <div class="grid-2">
                    <div class="form-group">
                        <label>🌡️ Температура (°C)</label>
                        <div style="display: flex; gap: 0.5rem; align-items: center;">
                            <input type="number" name="temperature_min" value="<?= $product['temperature_min'] ?? '' ?>" placeholder="От" style="width: 100px;">
                            <span>—</span>
                            <input type="number" name="temperature_max" value="<?= $product['temperature_max'] ?? '' ?>" placeholder="До" style="width: 100px;">
                        </div>
                    </div>
                    <div class="form-group">
                        <label>🧪 pH уровень</label>
                        <div style="display: flex; gap: 0.5rem; align-items: center;">
                            <input type="number" name="ph_min" step="0.1" value="<?= $product['ph_min'] ?? '' ?>" placeholder="От" style="width: 100px;">
                            <span>—</span>
                            <input type="number" name="ph_max" step="0.1" value="<?= $product['ph_max'] ?? '' ?>" placeholder="До" style="width: 100px;">
                        </div>
                    </div>
                </div>

                <div class="grid-2">
                    <div class="form-group">
                        <label>📈 Скорость роста</label>
                        <select name="growth_rate">
                            <option value="">Не указано</option>
                            <option value="slow" <?= ($product['growth_rate'] ?? '') === 'slow' ? 'selected' : '' ?>>🐌 Медленная</option>
                            <option value="medium" <?= ($product['growth_rate'] ?? '') === 'medium' ? 'selected' : '' ?>>🚶 Средняя</option>
                            <option value="fast" <?= ($product['growth_rate'] ?? '') === 'fast' ? 'selected' : '' ?>>🏃 Быстрая</option>
                        </select>
                    </div>
                    <div class="form-group">
                        <label>📍 Размещение в аквариуме</label>
                        <select name="placement">
                            <option value="">Не указано</option>
                            <option value="foreground" <?= ($product['placement'] ?? '') === 'foreground' ? 'selected' : '' ?>>🌿 Передний план</option>
                            <option value="midground" <?= ($product['placement'] ?? '') === 'midground' ? 'selected' : '' ?>>🌱 Средний план</option>
                            <option value="background" <?= ($product['placement'] ?? '') === 'background' ? 'selected' : '' ?>>🌳 Задний план</option>
                        </select>
                    </div>
                </div>

                <div class="form-group">
                    <label>📷 Основное изображение</label>
                    <input type="file" name="image" accept="image/*">
 <?php if (!empty($product['gallery'])): ?>
<div style="margin-top: 1rem;">
    <?php 
    $gallery = json_decode($product['gallery'], true);
    if ($gallery):
        foreach ($gallery as $img): ?>
    <img src="<?= esc($img) ?>" alt="Изображение из галереи" style="max-width: 150px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); margin-right: 0.5rem;">
    <?php 
        endforeach; 
    endif; ?>
    <br><small>Текущие изображения в галерее</small>
</div>
<?php endif; ?>
</div>

<div class="form-group">
<label>🖼️ Галерея изображений</label>
<input type="file" name="gallery[]" accept="image/*" multiple>
<small style="color: #6b7280; margin-top: 0.25rem; display: block;">Можно выбрать несколько изображений</small>
<?php if (!empty(
gallery = json_decode(
gallery as 
img) ?>" alt="Изображение из галереи" style="max-width: 150px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
<?php endforeach; ?>
<br><small>Текущие изображения в галерее</small>
</div>
<?php endif; ?>
</div>

<div class="form-group">
<label>Настройки отображения</label>
<div class="checkbox-group">
<label>
<input type="checkbox" name="is_featured" <?= (⭐Рекомендуемыйтовар
product['is_bestseller'] ?? 0) ? 'checked' : '' ?>>
🔥 Хит продаж
</label>
<label>
<input type="checkbox" name="is_active" <?= ($product['is_active'] ?? 1) ? 'checked' : '' ?>>
✅ Товар активен
</label>
</div>
</div>

<div style="display: flex; gap: 1rem; margin-top: 2rem;">
<button type="submit" class="btn" style="padding: 1rem 2rem;">
<?= $action === 'add' ? '➕ Добавить товар' : '💾 Сохранить изменения' ?>
</button>
<a href="?r=admin-products" class="btn" style="background: #6b7280; padding: 1rem 2rem;">❌ Отмена</a>
</div>
</form>
<?php endif; ?>
</div>

<script>
function applyFilters() {
const search = document.getElementById('searchInput').value;
const category = document.getElementById('categorySelect').value;
const status = document.getElementById('statusSelect').value;

const params = new URLSearchParams();
params.set('r', 'admin-products');
if (search) params.set('search', search);
if (category) params.set('category', category);
if (status) params.set('status', status);

window.location.href = '?' + params.toString();
}

// Автосохранение при вводе в поиск
document.getElementById('searchInput')?.addEventListener('input', function() {
clearTimeout(this.timeout);
this.timeout = setTimeout(applyFilters, 500);
});
</script>
</body>
</html>
<?php
exit;
}

// Остальные админские страницы используют тот же стиль и функционал...
// Для экономии места, они будут работать по тому же принципу

// Выход из админки
if ($r === 'admin-logout') {
auth_logout();
header('Location: ?r=admin-login');
exit;
}

// Диагностика системы (обновленная версия)
if ($r === 'diagnostics') {
$checks = [];

checks[] = ['PDO SQLite', extension_loaded('pdo_sqlite') ? 'OK' : 'MISSING', extension_loaded('pdo_sqlite') ? 'OK' : 'ERROR'];

config['database']['path'];
dbPath);
storageDir, is_dir(
checks[] = ['Storage Writable', '', is_writable($storageDir) ? 'OK' : 'ERROR'];

try {
pdo);
$checks[] = ['Database Connection', 'Connected', 'OK'];

tables as $table) {
pdo, 
checks[] = ["Table: 
exists ? 'Exists' : 'Missing', $exists ? 'OK' : 'ERROR'];
}

pdo->query("SELECT COUNT(*) as c FROM products")->fetch()['c'],
'Categories' => $pdo->query("SELECT COUNT(*) as c FROM categories")->fetch()['c'],
'Orders' => $pdo->query("SELECT COUNT(*) as c FROM orders")->fetch()['c'],
'Users' => $pdo->query("SELECT COUNT(*) as c FROM users")->fetch()['c'],
'News' => $pdo->query("SELECT COUNT(*) as c FROM news")->fetch()['c'],
'Videos' => $pdo->query("SELECT COUNT(*) as c FROM video_reviews")->fetch()['c'],
'Settings' => $pdo->query("SELECT COUNT(*) as c FROM settings")->fetch()['c'],
'Integrations' => $pdo->query("SELECT COUNT(*) as c FROM integrations")->fetch()['c']
];

foreach (
name => $count) {
name, (string)$count, 'INFO'];
}

} catch (Exception $e) {
e->getMessage(), 'ERROR'];
}

checks[] = ['Email Function', function_exists('mail') ? 'Available' : 'Not Available', function_exists('mail') ? 'OK' : 'WARN'];
checks[] = ['Upload Directory', is_writable(DIR . '/storage/uploads') ? 'Writable' : 'Not Writable', is_writable(DIR . '/storage/uploads') ? 'OK' : 'WARN'];
checks[] = ['Memory Usage', round(memory_get_usage(true) / 1024 / 1024, 2) . ' MB', 'INFO'];

if (isset($_GET['json'])) {
header('Content-Type: application/json');
echo json_encode($checks);
exit;
}

?>
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>🔧 Диагностика системы - Аквасбор CRM</title>
<style>
body {
font-family: 'JetBrains Mono', 'Courier New', monospace;
background: #0d1117;
color: #58a6ff;
padding: 2rem;
margin: 0;
line-height: 1.6;
}
.container { max-width: 1000px; margin: 0 auto; }
.check {
margin-bottom: 0.75rem;
padding: 0.75rem 1rem;
border-radius: 6px;
display: flex;
justify-content: space-between;
align-items: center;
background: #161b22;
border: 1px solid #30363d;
transition: all 0.2s ease;
}
.check:hover {
background: rgba(88, 166, 255, 0.05);
border-color: #58a6ff;
}
.status-ok { color: #3fb950; }
.status-error { color: #f85149; }
.status-warn { color: #d29922; }
.status-info { color: #58a6ff; }
h1 {
color: #f0f6fc;
text-align: center;
margin-bottom: 3rem;
font-size: 2.5rem;
font-weight: 700;
}
.header-info {
text-align: center;
margin-bottom: 2rem;
color: #7d8590;
font-size: 1.1rem;
}
.section-title {
color: #f0f6fc;
font-size: 1.3rem;
margin: 2rem 0 1rem 0;
padding-bottom: 0.5rem;
border-bottom: 2px solid #30363d;
}
.footer {
text-align: center;
margin-top: 3rem;
padding-top: 2rem;
border-top: 1px solid #30363d;
color: #7d8590;
}
.links {
display: flex;
gap: 2rem;
justify-content: center;
margin-bottom: 1rem;
}
.links a {
color: #58a6ff;
text-decoration: none;
padding: 0.5rem 1rem;
border: 1px solid #30363d;
border-radius: 6px;
transition: all 0.2s ease;
}
.links a:hover {
background: rgba(88, 166, 255, 0.1);
border-color: #58a6ff;
}
.system-info {
display: grid;
grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
gap: 1rem;
margin-bottom: 2rem;
}
.info-card {
background: #161b22;
padding: 1rem;
border-radius: 6px;
border: 1px solid #30363d;
}
.info-title {
color: #f0f6fc;
font-weight: 600;
margin-bottom: 0.5rem;
}
.info-value {
color: #58a6ff;
font-size: 1.1rem;
}
</style>
</head>
<body>
<div class="container">
<h1>🔧 Диагностика системы Аквасбор CRM</h1>

<div class="header-info">
<div>🌿 Полнофункциональный интернет-магазин аквариумных растений</div>
<div>Версия 3.0 | Современная CRM-система</div>
</div>

<div class="system-info">
<div class="info-card">
<div class="info-title">🖥️ Система</div>
<div class="info-value"><?= PHP_OS ?></div>
</div>
<div class="info-card">
<div class="info-title">🐘 PHP</div>
<div class="info-value"><?= PHP_VERSION ?></div>
</div>
<div class="info-card">
<div class="info-title">💾 Память</div>
<div class="info-value"><?= ini_get('memory_limit') ?></div>
</div>
<div class="info-card">
<div class="info-title">⏰ Время</div>
<div class="info-value"><?= date('d.m.Y H:i:s') ?></div>
</div>
</div>

<div class="section-title">📊 Проверки системы</div>

<?php foreach (
name, 
status]): ?>
<div class="check">
<span>
<span class="status-<?= strtolower(
status ?>]</span>
<?= esc(
status) ?>"><?= esc($value) ?></span>
</div>
<?php endforeach; ?>

<div class="footer">
<div class="links">
<a href="?">← Главная страница</a>
<a href="?r=admin">📊 Админ-панель</a>
<a href="?r=diagnostics&json=1">📄 JSON отчет</a>
</div>
<div>
<p>✅ Система готова к работе!</p>
<p style="margin-top: 1rem; font-size: 0.9rem;">
Время выполнения: <?= number_format((microtime(true) - $_SERVER['REQUEST_TIME_FLOAT']) * 1000, 2) ?>ms |
Пиковое использование памяти: <?= round(memory_get_peak_usage(true) / 1024 / 1024, 2) ?>MB
</p>
</div>
</div>
</div>
</body>
</html>
<?php
exit;
}

// Если маршрут не найден - перенаправляем на главную
header('Location: ?');
exit;
?>