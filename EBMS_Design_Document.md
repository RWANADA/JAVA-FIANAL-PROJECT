# Electricity Bill Management System (EBMS)
## Complete Design Document
**Student:** Iradukunda Kubana Christian | **ID:** 27143 | **Course:** AUCA INSY 7312

---

## 1. SYSTEM ARCHITECTURE DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CLIENT APPLICATION                                 │
│                  (Java Swing Desktop GUI)                               │
│                                                                         │
│  ┌──────────┐  ┌──────────────────────────────────────────────────────┐│
│  │LoginFrame│  │              DashboardFrame                          ││
│  │          │  │  ┌──────────────────┐  ┌───────────────────────────┐ ││
│  │Register  │  │  │    SIDEBAR       │  │      CONTENT AREA         │ ││
│  │Frame     │  │  │  ⚡ Dashboard    │  │  CustomerManagementFrame  │ ││
│  └──────────┘  │  │  👥 Customers   │  │  MeterManagementFrame     │ ││
│                │  │  📟 Meters      │  │  MeterReadingFrame        │ ││
│                │  │  📊 Readings    │  │  BillManagementFrame      │ ││
│                │  │  🧾 Bills       │  │  PaymentFrame             │ ││
│                │  │  💳 Payments    │  │  TariffManagementFrame    │ ││
│                │  │  📋 Tariffs     │  │  ReportFrame              │ ││
│                │  │  📈 Reports     │  │  UserManagementFrame      │ ││
│                │  │  👤 Users       │  └───────────────────────────┘ ││
│                │  └──────────────────┘                                ││
│                └──────────────────────────────────────────────────────┘│
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  RMIClientConnector (lookup: rmi://localhost:5000/ebms)          │  │
│  └────────────────────────┬─────────────────────────────────────────┘  │
└───────────────────────────┼─────────────────────────────────────────────┘
                            │  Java RMI  (TCP port 5000)
┌───────────────────────────┼─────────────────────────────────────────────┐
│                           ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  RMIServer — LocateRegistry.createRegistry(5000)                 │  │
│  │  Naming.rebind("rmi://localhost:5000/ebms", service)             │  │
│  └─────────────────────────┬────────────────────────────────────────┘  │
│                            │                                           │
│  ┌─────────────────────────▼────────────────────────────────────────┐  │
│  │              IElecService (Remote Interface)                      │  │
│  │  Auth · Customer · Meter · Bill · Payment · Tariff · Reports      │  │
│  └─────────────────────────┬────────────────────────────────────────┘  │
│                            │                                           │
│  ┌─────────────────────────▼────────────────────────────────────────┐  │
│  │                   ElecServiceImpl                                 │  │
│  └───┬──────────────┬──────────────┬────────────┬───────────────────┘  │
│      │              │              │            │                       │
│  ┌───▼──┐  ┌────────▼──┐  ┌───────▼──┐  ┌─────▼─────────────────────┐│
│  │DAO   │  │DAO        │  │DAO       │  │  Utilities                 ││
│  │Layer │  │(Customer  │  │(Bill,    │  │  EmailUtil (OTP/JavaMail)  ││
│  │(User │  │Meter,     │  │Payment,  │  │  HibernateUtil             ││
│  │Tariff│  │Reading)   │  │Otp)      │  │  NotificationPublisher     ││
│  └───┬──┘  └────────┬──┘  └───────┬──┘  │  (ActiveMQ 5.16.7)        ││
│      └──────────────┴─────────────┘     │  PasswordUtil (BCrypt)     ││
│                     │                   └───────────────────────────┘ │
│  ┌──────────────────▼───────────────────────────────────────────────┐  │
│  │              Hibernate 4.3.1 + C3P0 Connection Pool               │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │  JDBC                                    │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │     PostgreSQL  (localhost:5432/electricity_bill_management_db)   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                       SERVER APPLICATION                                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. UML CLASS DIAGRAM

```
┌──────────────────────┐         ┌──────────────────────┐
│       <<Entity>>     │         │       <<Entity>>      │
│         User         │1       1│       OtpRecord       │
│──────────────────────│─────────│──────────────────────│
│ -id: Long            │         │ -id: Long             │
│ -username: String    │         │ -otp: String          │
│ -passwordHash: String│         │ -expiresAt: Date      │
│ -role: String        │         │ -used: boolean        │
│ -email: String       │         │ -user: User           │
│──────────────────────│         └──────────────────────┘
│ +getters/setters     │
└──────────────────────┘

┌──────────────────────┐    1..* ┌──────────────────────┐
│       <<Entity>>     │────────▶│       <<Entity>>      │
│       Customer       │         │         Meter         │
│──────────────────────│         │──────────────────────│
│ -id: Long            │         │ -id: Long             │
│ -accountNo: String   │         │ -meterNumber: String  │
│ -name: String        │         │ -type: String         │
│ -address: String     │         │ -status: String       │
│ -phone: String       │         │ -installDate: Date    │
│ -email: String       │         │ -customer: Customer   │
│ -status: String      │         │──────────────────────│
│──────────────────────│         │ +getters/setters      │
│ +getters/setters     │         └──────────┬───────────┘
└──────────────────────┘                    │ 1..*
                                            ▼
                              ┌──────────────────────────┐
                              │         <<Entity>>        │
                              │        MeterReading       │
                              │──────────────────────────│
                              │ -id: Long                 │
                              │ -previousUnits: double    │
                              │ -currentUnits: double     │
                              │ -readingDate: Date        │
                              │ -meter: Meter             │
                              └──────────────┬────────────┘
                                             │ 1:1
                                             ▼
┌──────────────────────┐    1..* ┌──────────────────────┐
│       <<Entity>>     │◀────────│       <<Entity>>      │
│        Payment       │         │          Bill         │
│──────────────────────│         │──────────────────────│
│ -id: Long            │         │ -id: Long             │
│ -amount: double      │         │ -issueDate: Date      │
│ -method: String      │         │ -dueDate: Date        │
│ -transactionRef:Str  │         │ -totalAmount: double  │
│ -paymentDate: Date   │         │ -status: String       │
│ -bill: Bill          │         │ -penalty: double      │
└──────────────────────┘         │ -meterReading: ...    │
                                 │ -tariffs: List<Tariff>│
                                 └──────────┬────────────┘
                                            │ M:M (bill_tariff)
                                            ▼
                              ┌──────────────────────────┐
                              │         <<Entity>>        │
                              │          Tariff           │
                              │──────────────────────────│
                              │ -id: Long                 │
                              │ -name: String             │
                              │ -category: String         │
                              │ -ratePerUnit: double      │
                              │ -effectiveDate: Date      │
                              └──────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                   <<Remote Interface>>                        │
│                     IElecService                             │
│──────────────────────────────────────────────────────────────│
│ +authenticateUser(username, password): User                  │
│ +verifyOtp(userId, otp): boolean                             │
│ +registerUser(user): void                                    │
│ +getAllUsers(): List<User>                                    │
│ +getUserById(id): User                                       │
│ +updateUser(user): void                                      │
│ +updateUserWithPassword(user): void                          │
│ +deleteUser(userId): void                                    │
│                                                              │
│ +addCustomer(c): void     +updateCustomer(c): void           │
│ +deleteCustomer(id): void +getCustomerById(id): Customer     │
│ +getAllCustomers(): List<Customer>                            │
│                                                              │
│ +addMeter(m): void        +updateMeter(m): void              │
│ +deleteMeter(id): void    +getMeterById(id): Meter           │
│ +getAllMeters(): List<Meter>                                  │
│ +getMetersByCustomer(customerId): List<Meter>                │
│                                                              │
│ +addMeterReadingAndGenerateBill(r, tariffId): Bill           │
│ +getAllBills(): List<Bill>                                    │
│ +getBillsByStatus(status): List<Bill>                        │
│ +getBillsByCustomer(customerId): List<Bill>                  │
│ +markBillAsPaid(billId): void                                │
│ +applyPenaltyIfOverdue(billId): boolean                      │
│ +applyManualPenalty(billId, amount): void                    │
│                                                              │
│ +recordPayment(p): void                                      │
│ +getAllPayments(): List<Payment>                             │
│ +getPaymentsByBill(billId): List<Payment>                    │
│                                                              │
│ +addTariff(t): void       +updateTariff(t): void             │
│ +deleteTariff(id): void   +getTariffById(id): Tariff         │
│ +getAllTariffs(): List<Tariff>                                │
│                                                              │
│ +getOverdueBills(): List<Bill>                               │
└──────────────────────────────────────────────────────────────┘
                         ▲ implements
┌──────────────────────────────────────────────────────────────┐
│                   ElecServiceImpl                             │
│    (UnicastRemoteObject + IElecService)                       │
│──────────────────────────────────────────────────────────────│
│ -billDAO: IBillDAO                                            │
│ -customerDAO: ICustomerDAO                                    │
│ -meterDAO: IMeterDAO                                          │
│ -readingDAO: IMeterReadingDAO                                 │
│ -paymentDAO: IPaymentDAO                                      │
│ -tariffDAO: ITariffDAO                                        │
│ -userDAO: IUserDAO                                            │
│ -otpDAO: IOtpDAO                                              │
└──────────────────────────────────────────────────────────────┘

┌────────────────────┐  ┌──────────────────────┐
│  <<Interface>>     │  │  <<Interface>>        │
│  GenericDAO<T,ID>  │  │  IBillDAO             │
│──────────────────  │  │  ICustomerDAO         │
│ +save(T)           │  │  IMeterDAO            │
│ +update(T)         │  │  IMeterReadingDAO     │
│ +delete(T)         │  │  ITariffDAO           │
│ +findById(ID): T   │  │  IUserDAO             │
│ +findAll(): List   │  │  IOtpDAO              │
└────────────────────┘  └──────────────────────┘
         ▲                        ▲
         └──────────┬─────────────┘
              implements
    BillDAOImpl, CustomerDAOImpl, MeterDAOImpl,
    MeterReadingDAOImpl, PaymentDAOImpl,
    TariffDAOImpl, UserDAOImpl, OtpDAOImpl
    (all extend HibernateUtil SessionFactory)
```

---

## 3. SEQUENCE DIAGRAM — Login with OTP (2FA)

```
 LoginFrame          RMIClientConnector      IElecService        EmailUtil       OtpRecord
     │                       │                    │                  │               │
     │─── login() ──────────▶│                    │                  │               │
     │                       │─ authenticateUser()▶│                  │               │
     │                       │                    │─── hashPwd ──────▶               │
     │                       │                    │─── query DB ──────────────────────▶
     │                       │                    │◀── User object ────────────────────
     │                       │                    │─── generate OTP ──▶               │
     │                       │                    │─── save OTP ──────────────────────▶
     │                       │                    │─── sendEmail()────▶               │
     │                       │◀─── User ──────────│                  │               │
     │◀── User ──────────────│                    │                  │               │
     │                       │                    │                  │               │
     │─── showOtpDialog() ──▶(user types OTP)     │                  │               │
     │                       │                    │                  │               │
     │─── verifyOtp() ───────▶                    │                  │               │
     │                       │─── verifyOtp() ───▶│                  │               │
     │                       │                    │─── load OTP ──────────────────────▶
     │                       │                    │◀── OtpRecord ──────────────────────
     │                       │                    │─── check expiry & match           │
     │                       │                    │─── mark used ─────────────────────▶
     │                       │◀─── true ──────────│                  │               │
     │◀─── true ─────────────│                    │                  │               │
     │                       │                    │                  │               │
     │─── new DashboardFrame(user)                │                  │               │
```

---

## 4. SEQUENCE DIAGRAM — Generate Bill from Meter Reading

```
 MeterReadingFrame     RMIClientConnector    ElecServiceImpl    MeterReadingDAO    BillDAO    TariffDAO
       │                      │                    │                  │              │           │
       │── generateBill() ───▶│                    │                  │              │           │
       │                      │─addMeterReading    │                  │              │           │
       │                      │  AndGenerateBill()▶│                  │              │           │
       │                      │                    │──save(reading)──▶│              │           │
       │                      │                    │◀── MeterReading──│              │           │
       │                      │                    │──getTariffById()──────────────────────────▶│
       │                      │                    │◀── Tariff──────────────────────────────────│
       │                      │                    │── calc units consumed             │           │
       │                      │                    │   (current - previous)            │           │
       │                      │                    │── calc amount = units × rate      │           │
       │                      │                    │── set issueDate + dueDate(30days) │           │
       │                      │                    │── set status = UNPAID             │           │
       │                      │                    │── save(bill)─────────────────────▶│           │
       │                      │                    │◀── Bill──────────────────────────│           │
       │                      │◀── Bill ───────────│                  │              │           │
       │◀── Bill ─────────────│                    │                  │              │           │
       │── "Bill generated!"  │                    │                  │              │           │
```

---

## 5. DATABASE ER DIAGRAM

```
┌──────────────────┐
│     USERS        │
│──────────────────│
│ id          PK   │────────────────────────────┐
│ username  UNIQUE │                            │
│ password_hash    │                            │
│ role             │                            ▼
│ email            │         ┌───────────────────────────┐
└──────────────────┘         │         OTP_RECORDS       │
                             │───────────────────────────│
                             │ id          PK            │
                             │ user_id  FK→users.id      │
                             │ otp                       │
                             │ expires_at                │
                             │ used                      │
                             └───────────────────────────┘

┌──────────────────────────┐
│       CUSTOMERS          │
│──────────────────────────│
│ id          PK           │
│ account_no  UNIQUE       │
│ name                     │
│ address                  │
│ phone                    │
│ email                    │
│ status  (ACTIVE/INACTIVE)│
└───────────┬──────────────┘
            │ 1
            │ «has»
            │ *
┌───────────▼──────────────┐
│         METERS           │
│──────────────────────────│
│ id          PK           │
│ meter_number  UNIQUE     │
│ type  (Residential/..)   │
│ status                   │
│ install_date             │
│ customer_id  FK→customers│
└───────────┬──────────────┘
            │ 1
            │ «has»
            │ *
┌───────────▼──────────────┐
│      METER_READINGS      │
│──────────────────────────│
│ id           PK          │
│ previous_units           │
│ current_units            │
│ reading_date             │
│ meter_id  FK→meters.id   │
└───────────┬──────────────┘
            │ 1:1
            ▼
┌──────────────────────────┐          ┌─────────────────────────┐
│          BILLS           │          │         TARIFFS          │
│──────────────────────────│   M:M    │─────────────────────────│
│ id           PK          │──────────│ id          PK          │
│ reading_id FK→readings   │bill_tariff│ name                   │
│ issue_date               │          │ category                │
│ due_date                 │          │ rate_per_unit           │
│ total_amount             │          │ effective_date          │
│ status(PAID/UNPAID/OVR)  │          └─────────────────────────┘
│ penalty                  │
└───────────┬──────────────┘
            │ 1
            │ «has»
            │ *
┌───────────▼──────────────┐
│        PAYMENTS          │
│──────────────────────────│
│ id             PK        │
│ bill_id  FK→bills.id     │
│ amount                   │
│ method                   │
│ transaction_ref          │
│ payment_date             │
└──────────────────────────┘

┌──────────────────────────┐
│      NOTIFICATIONS       │
│──────────────────────────│
│ id          PK           │
│ message                  │
│ status                   │
│ created_at               │
└──────────────────────────┘

           ┌────────────────────────────────────┐
           │         BILL_TARIFF  (join)         │
           │────────────────────────────────────│
           │ bill_id    FK→bills.id              │
           │ tariff_id  FK→tariffs.id            │
           └────────────────────────────────────┘
```

---

## 6. UI DESIGN SUMMARY

### Color Palette

| Token       | Hex         | Usage                          |
|-------------|-------------|--------------------------------|
| `NAVY_DARK` | `#040E28`   | Sidebar gradient start, header |
| `NAVY`      | `#0A2351`   | Primary brand, text titles     |
| `BLUE`      | `#1565C0`   | Buttons, links, accents        |
| `ACCENT`    | `#FFC107`   | Bolt icon, highlights          |
| `GREEN`     | `#165C34`   | PAID status, Add button        |
| `RED`       | `#991B1B`   | UNPAID/OVERDUE, Delete button  |
| `AMBER`     | `#78350F`   | UNPAID/OVERDUE status badge    |
| `BG`        | `#F1F4F9`   | Application background         |
| `WHITE`     | `#FFFFFF`   | Cards, form panels             |

### UI Component Hierarchy

```
LoginFrame (860×560)
├── Left panel — brand/decorative (340px wide, gradient navy)
│   └── Logo ⚡  |  App name  |  Feature tags  |  Footer
└── Right panel — login form (white card)
    └── Welcome heading  |  Username  |  Password  |  Sign In  |  Register

DashboardFrame (1120×700)
├── Sidebar (225px, gradient navy)
│   ├── Logo area
│   ├── OVERVIEW group → Dashboard
│   ├── MANAGEMENT group → 6 items
│   ├── ANALYTICS & ADMIN → 2 items
│   └── User info + logout
├── Top bar (56px, white)
│   └── Breadcrumb  |  User badge + name
├── Content area
│   ├── Greeting row + date
│   ├── Stat bar (4 metric cards: Customers · Bills · Payments · Meters)
│   └── Quick-access grid (2 rows × 4 columns)
└── Status bar (28px, gradient navy)

Management Frames (980×640)
├── Header bar (gradient, icon + title + subtitle + accent stripe)
├── Search bar (white panel)
├── Table (striped, navy header, selection highlight)
├── Form card (white, rounded, subtle shadow)
│   └── Field labels + inputs/combos in GridBagLayout
├── Button row (Add · Update · Delete · Refresh · Export)
└── Status bar
```

### Status Badge Colors

| Status    | Background  | Text      |
|-----------|-------------|-----------|
| `PAID`    | `#DCFCE7`   | `#165C34` |
| `UNPAID`  | `#FEF3C7`   | `#78350F` |
| `OVERDUE` | `#FEE2E2`   | `#991B1B` |
| `ACTIVE`  | `#DCFCE7`   | `#165C34` |
| `INACTIVE`| `#FEE2E2`   | `#991B1B` |

---

## 7. PACKAGE STRUCTURE

```
chris project2/
├── ElectricityBillManagementSystemServer27143/
│   └── src/
│       ├── hibernate.cfg.xml
│       ├── GenericDAO.java
│       └── com/ebms27143/
│           ├── dao/
│           │   ├── GenericDAO.java
│           │   ├── IBillDAO.java
│           │   ├── ICustomerDAO.java
│           │   ├── IMeterReadingDAO.java
│           │   ├── IOtpDAO.java
│           │   ├── ITariffDAO.java
│           │   ├── IUserDAO.java
│           │   └── impl/
│           │       ├── BillDAOImpl.java
│           │       ├── CustomerDAOImpl.java
│           │       ├── HibernateUtil.java
│           │       ├── MeterDAOImpl.java
│           │       ├── MeterReadingDAOImpl.java
│           │       ├── OtpDAOImpl.java
│           │       ├── PaymentDAOImpl.java
│           │       ├── TariffDAOImpl.java
│           │       └── UserDAOImpl.java
│           ├── entity/
│           │   ├── Bill.java
│           │   ├── Customer.java
│           │   ├── Meter.java
│           │   ├── MeterReading.java
│           │   ├── Notification.java
│           │   ├── OtpRecord.java
│           │   ├── Payment.java
│           │   ├── Tariff.java
│           │   └── User.java
│           ├── rmi/
│           │   └── RMIServer.java          ← Main entry point
│           ├── service/
│           │   ├── IElecService.java       ← Remote interface (30 methods)
│           │   └── impl/
│           │       └── ElecServiceImpl.java
│           └── util/
│               ├── EmailUtil.java          ← OTP email sender
│               ├── HibernateUtil.java
│               ├── NotificationPublisher.java ← ActiveMQ
│               └── PasswordUtil.java       ← BCrypt hashing
│
└── ElectricityBillManagementSystemClient27143/
    └── src/
        └── com/ebms27143/
            ├── client/
            │   ├── service/
            │   │   └── RMIClientConnector.java
            │   └── view/
            │       ├── UITheme.java              ← Shared styles
            │       ├── LoginFrame.java           ← Split-panel login
            │       ├── RegisterFrame.java        ← Account creation
            │       ├── DashboardFrame.java       ← Main shell + navigation
            │       ├── CustomerManagementFrame.java
            │       ├── MeterManagementFrame.java
            │       ├── MeterReadingFrame.java
            │       ├── BillManagementFrame.java
            │       ├── PaymentFrame.java
            │       ├── TariffManagementFrame.java
            │       ├── ReportFrame.java
            │       └── UserManagementFrame.java
            ├── entity/                           ← Shared serializable entities
            └── service/
                └── IElecService.java             ← Remote interface stub
```

---

*Generated: 2026-05-18 | EBMS Design Document v2.0*
