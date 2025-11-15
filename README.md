# -Oracle-Shop-Database-Projekt
Projekt przedstawia kompletną bazę danych sklepu internetowego opartą o **Oracle Database**.   Repozytorium zawiera pełny model relacyjny, skrypty SQL, procedury PL/SQL, triggery oraz przykładowe dane.


# 🛒 Oracle Shop Database — Projekt Portfolio

Projekt przedstawia kompletną bazę danych sklepu internetowego opartą o **Oracle Database**.  
Repozytorium zawiera pełny model relacyjny, skrypty SQL, procedury PL/SQL, triggery oraz przykładowe dane.

## 📌 Zawartość projektu

### 1. Struktura bazy danych
- Tabele (klienci, produkty, zamówienia itd.)
- Klucze główne i obce
- Ograniczenia integralności
- Sekwencje
- Widoki i indeksy

### 2. Logika biznesowa PL/SQL
- Procedura składania zamówienia
- Funkcja licząca wartość zamówienia
- Trigger audytowy zapisujący operacje na wrażliwych tabelach

### 3. Przykładowe dane
- Lista produktów
- Klienci
- Zamówienia testowe

---

## 📊 ERD (Entity Relationship Diagram)
+------------------+         +------------------+
|    CUSTOMERS     |         |     ORDERS       |
+------------------+         +------------------+
| PK customer_id   | 1     ∞ | PK order_id      |
| first_name       |---------| FK customer_id   |
| last_name        |         | order_date       |
| email            |         +------------------+
+------------------+                  |
                                       | 1
                                       |  
                                       | ∞
                             +-----------------------+
                             |     ORDER_ITEMS       |
                             +-----------------------+
                             | PK item_id            |
                             | FK order_id           |
                             | FK product_id         |
                             | quantity              |
                             +----------+------------+
                                        |
                                        | ∞  
                                        | 1
                             +-----------------------+
                             |       PRODUCTS        |
                             +-----------------------+
                             | PK product_id         |
                             | name                  |
                             | price                 |
                             +-----------------------+

erDiagram

    CUSTOMERS {
        NUMBER customer_id PK
        VARCHAR first_name
        VARCHAR last_name
        VARCHAR email
    }

    ORDERS {
        NUMBER order_id PK
        NUMBER customer_id FK
        DATE order_date
    }

    ORDER_ITEMS {
        NUMBER item_id PK
        NUMBER order_id FK
        NUMBER product_id FK
        NUMBER quantity
    }

    PRODUCTS {
        NUMBER product_id PK
        VARCHAR name
        NUMBER price
    }

    CUSTOMERS ||--o{ ORDERS : "places"
    ORDERS ||--o{ ORDER_ITEMS : "contains"
    PRODUCTS ||--o{ ORDER_ITEMS : "included in"

---

## ▶️ Instalacja i uruchomienie

1. Uruchom wszystkie skrypty w kolejności:
   ```sql
   @schema/01_tables.sql
   @schema/02_constraints.sql
   @schema/03_sequences.sql
   @data/insert_data.sql
   @procedures/add_order.sql
   @procedures/calculate_order_value.sql
   @procedures/audit_trigger.sql
   ```

2. Przykład wywołania procedury:
   ```sql
   EXEC add_order(1, 3, 2);
   ```

---

## 🧩 Technologie
- **Oracle Database 19c+**
- **PL/SQL**
- Modelowanie ERD

---

## 👤 Autor
Projekt przygotowany jako część portfolio programistycznego.  
```  

---

# 📦 **Skrypty SQL do repozytorium**

## `01_tables.sql`

```sql
CREATE TABLE customers (
    customer_id     NUMBER PRIMARY KEY,
    first_name      VARCHAR2(50),
    last_name       VARCHAR2(50),
    email           VARCHAR2(100) UNIQUE NOT NULL
);

CREATE TABLE products (
    product_id      NUMBER PRIMARY KEY,
    name            VARCHAR2(100) NOT NULL,
    price           NUMBER(10,2) NOT NULL
);

CREATE TABLE orders (
    order_id        NUMBER PRIMARY KEY,
    customer_id     NUMBER NOT NULL,
    order_date      DATE DEFAULT SYSDATE,
    CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    item_id         NUMBER PRIMARY KEY,
    order_id        NUMBER NOT NULL,
    product_id      NUMBER NOT NULL,
    quantity        NUMBER NOT NULL,
    CONSTRAINT fk_items_order FOREIGN KEY (order_id)
        REFERENCES orders(order_id),
    CONSTRAINT fk_items_product FOREIGN KEY (product_id)
        REFERENCES products(product_id)
);
```

---

## `02_constraints.sql`

```sql
ALTER TABLE products 
    ADD CONSTRAINT chk_price CHECK (price > 0);

ALTER TABLE order_items
    ADD CONSTRAINT chk_quantity CHECK (quantity > 0);
```

---

## `03_sequences.sql`

```sql
CREATE SEQUENCE seq_customers START WITH 1;
CREATE SEQUENCE seq_products START WITH 1;
CREATE SEQUENCE seq_orders START WITH 1;
CREATE SEQUENCE seq_order_items START WITH 1;
```

---

## `insert_data.sql`

```sql
INSERT INTO customers VALUES (seq_customers.NEXTVAL, 'Jan', 'Kowalski', 'jan.kowalski@example.com');
INSERT INTO customers VALUES (seq_customers.NEXTVAL, 'Anna', 'Nowak', 'anna.nowak@example.com');

INSERT INTO products VALUES (seq_products.NEXTVAL, 'Laptop Lenovo', 3499.99);
INSERT INTO products VALUES (seq_products.NEXTVAL, 'Mysz Logitech', 59.99);
INSERT INTO products VALUES (seq_products.NEXTVAL, 'Monitor Samsung', 899.00);

COMMIT;
```

---

## `add_order.sql`

```sql
CREATE OR REPLACE PROCEDURE add_order(
    p_customer_id NUMBER,
    p_product_id  NUMBER,
    p_quantity    NUMBER
) AS
    v_order_id NUMBER;
BEGIN
    v_order_id := seq_orders.NEXTVAL;

    INSERT INTO orders(order_id, customer_id)
    VALUES (v_order_id, p_customer_id);

    INSERT INTO order_items(item_id, order_id, product_id, quantity)
    VALUES (seq_order_items.NEXTVAL, v_order_id, p_product_id, p_quantity);

    COMMIT;
END;
/
```

---

## `calculate_order_value.sql`

```sql
CREATE OR REPLACE FUNCTION calculate_order_value(p_order_id NUMBER)
RETURN NUMBER AS
    total NUMBER;
BEGIN
    SELECT SUM(oi.quantity * p.price)
    INTO total
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    WHERE oi.order_id = p_order_id;

    RETURN NVL(total, 0);
END;
/
```

---

## `audit_trigger.sql`

```sql
CREATE TABLE audit_log (
    action_id      NUMBER PRIMARY KEY,
    table_name     VARCHAR2(30),
    operation      VARCHAR2(10),
    change_date    DATE,
    user_name      VARCHAR2(30)
);

CREATE SEQUENCE seq_audit START WITH 1;

CREATE OR REPLACE TRIGGER trg_order_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW
BEGIN
    INSERT INTO audit_log
    VALUES (
        seq_audit.NEXTVAL,
        'ORDERS',
        CASE 
            WHEN INSERTING THEN 'INSERT'
            WHEN UPDATING THEN 'UPDATE'
            WHEN DELETING THEN 'DELETE'
        END,
        SYSDATE,
        USER
    );
END;
/
