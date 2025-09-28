/******************************************************************
   Loan Application System - Database Script
   Includes: Tables, Seed Data, and Stored Procedures
******************************************************************/

/***********************
  TABLE DEFINITIONS
***********************/

-- Applicants table: stores applicant personal info
CREATE TABLE Applicants (
  ApplicantId       VARCHAR(20) PRIMARY KEY,
  Name              VARCHAR(100),
  Phone             VARCHAR(20),
  Email             VARCHAR(100),
  Salary            DECIMAL(10, 2),
  CreditScore       INT,
  NationalIdNumber  VARCHAR(20),
  Gender            CHAR(1),
  Address           VARCHAR(255),
  CreatedAt         DATETIME DEFAULT GETDATE()
);

-- LoanApplications table: loan request info linked to Applicants
CREATE TABLE LoanApplications (
  ApplicationId   INT IDENTITY(1000,1) PRIMARY KEY,
  ApplicantId     VARCHAR(20),
  Salary          DECIMAL(10,2),
  LoanAmount      DECIMAL(10,2),
  TermMonths      INT,
  CreditScore     INT,
  StatusId        INT,
  MonthlyPayment  DECIMAL(10,2),
  CreatedAt       DATETIME DEFAULT GETDATE(),
  FOREIGN KEY (ApplicantId) REFERENCES Applicants(ApplicantId)
);

-- AuditLog table: request/response tracking for services
CREATE TABLE AuditLog (
  LogId           INT IDENTITY PRIMARY KEY,
  TransactionId   VARCHAR(50),
  RequestPayload  NVARCHAR(MAX),
  ResponsePayload NVARCHAR(MAX),
  LogType         VARCHAR(50),
  CreatedAt       DATETIME DEFAULT GETDATE()
);

-- LoanApplicationStatus table: master list of status values
CREATE TABLE LoanApplicationStatus (
  StatusId   INT PRIMARY KEY,
  StatusName VARCHAR(50)
);

-- Seed status values
INSERT INTO LoanApplicationStatus VALUES
(1, 'Pending'),
(2, 'Validated'),
(5, 'Rejected');



/***********************
  STORED PROCEDURES
***********************/

-- Insert a new loan application
GO
CREATE PROCEDURE sp_InsertLoanApplication
  @ApplicantId     VARCHAR(20),
  @Salary          DECIMAL(10,2),
  @LoanAmount      DECIMAL(10,2),
  @TermMonths      INT,
  @CreditScore     INT,
  @StatusId        INT,
  @MonthlyPayment  DECIMAL(10,2),
  @ApplicationId   INT OUTPUT
AS
BEGIN
  INSERT INTO LoanApplications (
    ApplicantId, Salary, LoanAmount, TermMonths,
    CreditScore, StatusId, MonthlyPayment
  )
  VALUES (
    @ApplicantId, @Salary, @LoanAmount, @TermMonths,
    @CreditScore, @StatusId, @MonthlyPayment
  );

  -- Return the new ApplicationId
  SET @ApplicationId = SCOPE_IDENTITY();
END;
GO

-- Approve or reject a loan by updating StatusId
GO
CREATE PROCEDURE sp_ApproveRejectLoan
  @ApplicationId INT,
  @NewStatusId   INT
AS
BEGIN
  UPDATE LoanApplications
  SET StatusId = @NewStatusId
  WHERE ApplicationId = @ApplicationId;
END;
GO

-- Insert audit log entry
GO
CREATE PROCEDURE sp_InsertAuditLog
  @TransactionId   VARCHAR(50),
  @RequestPayload  NVARCHAR(MAX),
  @ResponsePayload NVARCHAR(MAX),
  @LogType         VARCHAR(50)
AS
BEGIN
  INSERT INTO AuditLog (
    TransactionId, RequestPayload, ResponsePayload, LogType
  )
  VALUES (
    @TransactionId, @RequestPayload, @ResponsePayload, @LogType
  );
END;
GO

-- Get full details for a specific loan application
GO
CREATE PROCEDURE sp_GetApplicationDetails
    @ApplicationId INT
AS
BEGIN
    SELECT 
        LA.ApplicationId,
        LA.ApplicantId,
        LA.Salary,
        LA.LoanAmount,
        LA.TermMonths,
        LA.CreditScore,
        LA.StatusId,
        LA.MonthlyPayment,
        LA.CreatedAt AS LoanCreatedAt,
        A.Name,
        A.Phone,
        A.Email,
        A.Salary AS ApplicantSalary,
        A.CreditScore AS ApplicantCreditScore,
        A.NationalIdNumber,
        A.Gender,
        A.Address,
        A.CreatedAt AS ApplicantCreatedAt
    FROM LoanApplications LA
    INNER JOIN Applicants A ON LA.ApplicantId = A.ApplicantId
    WHERE LA.ApplicationId = @ApplicationId;
END;
GO


/***********************
  TEST DATA 
***********************/

-- View audit log
SELECT * FROM AuditLog;

-- Example cleanup operations
DELETE FROM AuditLog WHERE LogId = 5;
ALTER TABLE Applicants DROP COLUMN DateOfBirth;
DELETE FROM Applicants WHERE ApplicantId = '123456789';

-- Test select queries
SELECT * FROM Applicants;
SELECT * FROM LoanApplications;

-- Insert sample Applicants
INSERT INTO Applicants (ApplicantId, Name, Phone, Email, Salary, CreditScore, NationalIdNumber, Gender, Address)
VALUES
('A1003', 'Dana Salem', '0791234567', 'dana@example.com', 5000.00, 650, '99887766', 'M', 'Amman'),
('A1005', 'Test Test', '0797654321', 'youusef@example.com', 6000.00, 720, '11223344', 'F', 'Irbid');

-- Insert related LoanApplications
INSERT INTO LoanApplications (ApplicantId, Salary, LoanAmount, TermMonths, CreditScore, StatusId, MonthlyPayment)
VALUES
('A1003', 1000.00, 20000.00, 12, 650, 1, 1800.00), -- Pending
('A1005', 5000.00, 15000.00, 40, 720, 1, 1600.00); -- Pending

-- Insert extra pending loan for testing ApprovalService
INSERT INTO LoanApplications
(
    ApplicantId, Salary, LoanAmount, TermMonths,
    CreditScore, StatusId, MonthlyPayment, CreatedAt
)
VALUES
(
    'A1004', 1000, 5000, 12,
    550, 1, 450, GETDATE()
);

-- Force set status back to Pending (for testing)
UPDATE LoanApplications SET StatusId = 1 WHERE ApplicantId = 'A1004';
UPDATE LoanApplications SET StatusId = 1 WHERE StatusId = 5;

-- View pending loans
SELECT * FROM LoanApplications WHERE StatusId = 1;
