# Database Design and Implementation (SQL Server)

A production-grade relational banking database system demonstrating comprehensive database lifecycle management—from conceptual modeling to implementation, testing, and deployment using Microsoft SQL Server.

[![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=flat&logo=microsoft-sql-server&logoColor=white)](https://www.microsoft.com/en-us/sql-server)
[![T-SQL](https://img.shields.io/badge/T--SQL-316192?style=flat&logo=postgresql&logoColor=white)](https://docs.microsoft.com/en-us/sql/t-sql/)

## 📋 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Technologies Used](#technologies-used)
- [Database Architecture](#database-architecture)
- [Schema Design](#schema-design)
- [Business Logic Implementation](#business-logic-implementation)
- [Data Integrity & Constraints](#data-integrity--constraints)
- [Triggers & Automation](#triggers--automation)
- [Transaction Management](#transaction-management)
- [Testing Strategy](#testing-strategy)
- [Installation & Setup](#installation--setup)
- [Usage Examples](#usage-examples)
- [Skills Demonstrated](#skills-demonstrated)
- [Future Enhancements](#future-enhancements)
- [Contributing](#contributing)
- [License](#license)

## 🎯 Overview

This project implements a comprehensive banking database system that handles customer management, account operations, financial transactions, auditing, and reporting. The system emphasizes **data integrity**, **business rule enforcement**, and **audit compliance** through database-level constraints and stored procedures.

### Project Objectives

- Design a normalized, scalable relational database schema
- Implement robust business logic using T-SQL stored procedures
- Enforce data integrity through constraints and triggers
- Provide comprehensive transaction management and error handling
- Enable audit tracking for regulatory compliance
- Demonstrate production-ready testing practices

## ✨ Key Features

- **Customer Management**: Registration, profile updates, and duplicate detection
- **Account Operations**: Account creation, balance tracking, and status management
- **Transaction Processing**: Deposits, withdrawals, and inter-account transfers
- **Audit Logging**:  Automated tracking of critical data changes
- **Data Integrity**: Multi-layered validation and constraint enforcement
- **Reporting**: Aggregated views and analytical queries
- **Error Handling**: Graceful failure management with rollback support

## 🛠 Technologies Used

| Technology | Purpose |
|------------|---------|
| **Microsoft SQL Server** | Relational database management system |
| **T-SQL** | Data definition, manipulation, and procedural logic |
| **Stored Procedures** | Business logic encapsulation |
| **Triggers** | Automated business rule enforcement |
| **ERD/UML** | Database modeling and documentation |

## 🏗 Database Architecture

### Conceptual Model

The system follows a **normalized relational design** (3NF) to minimize redundancy and ensure data consistency. 

### Core Entities

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  Customer   │──────<│   Account   │>──────│ Transaction │
└─────────────┘       └─────────────┘       └─────────────┘
       │                                            │
       │                                            │
       ▼                                            ▼
┌──────────────────┐                  ┌────────────────────┐
│CustomerAddressLog│                  │TransactionAccount  │
│   (Audit Table)  │                  │TransactionComment  │
└──────────────────┘                  └────────────────────┘
```

### Entity Descriptions

| Entity | Description |
|--------|-------------|
| **Customer** | Stores customer personal information and identity data |
| **Account** | Manages account details, types, and balances |
| **Transaction** | Records all financial transactions with timestamps |
| **TransactionAccount** | Junction table linking transactions to accounts |
| **TransactionComment** | Stores optional metadata for transactions |
| **CustomerAddressLog** | Audit trail for customer address changes |

## 📐 Schema Design

### Data Integrity Constraints

```sql
-- Example constraint implementations: 

-- 1. PRIMARY KEY & IDENTITY
CREATE TABLE Customer (
    CustomerID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(50) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    -- ... 
);

-- 2. FOREIGN KEY relationships
ALTER TABLE Account 
ADD CONSTRAINT FK_Account_Customer 
FOREIGN KEY (CustomerID) REFERENCES Customer(CustomerID);

-- 3. CHECK constraints for business rules
ALTER TABLE Account 
ADD CONSTRAINT CHK_Account_Balance 
CHECK (Balance >= 0);

-- 4. UNIQUE constraints
ALTER TABLE Customer 
ADD CONSTRAINT UQ_Customer_SSN 
UNIQUE (SSN);
```

### Constraint Types

- **PRIMARY KEY**:  Unique identifier for each entity
- **FOREIGN KEY**:  Referential integrity across tables
- **UNIQUE**: Prevents duplicate customer identities
- **CHECK**: Business rule validation
  - Valid customer types (Individual, Business)
  - Valid account types (Checking, Savings, Investment)
  - Valid transaction types (Deposit, Withdrawal, Transfer)
  - Non-negative balances
  - Valid date ranges
- **IDENTITY**: Auto-generated surrogate keys

## 💼 Business Logic Implementation

### Stored Procedures

#### Customer Management

| Procedure | Purpose |
|-----------|---------|
| `sp_CreateCustomer` | Register new customer with validation |
| `sp_UpdateCustomerAddress` | Update address with audit logging |
| `sp_GetCustomersByCity` | Retrieve customers by location |

#### Account Management

| Procedure | Purpose |
|-----------|---------|
| `sp_CreateAccount` | Create new account linked to customer |
| `sp_GetCustomerTotalBalance` | Calculate total balance across accounts |
| `sp_ValidateAccountOwnership` | Verify customer-account relationship |

#### Transaction Processing

| Procedure | Purpose |
|-----------|---------|
| `sp_ProcessDeposit` | Handle deposit transactions with validation |
| `sp_ProcessWithdrawal` | Handle withdrawals with balance checks |
| `sp_ProcessTransfer` | Handle inter-account transfers atomically |

### Example Stored Procedure

```sql
CREATE PROCEDURE sp_ProcessDeposit
    @AccountID INT,
    @Amount DECIMAL(18,2),
    @TransactionComment NVARCHAR(200) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Validation
        IF @Amount <= 0
            RAISERROR('Deposit amount must be positive', 16, 1);
        
        -- Insert transaction
        INSERT INTO [Transaction] (TransactionType, Amount, TransactionDate)
        VALUES ('Deposit', @Amount, GETDATE());
        
        -- Link to account
        INSERT INTO TransactionAccount (TransactionID, AccountID, TransactionType)
        VALUES (SCOPE_IDENTITY(), @AccountID, 'TO');
        
        -- Update balance (handled by trigger)
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
```

## 🔒 Data Integrity & Constraints

### Multi-Layered Validation

1. **Database Level**: CHECK constraints, NOT NULL, FOREIGN KEY
2. **Trigger Level**: Balance validation, audit logging
3. **Procedure Level**: Business logic validation, duplicate detection
4. **Transaction Level**:  ACID compliance, rollback on error

### Constraint Benefits

- **Prevents invalid data** at the database level
- **Application-independent** validation
- **Consistent enforcement** across all access methods
- **Performance optimization** through constraint indexing

## ⚡ Triggers & Automation

### Implemented Triggers

| Trigger | Event | Purpose |
|---------|-------|---------|
| `trg_PreventNegativeBalance` | BEFORE UPDATE | Prevents negative account balances |
| `trg_AuditAddressChange` | AFTER UPDATE | Logs customer address changes |
| `trg_UpdateAccountBalance` | AFTER INSERT | Automatically updates balances from transactions |

### Example Trigger

```sql
CREATE TRIGGER trg_PreventNegativeBalance
ON Account
INSTEAD OF UPDATE
AS
BEGIN
    IF EXISTS (SELECT 1 FROM inserted WHERE Balance < 0)
    BEGIN
        RAISERROR('Account balance cannot be negative', 16, 1);
        ROLLBACK TRANSACTION;
    END
    ELSE
    BEGIN
        UPDATE a
        SET a.Balance = i.Balance,
            a.UpdatedDate = GETDATE()
        FROM Account a
        INNER JOIN inserted i ON a.AccountID = i. AccountID;
    END
END;
```

## 🔄 Transaction Management

### ACID Compliance

- **Atomicity**:  All operations complete or none do (via BEGIN/COMMIT/ROLLBACK)
- **Consistency**: Database remains in valid state (via constraints)
- **Isolation**: Concurrent transactions don't interfere (via isolation levels)
- **Durability**: Committed changes persist (via transaction log)

### Error Handling Pattern

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    
    -- Business logic here
    
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    -- Custom error handling
    DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
    RAISERROR(@ErrorMessage, 16, 1);
END CATCH
```

## 🧪 Testing Strategy

### Test Coverage

1. **Positive Test Cases**:  Valid operations that should succeed
2. **Negative Test Cases**: Invalid operations that should fail gracefully
3. **Edge Cases**:  Boundary conditions and special scenarios
4. **Performance Tests**: Load testing and optimization validation

### Test Scenarios

| Category | Test Cases |
|----------|------------|
| **Customer** | Valid creation, duplicate detection, invalid data |
| **Account** | Account creation, invalid ownership, status checks |
| **Transaction** | Deposits, withdrawals, transfers, overdrafts, invalid amounts |
| **Audit** | Address change logging, timestamp accuracy |
| **Constraints** | Negative balance prevention, referential integrity |

### Test Script Example

```sql
-- Test: Valid deposit
EXEC sp_ProcessDeposit @AccountID = 1, @Amount = 500.00;
SELECT Balance FROM Account WHERE AccountID = 1; -- Verify balance increased

-- Test: Invalid withdrawal (overdraft)
EXEC sp_ProcessWithdrawal @AccountID = 1, @Amount = 999999. 99;
-- Expected: Error raised, balance unchanged
SELECT Balance FROM Account WHERE AccountID = 1; -- Verify balance unchanged
```

## 📦 Installation & Setup

### Prerequisites

- Microsoft SQL Server 2016 or higher
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Setup Instructions

1. **Clone the repository**
   ```bash
   git clone https://github.com/oyebiyisunday/Database-Design-and-Implementation-.git
   cd Database-Design-and-Implementation-
   ```

2. **Create the database**
   ```sql
   -- Execute in SSMS
   CREATE DATABASE BankingDB;
   GO
   USE BankingDB;
   GO
   ```

3. **Run schema scripts** (in order)
   ```sql
   -- 1. Create tables
   : r scripts/01_CreateTables.sql
   
   -- 2. Create constraints
   :r scripts/02_CreateConstraints.sql
   
   -- 3. Create stored procedures
   :r scripts/03_CreateProcedures.sql
   
   -- 4. Create triggers
   :r scripts/04_CreateTriggers.sql
   
   -- 5. Insert sample data
   :r scripts/05_InsertSampleData. sql
   ```

4. **Run tests**
   ```sql
   : r tests/TestSuite.sql
   ```

## 📝 Usage Examples

### Create a Customer

```sql
EXEC sp_CreateCustomer 
    @FirstName = 'John',
    @LastName = 'Doe',
    @Email = 'john.doe@example.com',
    @Phone = '555-0123',
    @Address = '123 Main St',
    @City = 'New York',
    @State = 'NY',
    @ZipCode = '10001',
    @CustomerType = 'Individual';
```

### Create an Account

```sql
EXEC sp_CreateAccount
    @CustomerID = 1,
    @AccountType = 'Checking',
    @InitialBalance = 1000.00;
```

### Process a Transfer

```sql
EXEC sp_ProcessTransfer
    @FromAccountID = 1,
    @ToAccountID = 2,
    @Amount = 250.00,
    @TransactionComment = 'Rent payment';
```

### Generate Reports

```sql
-- Customer count by city
EXEC sp_GetCustomersByCity @City = 'New York';

-- Total balance per customer
EXEC sp_GetCustomerTotalBalance @CustomerID = 1;
```

## 🎓 Skills Demonstrated

### Database Design & Modeling
- ✅ ERD/UML conceptual modeling
- ✅ Normalization (3NF)
- ✅ Relational schema design
- ✅ Entity-relationship mapping

### T-SQL Programming
- ✅ Data Definition Language (DDL)
- ✅ Data Manipulation Language (DML)
- ✅ Stored procedures with parameters
- ✅ User-defined functions
- ✅ Triggers (BEFORE, AFTER, INSTEAD OF)
- ✅ Dynamic SQL (where applicable)

### Data Integrity & Quality
- ✅ Constraint design and implementation
- ✅ Referential integrity enforcement
- ✅ Business rule validation
- ✅ Audit trail implementation

### Transaction Management
- ✅ ACID compliance
- ✅ Transaction control (BEGIN, COMMIT, ROLLBACK)
- ✅ Error handling (TRY... CATCH)
- ✅ Isolation level management

### Testing & Quality Assurance
- ✅ Test script development
- ✅ Positive/negative testing
- ✅ Edge case identification
- ✅ Data validation queries

### Performance & Optimization
- ✅ Index strategy
- ✅ Query optimization
- ✅ Execution plan analysis

## 🚀 Future Enhancements

- [ ] Implement role-based access control (RBAC)
- [ ] Add stored procedure for loan management
- [ ] Create views for common reporting queries
- [ ] Implement encryption for sensitive data (SSN, account numbers)
- [ ] Add support for international transactions
- [ ] Integrate with external audit logging system
- [ ] Implement CDC (Change Data Capture) for advanced auditing
- [ ] Add performance monitoring and alerting
- [ ] Create API layer for application integration
- [ ] Develop automated backup and recovery procedures

## 🤝 Contributing

Contributions are welcome! Please follow these guidelines:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/YourFeature`)
3. Commit your changes (`git commit -m 'Add YourFeature'`)
4. Push to the branch (`git push origin feature/YourFeature`)
5. Open a Pull Request

### Contribution Areas

- Additional test cases
- Performance optimizations
- Documentation improvements
- New stored procedures or features
- Bug fixes

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 📞 Contact

**Sunday Oyebiyi**

- GitHub: [@oyebiyisunday](https://github.com/oyebiyisunday)
- LinkedIn: [Sunday Oyebiyi](https://www.linkedin.com/in/sunday-oyebiyi/))
- Email: oyebiyisunday@gmail.com

---

<div
