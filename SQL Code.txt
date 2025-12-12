/*===============================================================
    BANKDB PROJECT BY SUNDAY OYEBIYI
===============================================================*/

SET ANSI_NULLS ON;
SET QUOTED_IDENTIFIER ON;
GO

/*===============================================================
    1. TABLES
===============================================================*/

-- 1.1 CUSTOMER
CREATE TABLE dbo.Customer (
    customer_id      INT IDENTITY(1,1) PRIMARY KEY,
    first_name       VARCHAR(30) NOT NULL,
    middle_initial   CHAR(1) NULL,
    last_name        VARCHAR(50) NOT NULL,
    phone_number     VARCHAR(20) NOT NULL,
    street           VARCHAR(120) NOT NULL,
    city             VARCHAR(60) NOT NULL,
    state            VARCHAR(60) NOT NULL,
    postal_code      VARCHAR(15) NOT NULL,
    country          VARCHAR(60) NOT NULL,
    customer_type    VARCHAR(20) NOT NULL,
    CONSTRAINT CK_Customer_Type
        CHECK (customer_type IN ('PERSONAL','BUSINESS')),
    CONSTRAINT UQ_Customer_NamePhone
        UNIQUE (first_name,last_name,phone_number)
);
GO

-- 1.2 ACCOUNT
CREATE TABLE dbo.Account (
    account_id       INT IDENTITY(1,1) PRIMARY KEY,
    customer_id      INT NOT NULL,
    account_type     VARCHAR(20) NOT NULL,
    date_opened      DATE NOT NULL
        CONSTRAINT DF_Account_DateOpened DEFAULT (CAST(GETDATE() AS DATE)),
    account_balance  NUMERIC(14,2) NOT NULL
        CONSTRAINT DF_Account_Balance DEFAULT (0),
    CONSTRAINT FK_Account_Customer
        FOREIGN KEY (customer_id) REFERENCES dbo.Customer(customer_id),
    CONSTRAINT CK_Account_Type
        CHECK (account_type IN ('CHECKING','SAVINGS','STOCK')),
    CONSTRAINT CK_Account_DateOpened
        CHECK (date_opened >= '1910-01-01' AND date_opened <= CAST(GETDATE() AS DATE)),
    CONSTRAINT CK_Account_BalanceNonNegative
        CHECK (account_balance >= 0)
);
GO

-- 1.3 TRANSACTION
CREATE TABLE dbo.[Transaction] (
    transaction_id   INT IDENTITY(1,1) PRIMARY KEY,
    transaction_type VARCHAR(20) NOT NULL,
    amount           NUMERIC(14,2) NOT NULL,
    transaction_date DATETIME NOT NULL
        CONSTRAINT DF_Transaction_Date DEFAULT (GETDATE()),
    location         VARCHAR(120) NOT NULL,
    CONSTRAINT CK_Transaction_Type
        CHECK (transaction_type IN ('DEPOSIT','WITHDRAWAL','TRANSFER')),
    CONSTRAINT CK_Transaction_Amount
        CHECK (amount > 0),
    CONSTRAINT CK_Transaction_Date
        CHECK (transaction_date >= '1910-01-01' AND transaction_date <= GETDATE())
);
GO

-- 1.4 TRANSACTIONACCOUNT
CREATE TABLE dbo.TransactionAccount (
    transaction_account_id INT IDENTITY(1,1) PRIMARY KEY,
    transaction_id         INT NOT NULL,
    account_id             INT NOT NULL,
    role                   VARCHAR(10) NOT NULL,  -- 'FROM' or 'TO'
    CONSTRAINT FK_TransAcc_Transaction
        FOREIGN KEY (transaction_id) REFERENCES dbo.[Transaction](transaction_id),
    CONSTRAINT FK_TransAcc_Account
        FOREIGN KEY (account_id) REFERENCES dbo.Account(account_id),
    CONSTRAINT CK_TransAcc_Role
        CHECK (role IN ('FROM','TO')),
    CONSTRAINT UQ_TransAcc_Trans_Role
        UNIQUE (transaction_id, role)
);
GO

-- 1.5 TRANSACTIONCOMMENT
CREATE TABLE dbo.TransactionComment (
    comment_id     INT IDENTITY(1,1) PRIMARY KEY,
    transaction_id INT NOT NULL,
    comment_text   VARCHAR(255) NOT NULL,
    created_at     DATETIME NOT NULL
        CONSTRAINT DF_Comment_CreatedAt DEFAULT (GETDATE()),
    CONSTRAINT FK_Comment_Transaction
        FOREIGN KEY (transaction_id) REFERENCES dbo.[Transaction](transaction_id)
);
GO

-- 1.6 CUSTOMER ADDRESS LOG
CREATE TABLE dbo.CustomerAddressLog (
    log_id          INT IDENTITY(1,1) PRIMARY KEY,
    customer_id     INT NOT NULL,
    old_street      VARCHAR(120) NULL,
    old_city        VARCHAR(60)  NULL,
    old_state       VARCHAR(60)  NULL,
    old_postal_code VARCHAR(15)  NULL,
    old_country     VARCHAR(60)  NULL,
    new_street      VARCHAR(120) NULL,
    new_city        VARCHAR(60)  NULL,
    new_state       VARCHAR(60)  NULL,
    new_postal_code VARCHAR(15)  NULL,
    new_country     VARCHAR(60)  NULL,
    changed_at      DATETIME NOT NULL
        CONSTRAINT DF_CustAddrLog_ChangedAt DEFAULT (GETDATE()),
    changed_by      SYSNAME NOT NULL,
    CONSTRAINT FK_CustAddrLog_Customer
        FOREIGN KEY (customer_id) REFERENCES dbo.Customer(customer_id)
);
GO

/*===============================================================
    2. TRIGGERS
===============================================================*/

-- 2.1 Prevent Negative Balance
CREATE TRIGGER TR_Account_PreventNegative
ON dbo.Account
AFTER UPDATE
AS
BEGIN
    IF EXISTS (SELECT 1 FROM inserted WHERE account_balance < 0)
    BEGIN
        RAISERROR('Account balance may not go below zero.',16,1);
        ROLLBACK TRANSACTION;
    END
END;
GO

-- 2.2 Log Customer Address Change
CREATE TRIGGER trg_LogCustomerAddressChange
ON dbo.Customer
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF UPDATE(street) OR UPDATE(city) OR UPDATE(state) OR UPDATE(postal_code) OR UPDATE(country)
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;

            INSERT INTO dbo.CustomerAddressLog (
                customer_id,
                old_street, old_city, old_state, old_postal_code, old_country,
                new_street, new_city, new_state, new_postal_code, new_country,
                changed_at, changed_by
            )
            SELECT
                i.customer_id,
                d.street, d.city, d.state, d.postal_code, d.country,
                i.street, i.city, i.state, i.postal_code, i.country,
                GETDATE(), SYSTEM_USER
            FROM inserted i
            INNER JOIN deleted d ON i.customer_id = d.customer_id;

            COMMIT TRANSACTION;
        END TRY
        BEGIN CATCH
            IF @@TRANCOUNT > 0
                ROLLBACK TRANSACTION;

            DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
            RAISERROR('Failed to log customer address change: %s', 16, 1, @ErrorMessage);
        END CATCH
    END
END;
GO

-- 2.3 Update Account Balance
CREATE TRIGGER trg_UpdateAccountBalance
ON dbo.TransactionAccount
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        UPDATE a
        SET a.account_balance =
            CASE ta.role
                WHEN 'TO' THEN a.account_balance + t.amount
                WHEN 'FROM' THEN a.account_balance - t.amount
            END
        FROM dbo.Account a
        INNER JOIN dbo.TransactionAccount ta ON a.account_id = ta.account_id
        INNER JOIN dbo.[Transaction] t ON ta.transaction_id = t.transaction_id
        WHERE ta.transaction_account_id IN (SELECT transaction_account_id FROM inserted);

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR('Failed to update account balance: %s', 16, 1, @ErrorMessage);
    END CATCH
END;
GO

-- 2.4 Validate Account Type
CREATE TRIGGER trg_ValidateAccountType
ON dbo.Account
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted
        WHERE account_type NOT IN ('CHECKING', 'SAVINGS', 'STOCK')
    )
    BEGIN
        RAISERROR('Invalid account type. Must be CHECKING, SAVINGS, or STOCK.', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END
END;
GO

/*===============================================================
    3. STORED PROCEDURES
===============================================================*/

-- 3.1 Add Customer
CREATE PROCEDURE dbo.usp_AddCustomer
    @first_name VARCHAR(30),
    @middle_initial CHAR(1) = NULL,
    @last_name VARCHAR(50),
    @phone_number VARCHAR(20),
    @street VARCHAR(120),
    @city VARCHAR(60),
    @state VARCHAR(60),
    @postal_code VARCHAR(15),
    @country VARCHAR(60),
    @customer_type VARCHAR(20),
    @NewCustomerID INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    -- Normalize customer type
    SET @customer_type = UPPER(@customer_type);

    IF @customer_type NOT IN ('PERSONAL','BUSINESS')
    BEGIN 
        SELECT 'FAILURE: Invalid customer type.' AS Message; 
        RETURN; 
    END;

    -- Check for duplicate customer (case-insensitive)
    IF EXISTS(
        SELECT 1 
        FROM dbo.Customer
        WHERE UPPER(first_name) = UPPER(@first_name)
          AND UPPER(last_name)  = UPPER(@last_name)
          AND phone_number = @phone_number
    )
    BEGIN 
        SELECT 'FAILURE: Duplicate customer.' AS Message; 
        RETURN; 
    END;

    -- Insert new customer
    INSERT INTO dbo.Customer(
        first_name, middle_initial, last_name,
        phone_number, street, city, state, postal_code, country,
        customer_type
    )
    VALUES(
        @first_name, @middle_initial, @last_name,
        @phone_number, @street, @city, @state, @postal_code, @country,
        @customer_type
    );

    -- Return new customer ID
    SET @NewCustomerID = SCOPE_IDENTITY();
    SELECT 'SUCCESS' AS Message, @NewCustomerID AS CustomerID;
END;
GO

-- 3.2 Add Account
CREATE PROCEDURE dbo.usp_AddAccount
    @customer_id      INT,
    @account_type     VARCHAR(20),
    @initial_balance  NUMERIC(14,2) = 0,
    @NewAccountID     INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        IF NOT EXISTS (SELECT 1 FROM dbo.Customer WHERE customer_id=@customer_id)
        BEGIN
            SELECT 'FAILURE: Customer ID does not exist.' AS Message;
            RETURN;
        END

        IF @account_type NOT IN ('CHECKING','SAVINGS','STOCK')
        BEGIN
            SELECT 'FAILURE: Invalid account type.' AS Message;
            RETURN;
        END

        IF @initial_balance < 0
        BEGIN
            SELECT 'FAILURE: Initial balance cannot be negative.' AS Message;
            RETURN;
        END

        BEGIN TRANSACTION;

            INSERT INTO dbo.Account (customer_id,account_type,date_opened,account_balance)
            VALUES (@customer_id,@account_type,CAST(GETDATE() AS DATE),@initial_balance);

            SET @NewAccountID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;

        SELECT 'SUCCESS: Account created.' AS Message, @NewAccountID AS AccountID;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

        SELECT
            'FAILURE: Error creating account.' AS Message,
            ERROR_NUMBER() AS ErrorNumber,
            ERROR_MESSAGE() AS ErrorMessage;
    END CATCH
END;
GO

-- 3.3 Post Transaction
CREATE PROCEDURE dbo.usp_PostTransaction
    @customer_id       INT,
    @transaction_type  VARCHAR(20),
    @amount            NUMERIC(14,2),
    @location          VARCHAR(120),
    @from_account_id   INT = NULL,
    @to_account_id     INT = NULL,
    @NewTransactionID  INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        IF @amount <= 0
        BEGIN
            SELECT 'FAILURE: Amount must be greater than zero.' AS Message;
            RETURN;
        END

        IF @transaction_type NOT IN ('DEPOSIT','WITHDRAWAL','TRANSFER')
        BEGIN
            SELECT 'FAILURE: Invalid transaction type.' AS Message;
            RETURN;
        END

        -- Argument validation
        IF @transaction_type = 'DEPOSIT' AND (@to_account_id IS NULL OR @from_account_id IS NOT NULL)
        BEGIN
            SELECT 'FAILURE: Deposit requires only @to_account_id.' AS Message;
            RETURN;
        END
        ELSE IF @transaction_type = 'WITHDRAWAL' AND (@from_account_id IS NULL OR @to_account_id IS NOT NULL)
        BEGIN
            SELECT 'FAILURE: Withdrawal requires only @from_account_id.' AS Message;
            RETURN;
        END
        ELSE IF @transaction_type = 'TRANSFER' AND (@from_account_id IS NULL OR @to_account_id IS NULL OR @from_account_id=@to_account_id)
        BEGIN
            SELECT 'FAILURE: Transfer requires both FROM and TO accounts (must be different).' AS Message;
            RETURN;
        END

        -- Ownership validation
        IF @transaction_type IN ('WITHDRAWAL','TRANSFER')
           AND NOT EXISTS (SELECT 1 FROM dbo.Account WHERE account_id=@from_account_id AND customer_id=@customer_id)
        BEGIN
            SELECT 'FAILURE: Customer does not own FROM account.' AS Message;
            RETURN;
        END

        IF @transaction_type IN ('DEPOSIT','TRANSFER')
           AND NOT EXISTS (SELECT 1 FROM dbo.Account WHERE account_id=@to_account_id)
        BEGIN
            SELECT 'FAILURE: TO account does not exist or not owned.' AS Message;
            RETURN;
        END

        BEGIN TRANSACTION;

            INSERT INTO dbo.[Transaction] (transaction_type, amount, transaction_date, location)
            VALUES (@transaction_type, @amount, GETDATE(), @location);

            SET @NewTransactionID = SCOPE_IDENTITY();

            IF @transaction_type='DEPOSIT'
                INSERT INTO dbo.TransactionAccount (transaction_id, account_id, role)
                VALUES (@NewTransactionID, @to_account_id, 'TO');
            ELSE IF @transaction_type='WITHDRAWAL'
                INSERT INTO dbo.TransactionAccount (transaction_id, account_id, role)
                VALUES (@NewTransactionID, @from_account_id, 'FROM');
            ELSE IF @transaction_type='TRANSFER'
                INSERT INTO dbo.TransactionAccount (transaction_id, account_id, role)
                VALUES (@NewTransactionID, @from_account_id, 'FROM'),
                       (@NewTransactionID, @to_account_id, 'TO');

        COMMIT TRANSACTION;

        SELECT 'SUCCESS: Transaction posted.' AS Message, @NewTransactionID AS TransactionID;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        SELECT 'FAILURE: Error posting transaction.' AS Message,
               ERROR_NUMBER() AS ErrorNumber,
               ERROR_MESSAGE() AS ErrorMessage;
    END CATCH
END;
GO

-- 3.4 Add Transaction Comment
CREATE PROCEDURE dbo.usp_AddTransactionComment
    @transaction_id INT,
    @comment_text   VARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        IF NOT EXISTS (SELECT 1 FROM dbo.[Transaction] WHERE transaction_id=@transaction_id)
        BEGIN
            SELECT 'FAILURE: Transaction does not exist.' AS Message;
            RETURN;
        END

        INSERT INTO dbo.TransactionComment (transaction_id, comment_text)
        VALUES (@transaction_id, @comment_text);

        SELECT 'SUCCESS: Comment added.' AS Message;

    END TRY
    BEGIN CATCH
        SELECT 'FAILURE: Error adding comment.' AS Message,
               ERROR_NUMBER() AS ErrorNumber,
               ERROR_MESSAGE() AS ErrorMessage;
    END CATCH
END;
GO

-- 3.5 Count Customers by City
CREATE PROCEDURE dbo.usp_CountCustomersByCity
    @city VARCHAR(60)
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (SELECT 1 FROM dbo.Customer WHERE UPPER(city)=UPPER(@city))
    BEGIN
        SELECT customer_type, COUNT(*) AS NumCustomers
        FROM dbo.Customer
        WHERE UPPER(city)=UPPER(@city)
        GROUP BY customer_type;
    END
    ELSE
    BEGIN
        SELECT 'There are no customers in ' + @city AS Message;
    END
END;
GO

-- 3.6 Update Customer Address
CREATE PROCEDURE dbo.usp_UpdateCustomerAddress
    @customer_id INT,
    @street VARCHAR(120),
    @city VARCHAR(60),
    @state VARCHAR(60),
    @postal_code VARCHAR(15),
    @country VARCHAR(60)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        IF NOT EXISTS (SELECT 1 FROM dbo.Customer WHERE customer_id=@customer_id)
        BEGIN
            SELECT 'FAILURE: Customer ID not found.' AS Message;
            RETURN;
        END

        UPDATE dbo.Customer
        SET street=@street, city=@city, state=@state, postal_code=@postal_code, country=@country
        WHERE customer_id=@customer_id;

        SELECT 'SUCCESS: Address updated.' AS Message;

        -- Address change logged automatically by trigger
    END TRY
    BEGIN CATCH
        SELECT 'FAILURE: Error updating address.' AS Message,
               ERROR_NUMBER() AS ErrorNumber,
               ERROR_MESSAGE() AS ErrorMessage;
    END CATCH
END;
GO

-- 3.7 Get Customer Total Balance
CREATE PROCEDURE dbo.usp_GetCustomerTotalBalance
    @customer_id INT
AS
BEGIN
    SET NOCOUNT ON;

    IF NOT EXISTS (SELECT 1 FROM dbo.Customer WHERE customer_id=@customer_id)
    BEGIN
        SELECT 'FAILURE: Customer ID not found.' AS Message;
        RETURN;
    END

    SELECT SUM(account_balance) AS TotalBalance
    FROM dbo.Account
    WHERE customer_id=@customer_id;
END;
GO
