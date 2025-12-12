/*===============================================================
  BANKDB – TEST SCRIPT (Req 1–6 + extra credit)
  - Uses customers: Sunday M Oyebiyi, Harry Shasho
  - Includes working and non-working examples
================================================================*/

SET NOCOUNT ON;

-- 1. REQ 1 – ADD CUSTOMERS (Sunday & Harry)
-- =============================================================
DECLARE @SundayId INT, @HarryId INT;

-- 1A. Working example – Sunday (PERSONAL)
EXEC dbo.usp_AddCustomer
    @first_name     = 'Sunday',
    @middle_initial = 'M',
    @last_name      = 'Oyebiyi',
    @phone_number   = '410-555-1000',
    @street         = '101 Bank St',
    @city           = 'Baltimore',
    @state          = 'MD',
    @postal_code    = '21201',
    @country        = 'USA',
    @customer_type  = 'PERSONAL',
    @NewCustomerID  = @SundayId OUTPUT;

-- 1A. Working example – Harry (BUSINESS)
EXEC dbo.usp_AddCustomer
    @first_name     = 'Harry',
    @middle_initial = NULL,
    @last_name      = 'Shasho',
    @phone_number   = '410-555-2000',
    @street         = '500 Market Ave',
    @city           = 'Baltimore',
    @state          = 'MD',
    @postal_code    = '21202',
    @country        = 'USA',
    @customer_type  = 'BUSINESS',
    @NewCustomerID  = @HarryId OUTPUT;

SELECT @SundayId AS SundayID, @HarryId AS HarryID;

-- Show all customers
SELECT * FROM dbo.Customer ORDER BY customer_id;

-- 1B. NON-WORKING EXAMPLES
-- Invalid customer_type
EXEC dbo.usp_AddCustomer
    @first_name     = 'Test',
    @middle_initial = NULL,
    @last_name      = 'BadType',
    @phone_number   = '410-555-9999',
    @street         = 'X',
    @city           = 'X',
    @state          = 'X',
    @postal_code    = '00000',
    @country        = 'USA',
    @customer_type  = 'VIP',  -- invalid
    @NewCustomerID  = NULL;

-- Duplicate name + phone
EXEC dbo.usp_AddCustomer
    @first_name     = 'Sunday',
    @middle_initial = 'M',
    @last_name      = 'Oyebiyi',
    @phone_number   = '410-555-1000',  -- duplicate phone
    @street         = 'Somewhere',
    @city           = 'Baltimore',
    @state          = 'MD',
    @postal_code    = '21201',
    @country        = 'USA',
    @customer_type  = 'PERSONAL',
    @NewCustomerID  = NULL;



-- 2. REQ 2 – ADD ACCOUNTS FOR SUNDAY & HARRY
-- =============================================================
DECLARE @SundayChk INT, @SundaySav INT, @HarryChk INT;

-- Sunday: CHECKING and SAVINGS
EXEC dbo.usp_AddAccount
    @customer_id     = @SundayId,
    @account_type    = 'CHECKING',
    @initial_balance = 0,
    @NewAccountID    = @SundayChk OUTPUT;

EXEC dbo.usp_AddAccount
    @customer_id     = @SundayId,
    @account_type    = 'SAVINGS',
    @initial_balance = 0,
    @NewAccountID    = @SundaySav OUTPUT;

-- Harry: CHECKING
EXEC dbo.usp_AddAccount
    @customer_id     = @HarryId,
    @account_type    = 'CHECKING',
    @initial_balance = 0,
    @NewAccountID    = @HarryChk OUTPUT;

SELECT @SundayChk AS SundayChecking,
       @SundaySav AS SundaySavings,
       @HarryChk  AS HarryChecking;

-- Show accounts
SELECT * FROM dbo.Account ORDER BY account_id;

-- 2B. NON-WORKING EXAMPLES
-- Invalid account type
EXEC dbo.usp_AddAccount
    @customer_id     = @SundayId,
    @account_type    = 'CRYPTO',  -- invalid
    @initial_balance = 100,
    @NewAccountID    = NULL;

-- Non-existing customer
EXEC dbo.usp_AddAccount
    @customer_id     = 99999,
    @account_type    = 'CHECKING',
    @initial_balance = 50,
    @NewAccountID    = NULL;


-- 3. REQ 3 – TRANSACTIONS (DEPOSIT, WITHDRAWAL, TRANSFER)
--          + BALANCE TRIGGER
-- =============================================================
DECLARE @txDeposit INT, @txWithdraw INT, @txTransfer INT;

-- 3A. Deposit $500 into Sunday checking
EXEC dbo.usp_PostTransaction
    @customer_id      = @SundayId,
    @transaction_type = 'DEPOSIT',
    @amount           = 500,
    @location         = 'Branch',
    @to_account_id    = @SundayChk,
    @NewTransactionID = @txDeposit OUTPUT;

-- 3B. Withdraw $200 from Sunday checking
EXEC dbo.usp_PostTransaction
    @customer_id      = @SundayId,
    @transaction_type = 'WITHDRAWAL',
    @amount           = 200,
    @location         = 'ATM',
    @from_account_id  = @SundayChk,
    @NewTransactionID = @txWithdraw OUTPUT;

-- 3C. Transfer $100 from Sunday checking to Sunday savings
EXEC dbo.usp_PostTransaction
    @customer_id      = @SundayId,
    @transaction_type = 'TRANSFER',
    @amount           = 100,
    @location         = 'Online',
    @from_account_id  = @SundayChk,
    @to_account_id    = @SundaySav,
    @NewTransactionID = @txTransfer OUTPUT;

-- Check balances after transactions
SELECT * FROM dbo.Account ORDER BY account_id;

-- 3D. NON-WORKING TRANSACTIONS
-- Attempt overdraft
EXEC dbo.usp_PostTransaction
    @customer_id      = @SundayId,
    @transaction_type = 'WITHDRAWAL',
    @amount           = 100000,
    @location         = 'ATM',
    @from_account_id  = @SundayChk,
    @NewTransactionID = NULL;

-- Sunday tries to deposit into Harry's account
EXEC dbo.usp_PostTransaction
    @customer_id      = @SundayId,
    @transaction_type = 'DEPOSIT',
    @amount           = 50,
    @location         = 'Branch',
    @to_account_id    = @HarryChk,
    @NewTransactionID = NULL;



-- 4. REQ 3 (COMMENTS) – ADD & NON-WORKING COMMENT
-- =============================================================
-- Add a comment to the transfer transaction
EXEC dbo.usp_AddTransactionComment
    @transaction_id = @txTransfer,
    @comment_text   = 'Monthly transfer to savings';

-- Show comments
SELECT * FROM dbo.TransactionComment;

-- Non-working: comment on non-existing transaction
EXEC dbo.usp_AddTransactionComment
    @transaction_id = 99999,
    @comment_text   = 'This failed';



-- 5. REQ 4 – COUNT CUSTOMERS BY CITY (CASE-INSENSITIVE)
-- =============================================================
EXEC dbo.usp_CountCustomersByCity @city='Baltimore';
EXEC dbo.usp_CountCustomersByCity @city='baltimore';
EXEC dbo.usp_CountCustomersByCity @city='BALTIMORE';
EXEC dbo.usp_CountCustomersByCity @city='NowhereCity';



-- 6. REQ 5 – UPDATE ADDRESS + ADDRESS LOG TRIGGER
-- =============================================================
EXEC dbo.usp_UpdateCustomerAddress
    @customer_id = @SundayId,
    @street      = '999 New Bank Blvd',
    @city        = 'Baltimore',
    @state       = 'MD',
    @postal_code = '21209',
    @country     = 'USA';

-- Check updated customer row
SELECT * 
FROM dbo.Customer 
WHERE customer_id = @SundayId;

-- Check address change log 
SELECT * 
FROM dbo.CustomerAddressLog 
WHERE customer_id = @SundayId;



-- 7. REQ 6 – TOTAL BALANCE PER CUSTOMER
-- =============================================================
-- Sunday (should be 300)
EXEC dbo.usp_GetCustomerTotalBalance @customer_id = @SundayId;

-- Harry (should be 0)
EXEC dbo.usp_GetCustomerTotalBalance @customer_id = @HarryId;

-- Non-existing customer
EXEC dbo.usp_GetCustomerTotalBalance @customer_id = 99999;

SET NOCOUNT OFF;
