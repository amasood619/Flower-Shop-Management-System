# Flower-Shop-Management-System
MySQL database design and implementation for a flower shop management system. Includes normalized schema, foreign key constraints, inventory triggers, and supporting design documentation (ERD and data dictionary) demonstrating design-to-implementation traceability.


-- =========================
-- TABLE: Customer
-- =========================
CREATE TABLE Customer (
  CustomerID INT NOT NULL,
  FirstName  VARCHAR(25) NOT NULL,
  LastName   VARCHAR(25) NOT NULL,
  Email      VARCHAR(50) NOT NULL,
  Phone      VARCHAR(14) NULL,
  Street     VARCHAR(60) NOT NULL,
  City       VARCHAR(25) NOT NULL,
  State      VARCHAR(25) NOT NULL,
  Zip        VARCHAR(10) NOT NULL,
  CONSTRAINT PK_Customer PRIMARY KEY (CustomerID),
  CONSTRAINT UQ_Customer_Email UNIQUE (Email)
) ENGINE=InnoDB;

-- =========================
-- TABLE: Employee
-- =========================
CREATE TABLE Employee (
  EmployeeID INT NOT NULL,
  FirstName  VARCHAR(25) NOT NULL,
  LastName   VARCHAR(25) NOT NULL,
  Role       VARCHAR(40) NOT NULL,
  Phone      VARCHAR(14) NULL,
  CONSTRAINT PK_Employee PRIMARY KEY (EmployeeID)
) ENGINE=InnoDB;

-- =========================
-- TABLE: Supplier
-- =========================
CREATE TABLE Supplier (
  SupplierID INT NOT NULL,
  Name       VARCHAR(50) NOT NULL,
  Contact    VARCHAR(50) NULL,
  Phone      VARCHAR(14) NULL,
  Address    VARCHAR(100) NULL,
  CONSTRAINT PK_Supplier PRIMARY KEY (SupplierID)
) ENGINE=InnoDB;

-- =========================
-- TABLE: Flower
-- =========================
CREATE TABLE Flower (
  FlowerID         INT NOT NULL,
  SupplierID       INT NOT NULL,
  Name             VARCHAR(50) NOT NULL,
  Type             VARCHAR(30) NOT NULL,
  Price            DECIMAL(9,2) NOT NULL,
  QuantityInStock  INT NOT NULL,
  CONSTRAINT PK_Flower PRIMARY KEY (FlowerID),
  CONSTRAINT FK_Flower_Supplier FOREIGN KEY (SupplierID)
    REFERENCES Supplier(SupplierID)
    ON UPDATE CASCADE
    ON DELETE RESTRICT,
  CONSTRAINT CHK_Flower_Price CHECK (Price >= 0),
  CONSTRAINT CHK_Flower_Quantity CHECK (QuantityInStock >= 0)
) ENGINE=InnoDB;

CREATE INDEX IX_Flower_SupplierID ON Flower(SupplierID);

-- =========================
-- TABLE: `Order`
-- NOTE: Order is reserved-ish; keep as `Order` with backticks in MySQL.
-- =========================
CREATE TABLE `Order` (
  OrderID      INT NOT NULL,
  CustomerID   INT NOT NULL,
  EmployeeID   INT NOT NULL,
  OrderDate    DATE NOT NULL,
  Status       VARCHAR(20) NOT NULL,
  TotalAmount  DECIMAL(9,2) NOT NULL,
  CONSTRAINT PK_Order PRIMARY KEY (OrderID),
  CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerID)
    REFERENCES Customer(CustomerID)
    ON UPDATE CASCADE
    ON DELETE RESTRICT,
  CONSTRAINT FK_Order_Employee FOREIGN KEY (EmployeeID)
    REFERENCES Employee(EmployeeID)
    ON UPDATE CASCADE
    ON DELETE RESTRICT,
  CONSTRAINT CHK_Order_TotalAmount CHECK (TotalAmount >= 0),
  CONSTRAINT CHK_Order_Status CHECK (Status IN ('Pending','Prepared','Delivered','Cancelled'))
) ENGINE=InnoDB;

CREATE INDEX IX_Order_CustomerID ON `Order`(CustomerID);
CREATE INDEX IX_Order_EmployeeID ON `Order`(EmployeeID);
CREATE INDEX IX_Order_OrderDate ON `Order`(OrderDate);

-- =========================
-- TABLE: Payment
-- =========================
CREATE TABLE Payment (
  PaymentID  INT NOT NULL,
  OrderID    INT NOT NULL,
  Method     VARCHAR(20) NOT NULL,
  Amount     DECIMAL(9,2) NOT NULL,
  PaidDate   DATE NOT NULL,
  CONSTRAINT PK_Payment PRIMARY KEY (PaymentID),
  CONSTRAINT FK_Payment_Order FOREIGN KEY (OrderID)
    REFERENCES `Order`(OrderID)
    ON UPDATE CASCADE
    ON DELETE RESTRICT,
  CONSTRAINT UQ_Payment_Order UNIQUE (OrderID),
  CONSTRAINT CHK_Payment_Amount CHECK (Amount >= 0),
  CONSTRAINT CHK_Payment_Method CHECK (Method IN ('Cash','Card','Online'))
) ENGINE=InnoDB;

CREATE INDEX IX_Payment_OrderID ON Payment(OrderID);

-- =========================
-- TABLE: Order_Flower (bridge / line items)
-- =========================
CREATE TABLE Order_Flower (
  OrderID    INT NOT NULL,
  FlowerID   INT NOT NULL,
  Quantity   INT NOT NULL,
  UnitPrice  DECIMAL(9,2) NOT NULL,
  CONSTRAINT PK_Order_Flower PRIMARY KEY (OrderID, FlowerID),
  CONSTRAINT FK_OrderFlower_Order FOREIGN KEY (OrderID)
    REFERENCES `Order`(OrderID)
    ON UPDATE CASCADE
    ON DELETE CASCADE,
  CONSTRAINT FK_OrderFlower_Flower FOREIGN KEY (FlowerID)
    REFERENCES Flower(FlowerID)
    ON UPDATE CASCADE
    ON DELETE RESTRICT,
  CONSTRAINT CHK_OrderFlower_Quantity CHECK (Quantity > 0),
  CONSTRAINT CHK_OrderFlower_UnitPrice CHECK (UnitPrice >= 0)
) ENGINE=InnoDB;

CREATE INDEX IX_OrderFlower_FlowerID ON Order_Flower(FlowerID);

-- =========================
-- TRIGGERS: Inventory update + stock validation
-- Business rule: Inventory updates automatically after each sale.
-- =========================
DELIMITER $$

-- Prevent selling more than you have
CREATE TRIGGER trg_orderflower_before_insert_stockcheck
BEFORE INSERT ON Order_Flower
FOR EACH ROW
BEGIN
  DECLARE current_stock INT;

  SELECT QuantityInStock
    INTO current_stock
  FROM Flower
  WHERE FlowerID = NEW.FlowerID
  FOR UPDATE;

  IF current_stock IS NULL THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Invalid FlowerID (not found).';
  END IF;

  IF NEW.Quantity > current_stock THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Insufficient stock for requested flower quantity.';
  END IF;
END$$

-- Deduct stock after line item is created
CREATE TRIGGER trg_orderflower_after_insert_decrement_stock
AFTER INSERT ON Order_Flower
FOR EACH ROW
BEGIN
  UPDATE Flower
  SET QuantityInStock = QuantityInStock - NEW.Quantity
  WHERE FlowerID = NEW.FlowerID;
END$$

DELIMITER ;
