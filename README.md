-- Создание базы данных
CREATE DATABASE partner_requests_db;

-- Подключение к базе данных
\c partner_requests_db;

-- Создание таблицы Partners (Партнеры)
CREATE TABLE partners (
    partner_id SERIAL PRIMARY KEY,
    partner_type VARCHAR(10) NOT NULL CHECK (partner_type IN ('ИП', 'ООО', 'АО', 'ПАО')),
    partner_name VARCHAR(255) NOT NULL,
    director_name VARCHAR(150) NOT NULL,
    director_surname VARCHAR(150) NOT NULL,
    director_patronymic VARCHAR(150),
    legal_address VARCHAR(255) NOT NULL,
    inn VARCHAR(12) UNIQUE NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(255) NOT NULL,
    logo VARCHAR(500),
    rating INT DEFAULT 0 CHECK (rating >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Создание таблицы Products (Продукция)
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    article_number VARCHAR(50) UNIQUE NOT NULL,
    product_type VARCHAR(100) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    description TEXT,
    image_path VARCHAR(500),
    min_price_for_partner DECIMAL(10, 2) NOT NULL CHECK (min_price_for_partner >= 0),
    package_length DECIMAL(10, 2),
    package_width DECIMAL(10, 2),
    package_height DECIMAL(10, 2),
    weight_without_package DECIMAL(10, 3),
    weight_with_package DECIMAL(10, 3),
    certificate_path VARCHAR(500),
    standard_number VARCHAR(100),
    production_time_hours DECIMAL(6, 2),
    cost_price DECIMAL(10, 2) CHECK (cost_price >= 0),
    workshop_number INT,
    workers_count INT DEFAULT 1 CHECK (workers_count > 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Создание таблицы Requests (Заявки)
CREATE TABLE requests (
    request_id SERIAL PRIMARY KEY,
    request_number VARCHAR(50) UNIQUE NOT NULL,
    partner_id INT NOT NULL REFERENCES partners(partner_id) ON DELETE CASCADE,
    request_date DATE NOT NULL DEFAULT CURRENT_DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'Новая' 
        CHECK (status IN ('Новая', 'В обработке', 'Согласована', 'Производство', 
                         'Готова к отгрузке', 'Оплачена', 'Выполнена', 'Отменена')),
    total_cost DECIMAL(12, 2) DEFAULT 0 CHECK (total_cost >= 0),
    prepayment DECIMAL(12, 2) DEFAULT 0 CHECK (prepayment >= 0),
    delivery_required BOOLEAN DEFAULT FALSE,
    delivery_address VARCHAR(255),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Создание таблицы RequestProducts (Продукция в заявке)
CREATE TABLE request_products (
    request_product_id SERIAL PRIMARY KEY,
    request_id INT NOT NULL REFERENCES requests(request_id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL CHECK (unit_price >= 0),
    total_price DECIMAL(12, 2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    production_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (request_id, product_id)
);

-- Создание индексов для оптимизации
CREATE INDEX idx_requests_partner_id ON requests(partner_id);
CREATE INDEX idx_requests_status ON requests(status);
CREATE INDEX idx_request_products_request_id ON request_products(request_id);
CREATE INDEX idx_request_products_product_id ON request_products(product_id);

-- Создание триггера для обновления total_cost в requests
CREATE OR REPLACE FUNCTION update_request_total_cost()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE requests 
    SET total_cost = (
        SELECT COALESCE(SUM(total_price), 0) 
        FROM request_products 
        WHERE request_id = COALESCE(NEW.request_id, OLD.request_id)
    ),
    updated_at = CURRENT_TIMESTAMP
    WHERE request_id = COALESCE(NEW.request_id, OLD.request_id);
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_request_total_cost
AFTER INSERT OR UPDATE OR DELETE ON request_products
FOR EACH ROW
EXECUTE FUNCTION update_request_total_cost();

-- Функция для автоматической отмены заявок без предоплаты через 3 дня
CREATE OR REPLACE FUNCTION auto_cancel_unpaid_requests()
RETURNS void AS $$
BEGIN
    UPDATE requests 
    SET status = 'Отменена',
        updated_at = CURRENT_TIMESTAMP
    WHERE status = 'Согласована' 
      AND prepayment = 0
      AND created_at < CURRENT_TIMESTAMP - INTERVAL '3 days';
END;
$$ LANGUAGE plpgsql;

-- Тестовые данные для импорта
INSERT INTO partners (partner_type, partner_name, director_name, director_surname, director_patronymic, 
                      legal_address, inn, phone, email, rating) VALUES
('ООО', 'СтройМаркет', 'Иван', 'Петров', 'Сергеевич', 'г. Москва, ул. Ленина, 10', '7701234567', '+7(495)123-45-67', 'info@stroymarket.ru', 8),
('ИП', 'РемонтПлюс', 'Анна', 'Смирнова', 'Ивановна', 'г. Санкт-Петербург, пр. Невский, 25', '7809876543', '+7(812)987-65-43', 'anna@remontplus.ru', 6),
('ООО', 'ДомСтрой', 'Петр', 'Иванов', 'Александрович', 'г. Казань, ул. Баумана, 5', '1601234567', '+7(843)111-22-33', 'info@domstroy.ru', 7),
('АО', 'СтройГарант', 'Мария', 'Козлова', 'Дмитриевна', 'г. Новосибирск, ул. Мира, 15', '5409876543', '+7(383)555-66-77', 'office@stroygarant.ru', 9),
('ООО', 'ОтделкаПро', 'Сергей', 'Волков', 'Петрович', 'г. Екатеринбург, ул. Ленина, 30', '6601234567', '+7(343)222-33-44', 'sale@otdelkapro.ru', 5);

INSERT INTO products (article_number, product_type, product_name, description, min_price_for_partner,
                     package_length, package_width, package_height, weight_without_package, 
                     weight_with_package, standard_number, production_time_hours, cost_price, 
                     workshop_number, workers_count) VALUES
('ART-001', 'Краска', 'Краска акриловая белая', 'Акриловая краска для стен и потолков', 250.00, 20, 20, 25, 2.5, 2.7, 'ГОСТ 28196-89', 2.5, 150.00, 1, 3),
('ART-002', 'Краска', 'Краска латексная бежевая', 'Латексная краска для интерьеров', 320.00, 20, 20, 25, 2.8, 3.0, 'ГОСТ 28196-89', 3.0, 200.00, 1, 3),
('ART-003', 'Обои', 'Обои флизелиновые', 'Флизелиновые обои с рисунком', 450.00, 106, 15, 15, 1.2, 1.5, 'ГОСТ 6810-2002', 1.5, 280.00, 2, 4),
('ART-004', 'Обои', 'Обои виниловые', 'Виниловые обои на флизелиновой основе', 380.00, 106, 15, 15, 1.0, 1.3, 'ГОСТ 6810-2002', 1.5, 230.00, 2, 4),
('ART-005', 'Ламинат', 'Ламинат дуб натуральный', 'Ламинат 33 класса износостойкости', 850.00, 120, 19, 7, 14.0, 14.5, 'ГОСТ 32304-2013', 4.0, 520.00, 3, 5),
('ART-006', 'Плитка', 'Плитка керамическая', 'Керамическая плитка для пола', 720.00, 30, 30, 8, 2.0, 2.2, 'ГОСТ 6787-2001', 3.5, 430.00, 4, 6),
('ART-007', 'Штукатурка', 'Штукатурка декоративная', 'Декоративная штукатурка для стен', 290.00, 50, 40, 30, 15.0, 15.5, 'ГОСТ 33083-2014', 2.0, 175.00, 5, 2);

INSERT INTO requests (request_number, partner_id, request_date, status, prepayment, delivery_required, delivery_address) VALUES
('REQ-2024001', 1, '2024-01-15', 'Выполнена', 50000.00, TRUE, 'г. Москва, ул. Тверская, 1'),
('REQ-2024002', 2, '2024-01-20', 'Согласована', 25000.00, FALSE, NULL),
('REQ-2024003', 3, '2024-02-01', 'В обработке', 0.00, TRUE, 'г. Казань, ул. Пушкина, 10'),
('REQ-2024004', 1, '2024-02-10', 'Новая', 0.00, FALSE, NULL),
('REQ-2024005', 4, '2024-02-15', 'Согласована', 75000.00, TRUE, 'г. Новосибирск, ул. Ленина, 20'),
('REQ-2024006', 5, '2024-02-20', 'Отменена', 0.00, FALSE, NULL),
('REQ-2024007', 2, '2024-03-01', 'Производство', 30000.00, TRUE, 'г. Санкт-Петербург, Невский пр., 15');

INSERT INTO request_products (request_id, product_id, quantity, unit_price) VALUES
(1, 1, 50, 250.00),
(1, 3, 30, 450.00),
(2, 2, 40, 320.00),
(3, 5, 20, 850.00),
(3, 6, 25, 720.00),
(4, 1, 10, 250.00),
(5, 4, 35, 380.00),
(5, 7, 60, 290.00),
(7, 2, 25, 320.00),
(7, 3, 20, 450.00);
