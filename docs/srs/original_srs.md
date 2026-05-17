# **🧩 MODULE: AUTHENTICATION & AUTHORIZATION**

---

## **1\. Purpose**

Enables secure user access, identity verification, and role-based system control.

---

## **2\. Scope**

### **In Scope:**

* User login (email/phone \+ password)  
* Logout  
* Password reset  
* Session/token management  
* Role-based access enforcement

### **Out of Scope:**

* User profile data (handled in Profile Module)  
* User creation (handled in User/Admission Module)

---

## **3\. Actors**

* Admin  
* Teacher  
* Student  
* Accountant  
* System (for automated flows like token generation)

---

## 

## **4\. Functional Requirements**

### **Authentication**

* **FR-AUTH-001:** System shall allow login using email \+ password  
* **FR-AUTH-002:** System shall allow login using phone number \+ password  
* **FR-AUTH-003:** System shall validate credentials before granting access  
* **FR-AUTH-004:** System shall block login for inactive users

---

### **Session Management**

* **FR-AUTH-005:** System shall generate JWT token on successful login  
* **FR-AUTH-006:** System shall maintain active session per user  
* **FR-AUTH-007:** System shall invalidate session on logout  
* **FR-AUTH-008:** System shall expire session after configurable duration

---

### **Password Management**

* **FR-AUTH-009:** System shall support forgot password via email or phone  
* **FR-AUTH-010:** System shall generate secure reset token  
* **FR-AUTH-011:** System shall allow password reset using valid token  
* **FR-AUTH-012:** System shall invalidate token after use or expiry

---

### **Optional 2FA**

* **FR-AUTH-013:** System shall support optional OTP-based 2FA  
* **FR-AUTH-014:** System shall send OTP via SMS or email  
* **FR-AUTH-015:** System shall verify OTP before login completion

---

### **Role-Based Access**

* **FR-AUTH-016:** System shall assign role to each user  
* **FR-AUTH-017:** System shall restrict API access based on role  
* **FR-AUTH-018:** System shall deny unauthorized access requests

---

## **5\. Business Rules**

* Password must be minimum **6 characters**  
* Phone must follow **Bangladesh format (+880XXXXXXXXXX)**  
* Reset token validity: **15 minutes**  
* OTP validity (if enabled): **5 minutes**  
* Maximum failed login attempts: **5 → then temporary lock (15 min)**  
* One active session per device (multi-device allowed)   
* JWT expiry: **24 hours (configurable)**

---

## **6\. User Flow**

### **Login Flow**

1. User enters email/phone \+ password  
2. System validates input format  
3. System checks user existence  
4. System verifies password  
5. (If 2FA enabled) → OTP sent → verified  
6. System generates JWT token  
7. Session stored  
8. User redirected to dashboard

---

### **Forgot Password Flow**

1. User submits email/phone  
2. System generates reset token  
3. Token sent via SMS/email  
4. User opens link  
5. User sets new password  
6. Token invalidated  
7. User can login

---

### **Logout Flow**

1. User clicks logout  
2. System invalidates token/session  
3. User redirected to login

---

## **7\. Data Requirements**

### **Entities**

* User  
* AuthSession  
* PasswordResetToken  
* OTP (if 2FA enabled)

---

### **Key Fields**

**User**

* id (UUID)  
* email / phone (unique)  
* password\_hash  
* role  
* is\_active  
* last\_login

**AuthSession**

* token  
* user\_id  
* expires\_at  
* device\_info

**PasswordResetToken**

* token  
* user\_id  
* expires\_at  
* used

---

## **8\. API Contracts**

---

### **POST `/auth/login`**

**Request**

{  
  "identifier": "email\_or\_phone",  
  "password": "string"  
}

**Response**

{  
  "access\_token": "jwt\_token",  
  "user": {  
    "id": "uuid",  
    "role": "ADMIN"  
  }  
}

**Errors**

* 401: Invalid credentials  
* 403: Account inactive  
* 429: Too many attempts

---

### **POST `/auth/logout`**

**Headers**

Authorization: Bearer \<token\>

**Response**

{  
  "message": "Logged out successfully"  
}

---

### **POST `/auth/forgot-password`**

{  
  "identifier": "email\_or\_phone"  
}

---

### **POST `/auth/reset-password`**

{  
  "token": "reset\_token",  
  "new\_password": "string"  
}

---

## **9\. Non-Functional Requirements**

* Passwords must be hashed using **bcrypt or equivalent**  
* All APIs must be served over **HTTPS**  
* Login response time \< **2 seconds**  
* System must support **1000+ concurrent sessions**  
* Tokens must be securely signed (JWT secret rotation supported)

---

## **10\. Edge Cases**

* Invalid credentials  
* Expired reset token  
* OTP mismatch  
* Account locked due to failed attempts  
* Multiple rapid login attempts (rate limiting)  
* User deleted but token still active

---

## **11\. Dependencies**

* Notification System (SMS/Email for OTP/reset)  
* User Management Module  
* Security infrastructure (JWT, encryption)

---

## **12\. Assumptions**

* Users are pre-created (via admission or admin)  
* SMS/email service is available  
* JWT-based authentication is used

---

## **13\. Open Questions**

* Do we enforce 2FA for Admin only or optional for all?  
* Should we allow social login in future?  
* Max concurrent sessions per user?

---

# **🧩 MODULE: ADMISSION MANAGEMENT SYSTEM**

---

## **1\. Purpose**

Handles the complete student admission lifecycle from application to final enrollment with payment verification.

---

## **2\. Scope**

### **In Scope:**

* Online admission application (basic)  
* Manual admission by admin (full \+ payment)  
* Admission review & approval workflow  
* Registration completion (extended data)  
* Admission fee payment (online \+ manual)  
* Student account creation

### **Out of Scope:**

* Monthly fee collection (handled in Payment Module)  
* Student profile updates post-admission

---

## **3\. Actors**

* Student (Class 6–12)  
* Parent/Guardian (primary for younger students)  
* Admin  
* System (token, notification, validation)

---

## **4\. Functional Requirements**

---

### **Application Submission**

* **FR-ADM-001:** System shall allow submission of online admission form with basic information  
* **FR-ADM-002:** System shall allow class and batch selection during application  
* **FR-ADM-003:** System shall create application with status \= PENDING  
* **FR-ADM-004:** System shall prevent duplicate applications using phone number

---

### **Admin Review**

* **FR-ADM-005:** Admin shall view list of applications  
* **FR-ADM-006:** Admin shall approve or reject application  
* **FR-ADM-007:** System shall store admin decision with timestamp

---

### **Approval & Registration Link**

* **FR-ADM-008:** System shall generate secure registration link on approval  
* **FR-ADM-009:** Link shall expire after 7 days  
* **FR-ADM-010:** System shall send link via SMS/email  
* **FR-ADM-011:** System shall allow link regeneration

---

### **Registration Completion**

* **FR-ADM-012:** System shall allow full registration using secure link  
* **FR-ADM-013:** System shall collect detailed student and parent information  
* **FR-ADM-014:** System shall allow document upload (photo, birth certificate, academic papers)  
* **FR-ADM-015:** System shall require password setup  
* **FR-ADM-016:** System shall mark registration as completed

---

### **Payment Handling**

* **FR-ADM-017:** System shall require admission fee before final activation  
* **FR-ADM-018:** System shall support online payment  
* **FR-ADM-019:** System shall allow admin to verify payment manually  
* **FR-ADM-020:** System shall record payment details (amount, method, transaction ID)

---

### **Manual Admission (Admin)**

* **FR-ADM-021:** Admin shall create student directly with full data  
* **FR-ADM-022:** Admin shall record payment during manual admission  
* **FR-ADM-023:** System shall bypass approval flow for manual admission

---

### **Finalization**

* **FR-ADM-024:** System shall create student account after successful registration \+ payment  
* **FR-ADM-025:** System shall assign student to selected batch  
* **FR-ADM-026:** System shall generate login credentials  
* **FR-ADM-027:** System shall notify student/parent

---

## **5\. Business Rules**

* Registration link validity: **7 days**  
* Admission fee must be paid before activation  
* One active application per phone number  
* Mandatory fields for full registration  
* Documents must be uploaded before completion  
* Payment methods:  
  * Online (gateway)  
  * Manual (Cash / MFS / POS)  
* Bangladesh context:  
  * Parent phone is primary contact  
  * NID required for parent (optional but recommended)

---

## **6\. User Flow**

---

### **Online Admission Flow (Optimized)**

1. Student/Parent submits basic form:  
   * Name, phone, class, batch  
2. Application \= PENDING  
3. Admin reviews:  
   * Verify details  
   * Approve / Reject  
4. If Approved:  
   * System generates secure link  
   * Sends SMS/email  
5. Parent opens link:  
   * Fills full form  
   * Uploads documents  
   * Sets password  
6. Payment Step:  
   * Pay admission fee (online OR manual verification)  
7. System:  
   * Verifies payment  
   * Creates student account  
   * Assigns batch  
   * Sends confirmation

---

### **Manual Admission Flow (Admin)**

1. Admin opens admission form  
2. Inputs full data  
3. Uploads documents  
4. Records payment:  
   * Amount  
   * Method (MFS, POS, Cash)  
   * Transaction ID  
5. System:  
   * Creates student directly  
   * Assigns batch  
   * Sends credentials

---

## **7\. Data Requirements**

---

### **AdmissionApplication**

* id (UUID)  
* student\_name  
* phone  
* email (optional)  
* class\_id  
* batch\_id  
* status (PENDING / APPROVED / REJECTED / EXPIRED)  
* created\_at

---

### **RegistrationDetails**

**Student Info**

* full\_name  
* dob  
* gender

**Parent Info**

* father\_name  
* mother\_name  
* parent\_phone  
* parent\_nid

**Other**

* address  
* photo\_url  
* documents (array)

---

### **AdmissionPayment**

* id  
* student\_id / application\_id  
* amount  
* method (CASH, MFS, POS, ONLINE)  
* transaction\_id  
* status  
* verified\_by (admin\_id)

---

### **RegistrationToken**

* token  
* application\_id  
* expires\_at  
* used

---

## **8\. API Contracts**

---

### **POST `/admissions/apply`**

{  
  "student\_name": "string",  
  "phone": "string",  
  "class\_id": "uuid",  
  "batch\_id": "uuid"  
}

---

### **POST `/admissions/{id}/approve`**

{  
  "status": "APPROVED"  
}

---

### **POST `/admissions/{id}/reject`**

{  
  "reason": "string"  
}

---

### **POST `/admissions/complete-registration`**

{  
  "token": "string",  
  "student\_data": {},  
  "parent\_data": {},  
  "documents": \[\],  
  "password": "string"  
}

---

### **POST `/admissions/manual`**

{  
  "student\_data": {},  
  "parent\_data": {},  
  "payment": {  
    "amount": 5000,  
    "method": "MFS",  
    "transaction\_id": "abc123"  
  }  
}

---

## **9\. Non-Functional Requirements**

* Registration link must be **secure & unguessable (UUID \+ hash)**  
* Upload size limit: **5MB per file**  
* System must handle **slow internet users (retry support)**  
* Mobile-first UI (parents using phones)  
* SMS delivery within **5 seconds**

---

## **10\. Edge Cases**

* Link expired before completion  
* Payment done but registration incomplete  
* Duplicate student (same phone)  
* Admin approves but student never completes  
* Wrong batch selected  
* Partial document upload  
* Network drop during submission  
* Manual payment recorded incorrectly

---

## **11\. Dependencies**

* Authentication system (account creation)  
* Payment system (online \+ manual)  
* Notification system (SMS/email)  
* Class & Batch module

---

## **12\. Assumptions**

* Parents will primarily complete registration for younger students  
* SMS gateway is available in Bangladesh  
* Admin verifies manual payments physically

---

## **13\. Open Questions**

* Should we allow **admission fee waiver/discount**?  
* Do we allow **multiple attempts after rejection**?  
* Should batch capacity block admission automatically?  
* Do we need **document verification by the admin after upload**?

---

# **🧩 MODULE: USER MANAGEMENT SYSTEM**

---

## **1\. Purpose**

Manages all users (students, teachers, employees) including creation, updates, assignments, and administrative control.

---

## **2\. Scope**

### **In Scope:**

* Student, Teacher, Employee management  
* Manual user creation by admin  
* User listing, search, filtering  
* Profile view (admin-side)  
* Batch assignment & changes  
* Payment status updates (manual entry)  
* Salary tracking (manual updates)

### **Out of Scope:**

* Authentication (handled in Auth module)  
* Admission workflow (handled separately)  
* Payroll calculation logic (handled in Payroll module)

---

## **3\. Actors**

* Admin  
* Accountant (limited access)  
* System (data sync with other modules)

---

## **4\. Functional Requirements**

---

## **🔹 4.1 Student Management**

### **Creation & Registration**

* **FR-USER-STU-001:** Admin shall create student manually with full data  
* **FR-USER-STU-002:** System shall allow assigning class and batch during creation  
* **FR-USER-STU-003:** System shall generate student ID automatically

---

### **Listing & Search**

* **FR-USER-STU-004:** System shall provide student list view  
* **FR-USER-STU-005:** System shall support search by name, phone, student ID  
* **FR-USER-STU-006:** System shall support filters (class, batch, payment status)

---

### **Profile & Details**

* **FR-USER-STU-007:** Admin shall view student details (profile \+ academic \+ attendance \+ payment summary)  
* **FR-USER-STU-008:** Admin shall update student profile data  
* **FR-USER-STU-009:** System shall display attendance summary  
* **FR-USER-STU-010:** System shall display payment summary

---

### **Batch & Academic Control**

* **FR-USER-STU-011:** Admin shall change student batch  
* **FR-USER-STU-012:** System shall update mapping after batch change

---

### **Payment Updates**

* **FR-USER-STU-013:** Admin shall manually update payment records  
* **FR-USER-STU-014:** System shall reflect payment status in student profile

---

---

## **🔹 4.2 Teacher Management**

### **Creation**

* **FR-USER-TEA-001:** Admin shall register teacher manually  
* **FR-USER-TEA-002:** System shall store subject expertise  
* **FR-USER-TEA-003:** System shall store salary configuration

---

### **Listing**

* **FR-USER-TEA-004:** System shall provide teacher list  
* **FR-USER-TEA-005:** System shall support search/filter

---

### **Profile**

* **FR-USER-TEA-006:** Admin shall view teacher profile (academic \+ attendance \+ payment summary)  
* **FR-USER-TEA-007:** Admin shall update teacher profile

---

### **Payment**

* **FR-USER-TEA-008:** Admin shall manually update salary/payment records  
* **FR-USER-TEA-009:** System shall maintain payment history

---

---

## **🔹 4.3 Employee Management**

### **Creation**

* **FR-USER-EMP-001:** Admin shall create employee manually  
* **FR-USER-EMP-002:** System shall store role and salary

---

### **Listing**

* **FR-USER-EMP-003:** System shall provide employee list  
* **FR-USER-EMP-004:** System shall support search/filter

---

### **Profile**

* **FR-USER-EMP-005:** Admin shall view employee profile  
* **FR-USER-EMP-006:** Admin shall update employee profile

---

### **Payment**

* **FR-USER-EMP-007:** Admin shall manually update salary/payment  
* **FR-USER-EMP-008:** System shall maintain payment history

---

---

## **5\. Business Rules**

* Student must belong to **one class and one batch at a time**  
* Batch change must:  
  * Remove from previous batch  
  * Assign to new batch  
* Payment updates:  
  * Must be logged with timestamp  
  * Must include method and reference  
* Teacher:  
  * Can be assigned to multiple batches  
  * Must have at least one subject expertise  
* Employee:  
  * Non-academic role  
  * Salary always manually controlled

---

## **6\. User Flow**

---

### **Student Manual Creation**

1. Admin opens create form  
2. Inputs:  
   * Student info  
   * Parent info  
   * Class & batch  
3. Saves record  
4. System:  
   * Generates student ID  
   * Creates account

---

### **Student Batch Change**

1. Admin selects student  
2. Chooses new batch  
3. System:  
   * Removes old mapping  
   * Assigns new batch  
   * Updates schedule reference

---

### **Payment Update Flow**

1. Admin opens student/teacher profile  
2. Inputs payment details  
3. System:  
   * Saves record  
   * Updates payment summary

---

## **7\. Data Requirements**

---

### **Student**

* id  
* student\_id (generated)  
* name  
* phone  
* class\_id  
* batch\_id  
* parent\_info (JSON)  
* status  
* created\_at

---

### **Teacher**

* id  
* name  
* phone  
* subject\_expertise (array)  
* salary\_type  
* salary\_amount

---

### **Employee**

* id  
* name  
* role  
* phone  
* salary

---

### **PaymentRecord**

* id  
* user\_id  
* user\_type (STUDENT / TEACHER / EMPLOYEE)  
* amount  
* type (FEE / SALARY)  
* method  
* transaction\_id  
* created\_at

---

## **8\. API Contracts**

---

### **Student APIs**

* GET `/students`  
* GET `/students/{id}`  
* POST `/students/manual`  
* PUT `/students/{id}`  
* POST `/students/{id}/change-batch`

---

### **Teacher APIs**

* GET `/teachers`  
* POST `/teachers`  
* PUT `/teachers/{id}`

---

### **Employee APIs**

* GET `/employees`  
* POST `/employees`  
* PUT `/employees/{id}`

---

### **Payment Update**

POST `/users/payment`

{  
  "user\_id": "uuid",  
  "user\_type": "STUDENT",  
  "amount": 2000,  
  "method": "MFS",  
  "transaction\_id": "abc123"  
}

---

## **9\. Non-Functional Requirements**

* List APIs must support pagination (\< 1s response)  
* Search must support partial match  
* System must handle **10,000+ users**  
* All updates must be logged (audit-ready)

---

## **10\. Edge Cases**

* Duplicate phone number  
* Batch change mid-month (payment mismatch)  
* Payment entered twice  
* Teacher assigned but not active  
* Student without batch  
* Deactivated user but still in schedule

---

## **11\. Dependencies**

* Class & Batch module  
* Attendance module  
* Payment module  
* Admission module

---

## **12\. Assumptions**

* All users are created by admin or admission system  
* Manual payment is common (BD context)  
* No self-registration for teachers/employees

---

## **13\. Open Questions**

* Do we allow **student transfer history tracking**?  
* Should payment edits be **restricted after submission**?  
* Do we need **soft delete or permanent delete**?  
* Should inactive users be hidden or visible?

---

# **🧩 MODULE: PROFILE MANAGEMENT SYSTEM**

---

## **1\. Purpose**

Allows authenticated users to manage and update their own profile information based on role-specific permissions.

---

## **2\. Scope**

### **In Scope:**

* Profile view for all users  
* Profile update (role-based editable fields)  
* Profile picture upload/update  
* Display name and address updates

### **Out of Scope:**

* User creation (handled in User/Admission module)  
* Authentication credentials (handled in Auth module)

---

## **3\. Actors**

* Admin  
* Student  
* Teacher  
* Accountant

---

## **4\. Functional Requirements**

---

### **🔹 Profile View**

* **FR-PRO-001:** System shall allow users to view their profile  
* **FR-PRO-002:** System shall display role-specific profile data  
* **FR-PRO-003:** System shall restrict users to view only their own profile (except Admin override if needed)

---

### **🔹 Profile Update (General)**

* **FR-PRO-004:** System shall allow users to update editable profile fields  
* **FR-PRO-005:** System shall validate input before saving  
* **FR-PRO-006:** System shall persist changes immediately

---

### **🔹 Profile Picture**

* **FR-PRO-007:** System shall allow uploading profile picture  
* **FR-PRO-008:** System shall allow updating/removing profile picture  
* **FR-PRO-009:** System shall store image securely

---

### **🔹 Role-Based Permissions**

#### **Admin**

* **FR-PRO-010:** Admin shall update all editable profile properties

---

#### **Student**

* **FR-PRO-011:** Student shall update:  
  * Profile picture  
  * Display name  
  * Address

---

#### **Teacher**

* **FR-PRO-012:** Teacher shall update:  
  * Profile picture  
  * Display name  
  * Address

---

#### **Accountant**

* **FR-PRO-013:** Accountant shall update:  
  * Profile picture  
  * Display name  
  * Address

---

### **🔹 Security**

* **FR-PRO-014:** System shall require authentication for profile access  
* **FR-PRO-015:** System shall prevent unauthorized profile modification  
* **FR-PRO-016:** System shall log profile changes

---

## **5\. Business Rules**

* Email and phone are **not editable** via profile  
* Role cannot be changed from profile  
* Profile image:  
  * Max size: **2MB**  
  * Allowed formats: JPG, PNG  
* Display name:  
  * Minimum 2 characters  
  * No special characters (except space, dot)  
* Address:  
  * Free text (max 255 chars)  
* Admin:  
  * Can update additional fields if required

---

## **6\. User Flow**

---

### **Profile View Flow**

1. User logs in  
2. Navigates to profile page  
3. System fetches user data  
4. Displays role-specific fields

---

### **Profile Update Flow**

1. User edits allowed fields  
2. Clicks save  
3. System validates input  
4. Updates database  
5. Shows success message

---

### **Profile Picture Update**

1. User uploads image  
2. System validates:  
   * File type  
   * Size  
3. Upload stored  
4. Profile updated

---

## **7\. Data Requirements**

---

### **User (Extended Fields)**

* id  
* display\_name  
* profile\_picture\_url  
* address

---

### **AuditLog (Recommended)**

* id  
* user\_id  
* action (PROFILE\_UPDATE)  
* changes (JSON diff)  
* timestamp

---

## **8\. API Contracts**

---

### **GET `/profile`**

**Headers**

Authorization: Bearer \<token\>

**Response**

{  
  "id": "uuid",  
  "role": "STUDENT",  
  "display\_name": "John",  
  "profile\_picture\_url": "url",  
  "address": "Dhaka"  
}

---

---

### **PUT `/profile`**

{  
  "display\_name": "John Doe",  
  "address": "Dhaka"  
}

---

---

### **POST `/profile/upload-image`**

**Form Data**

* file

---

---

## **9\. Non-Functional Requirements**

* Profile load time \< **1 second**  
* Image upload must support **slow networks (retry support)**  
* Storage must be scalable (cloud storage recommended)  
* Secure file access (signed URLs if needed)

---

## **10\. Edge Cases**

* Upload invalid image format  
* Upload exceeds size limit  
* Partial update failure  
* Concurrent updates (last write wins or version control)  
* Missing profile fields  
* Broken image URL

---

## **11\. Dependencies**

* Authentication module  
* User Management module  
* File storage service (S3 / local / CDN)

---

## **12\. Assumptions**

* Users will access profile mostly via mobile  
* Only limited fields are editable for non-admin roles  
* Profile updates are frequent but lightweight

---

## **13\. Open Questions**

* Should Admin be able to edit **other users’ profiles from here or only User Management?**  
* Do we need **profile completion percentage (for UX)?**  
* Should we allow **changing phone/email in future with verification?**

---

# 

# 

# 

# 

# 

# 

# 

# 

# 

# 

# **🧩 MODULE: Class, Batch & Subject Context Management System**

---

## **1\. Purpose**

Defines academic structure by managing **Class (grade), Batch (group), and Subject Mapping layers**, ensuring consistent subject resolution for all dependent modules.

---

## **2\. Scope / Responsibilities**

### **✅ Handles:**

* Class creation (grade-level container)  
* Subject mapping to class (mandatory academic structure)  
* Batch creation under class  
* Subject inheritance \+ override at batch level  
* Teacher assignment per subject per batch  
* Student grouping under batch

### **❌ Does NOT handle:**

* Scheduling execution (uses this module)  
* Admission workflow (only consumes batch)  
* Exam logic (depends on subject resolution)

---

## **3\. Actors**

* Admin  
* Teacher (view \+ limited mapping visibility)  
* System (auto-resolution engine)

---

## **4\. Functional Requirements (FR)**

* FR-CBS-001: System shall allow admin to create class  
* FR-CBS-002: System shall allow admin to assign subjects to class  
* FR-CBS-003: System shall enforce class as primary subject container  
* FR-CBS-004: System shall allow admin to create batch under class  
* FR-CBS-005: System shall auto-inherit class subjects to batch  
* FR-CBS-006: System shall allow batch-level subject override  
* FR-CBS-007: System shall resolve effective subjects per batch  
* FR-CBS-008: System shall allow teacher assignment per subject per batch  
* FR-CBS-009: System shall allow student assignment to batch  
* FR-CBS-010: System shall return resolved subject-teacher mapping for downstream modules

---

## **5\. Business Logic / Rules**

---

### **🔥 5.1 Subject Hierarchy (CORE DESIGN)**

GLOBAL SUBJECTS  
      ↓  
CLASS SUBJECTS (MANDATORY)  
      ↓  
BATCH SUBJECTS (OPTIONAL OVERRIDE)  
      ↓  
BATCH-SUBJECT-TEACHER MAPPING

---

### **🔥 5.2 Subject Resolution Engine (CRITICAL)**

When any module (Scheduling / Class Access / Exam) requests subjects:

IF batch has override subjects:  
    USE batch subjects  
ELSE:  
    USE class subjects

👉 This is NOT optional — must be centralized in backend service

---

### **🔥 5.3 Teacher Mapping Rule**

* Teacher is assigned **per subject per batch**  
* NOT directly to batch only

✅ Correct:

Batch \+ Subject \+ Teacher

❌ Wrong:

Batch \+ Teacher (without subject)

---

### **🔥 5.4 Subject Ownership Constraint**

* Batch override subjects MUST belong to:  
  * Either global subject pool OR class subjects

---

### **🔥 5.5 Multi-System Support Logic**

| Use Case | Behavior |
| ----- | ----- |
| Coaching | Strict class → subject |
| National Institute | Fixed subject structure |
| Private Tutor | Auto-create “Generic Class” |

---

## **6\. User Flow / Process Flow**

---

### **FLOW 1: Academic Setup**

1. Admin creates Class  
2. Admin assigns subjects to class  
3. System locks base academic structure

---

### **FLOW 2: Batch Creation**

1. Admin selects class  
2. Creates batch  
3. System auto-inherits subjects

---

### **FLOW 3: Override (Optional)**

1. Admin selects batch  
2. Updates subject list  
3. System replaces inherited subjects

---

### **FLOW 4: Teacher Assignment**

1. Admin selects batch  
2. Selects subject  
3. Assigns teacher

---

### **FLOW 5: Subject Resolution (SYSTEM FLOW)**

1. Request comes (e.g., scheduling)  
2. System fetches batch  
3. Checks override  
4. Returns resolved subjects \+ teachers

---

## **7\. Data Model (VERY IMPORTANT – EXTENDED)**

---

## **7.1 Entities**

* Class  
* Subject  
* ClassSubject  
* Batch  
* BatchSubjectOverride  
* BatchSubjectTeacher  
* StudentBatch

---

## **7.2 Schema Definition**

---

### **📘 Class**

{  
  "id": "UUID",  
  "name": "Class 10",  
  "code": "C10",  
  "status": "ACTIVE",  
  "createdAt": "timestamp"  
}

---

### **📘 Subject**

{  
  "id": "UUID",  
  "name": "Physics",  
  "code": "PHY",  
  "isGlobal": true,  
  "status": "ACTIVE"  
}

---

### **📘 ClassSubject (MANDATORY LAYER)**

{  
  "id": "UUID",  
  "classId": "UUID",  
  "subjectId": "UUID",  
  "isMandatory": true  
}

---

### **📘 Batch**

{  
  "id": "UUID",  
  "classId": "UUID",  
  "name": "Batch A",  
  "capacity": 40,  
  "batchType": "ONLINE | OFFLINE | HYBRID",  
  "status": "ACTIVE"  
}

---

### **📘 BatchSubjectOverride (OVERRIDE LAYER)**

{  
  "id": "UUID",  
  "batchId": "UUID",  
  "subjectId": "UUID"  
}

👉 If EMPTY → fallback to ClassSubject

---

### **📘 BatchSubjectTeacher (CRITICAL TABLE)**

{  
  "id": "UUID",  
  "batchId": "UUID",  
  "subjectId": "UUID",  
  "teacherId": "UUID"  
}

👉 This table is the **core for scheduling \+ class delivery**

---

### **📘 StudentBatch**

{  
  "id": "UUID",  
  "studentId": "UUID",  
  "batchId": "UUID",  
  "status": "ACTIVE"  
}

---

## **7.3 Relationships**

* Class → has many ClassSubjects  
* Batch → belongs to Class  
* Batch → inherits ClassSubjects  
* Batch → may override subjects  
* BatchSubjectTeacher → depends on resolved subjects  
* Student → belongs to Batch

---

## **🔥 7.4 FINAL DATA RESOLUTION MODEL (NO CONFUSION)**

When system needs full structure:

Batch  
 → Class  
 → Subjects (Resolved)  
 → Teachers (via BatchSubjectTeacher)  
 → Students

---

## **8\. API Contracts**

---

### **API: Assign Subjects to Class**

POST /api/v1/classes/{classId}/subjects

{  
  "subjectIds": \["s1", "s2"\]  
}

---

### **API: Create Batch**

POST /api/v1/batches

---

### **API: Override Batch Subjects**

POST /api/v1/batches/{batchId}/subjects

---

### **API: Assign Teacher (STRICT)**

POST /api/v1/batches/{batchId}/teachers

{  
  "subjectId": "uuid",  
  "teacherId": "uuid"  
}

---

### **API: Get Batch Full Context (MOST IMPORTANT API)**

GET /api/v1/batches/{batchId}/context

Response:

{  
  "batch": {},  
  "class": {},  
  "subjects": \[\],  
  "teachers": \[\],  
  "students": \[\]  
}

👉 This API is used by:

* Scheduling  
* Class Access  
* Attendance  
* Exams

---

## **9\. UI Components**

### **Screen: Class & Subject Setup**

* Class List  
* Subject Selector (multi-select)  
* Mandatory toggle

### **Screen: Batch Management**

* Batch List  
* Capacity indicator  
* Subject Override Panel  
* Teacher Mapping Grid (Subject vs Teacher)

---

## **10\. UX Flow**

### **Subject Setup UX**

1. Admin creates class  
2. Opens subject panel  
3. Selects subjects  
4. Saves

---

### **Teacher Mapping UX**

1. Admin opens batch  
2. Sees subject list  
3. Assigns teacher per subject  
4. Saves

---

## **11\. Validation Rules**

* Class must have at least 1 subject  
* Batch must resolve at least 1 subject  
* Teacher must match subject expertise  
* Cannot assign teacher to non-existing subject  
* Override subjects must be valid

---

## **12\. Edge Cases**

* Override removed → fallback must trigger instantly  
* Subject removed but already scheduled → block  
* Teacher assigned but subject removed → invalidate  
* Empty subject set → block batch activation

---

## **13\. Dependencies**

* Admission Module  
* Scheduling Module (CRITICAL)  
* Teacher Module  
* Attendance Module

---

## **14\. Non-Functional Requirements**

* Subject resolution \< 100ms  
* High consistency (no stale mapping)  
* Concurrent-safe assignment

---

## **15\. Assumptions**

* Subjects pre-seeded  
* Teacher expertise available  
* Bangladesh curriculum supported

---

## **16\. Open Questions**

* Subject weightage needed?  
* Multi-teacher per subject allowed?

---

**🧩 MODULE: CLASS SCHEDULING SYSTEM**  
---

## **1\. Module Overview**

The Class Scheduling System manages **batch-based weekly recurring routines**, allowing admins to define, publish, and maintain structured class schedules across on-site and digital delivery modes. The system supports conflict detection, routine-level publishing, and date-specific overrides.

---

## **2\. Actors**

* **Admin** → Full control (create, edit, publish, override)  
* **System** → Validation, conflict detection, notifications  
* **Teacher** → View assigned schedule (read-only)  
* **Student** → View batch schedule via class access module

---

## **3\. Core Concepts**

### **3.1 Batch Routine**

* Represents a **weekly recurring schedule for a batch**  
* Only **one active routine per batch**  
* Status:  
  * `DRAFT`  
  * `PUBLISHED`  
  * `ARCHIVED`

---

### **3.2 Schedule Entry**

* One recurring class slot inside a routine

Defined by:  
day\_of\_week \+ time\_range \+ subject \+ teacher \+ mode

* 

---

### **3.3 Override (Exception Layer)**

* Used after routine is published  
* Types:  
  * **Occurrence Override** → affects one specific date  
  * **Future Update** → modifies recurring pattern going forward

---

## **4\. Data Model**

### **4.1 BatchRoutine**

* id (UUID)  
* batch\_id  
* status (DRAFT / PUBLISHED / ARCHIVED)  
* effective\_from (date)  
* effective\_to (nullable)  
* created\_by  
* updated\_by  
* created\_at  
* qupdated\_at

---

### **4.2 ScheduleEntry**

* id (UUID)  
* routine\_id  
* day\_of\_week (0–6)  
* start\_time  
* end\_time  
* subject\_id  
* teacher\_id  
* delivery\_mode (ON\_SITE / LIVE\_ONLINE / RECORDED\_SUPPORT / HYBRID)  
* room\_id (nullable)  
* live\_session\_ref (nullable)  
* notes (nullable)  
* created\_at  
* updated\_at

---

### **4.3 ScheduleOverride**

* id (UUID)  
* schedule\_entry\_id  
* override\_date  
* new\_start\_time (nullable)  
* new\_end\_time (nullable)  
* new\_teacher\_id (nullable)  
* new\_mode (nullable)  
* new\_room\_id (nullable)  
* new\_live\_session\_ref (nullable)  
* status (UPDATED / CANCELLED)  
* reason (nullable)  
* created\_by  
* created\_at

---

### **4.4 Room**

* id  
* name  
* capacity  
* branch\_id (optional)  
* is\_active

---

## **5\. Functional Requirements**

---

### **5.1 Routine Creation**

* Admin shall create a **weekly routine per batch**  
* System shall allow adding multiple schedule entries inside a routine  
* System shall auto-load existing routine (if exists)

---

### **5.2 Schedule Entry Management**

* Admin shall create/update/delete schedule entries  
* Each entry must include:  
  * day\_of\_week  
  * start\_time  
  * end\_time  
  * subject  
  * teacher  
  * delivery\_mode  
* System shall allow optional `notes`

---

### **5.3 Delivery Mode Handling**

Supported modes:

* ON\_SITE → requires room\_id  
* LIVE\_ONLINE → requires live\_session\_ref  
* RECORDED\_SUPPORT → optional reference  
* HYBRID → flexible mode, may switch per occurrence

---

### **5.4 Conflict Detection (HARD BLOCK)**

System shall detect:

* Batch time overlap  
* Teacher time overlap  
* Room double booking

Rules:

* Conflict detection happens during entry creation/edit  
* Conflict must **block publish**  
* Draft save is allowed with conflict

---

### **5.5 Routine Publish**

* Admin publishes entire routine (not entry-level)  
* System validates all entries before publish  
* On success:  
  * routine status → PUBLISHED  
  * schedule becomes visible downstream

---

### **5.6 Routine Update (Post-Publish)**

System shall support:

#### **A. Recurring Update**

* Modify entry for all future occurrences

#### **B. Date-specific Override**

* Modify one specific date instance

---

### **5.7 Override Behavior**

* Override shall not modify original schedule entry  
* Override shall take precedence during schedule fetch  
* Override must re-run conflict validation

---

### **5.8 Visibility Rules**

* Draft routine → not visible to teacher/student  
* Published routine → visible  
* Cancelled class → visible with status

---

### **5.9 Views**

System shall provide:

* Weekly Grid View (batch routine)  
* List View (entries with filters)

Filters:

* class  
* batch  
* teacher  
* subject  
* date range  
* status

---

### **5.10 Notifications**

System shall trigger events on:

* Routine publish  
* Entry update  
* Entry override  
* Cancellation

Recipients:

* assigned teacher  
* affected batch students

---

## **6\. Business Rules**

* Only Admin can manage scheduling  
* One active routine per batch  
* Schedule is batch-centric  
* Duration \= end\_time \- start\_time  
* No overlapping allowed:  
  * batch  
  * teacher  
  * room  
* Draft routines are invisible  
* Published routines are source of truth  
* Overrides must not break base structure

---

## **7\. User Flows**

---

### **7.1 Weekly Routine Creation Flow**

1. Admin selects batch  
2. System loads existing or empty routine  
3. Admin adds schedule entries  
4. System validates each entry inline  
5. Admin reviews full routine  
6. Admin publishes routine

---

### **7.2 Conflict Handling Flow**

1. Admin inputs entry  
2. System detects conflict instantly  
3. UI shows inline error  
4. Admin resolves conflict  
5. Publish blocked until all resolved

---

### **7.3 Routine Update Flow**

1. Admin edits entry  
2. System asks:  
   * This occurrence only  
   * This and future  
3. Admin selects option  
4. System applies update  
5. Notifications triggered

---

### **7.4 Override Flow**

1. Admin selects specific date  
2. Applies change (time/teacher/mode/etc.)  
3. System creates override record  
4. System re-validates  
5. System notifies users

---

## **8\. API Contracts**

---

### **POST `/routines`**

Create routine

---

### **POST `/routines/{id}/entries`**

Add schedule entry

---

### **PUT `/entries/{id}`**

Update entry

---

### **POST `/routines/{id}/publish`**

Publish routine

---

### **POST `/entries/{id}/override`**

Create override

Request:

{  
  "override\_date": "2026-04-12",  
  "new\_start\_time": "12:00",  
  "new\_teacher\_id": "uuid",  
  "status": "UPDATED"  
}

---

### **GET `/batches/{id}/routine`**

Returns effective schedule (with overrides applied)

---

### **GET `/teachers/{id}/schedule`**

Returns teacher-specific schedule

---

## **9\. Non-Functional Requirements**

* Conflict validation \< 2 seconds  
* Schedule load \< 2 seconds  
* High data consistency across modules  
* Transaction-safe updates  
* Audit logging required  
* Timezone fixed to Bangladesh standard

---

## **10\. Edge Cases**

* Teacher becomes inactive after assignment  
* Batch changed mid-session  
* Override conflicts with another override  
* Room deleted after assignment  
* Missing live link before class time  
* Holiday overlap (future enhancement)  
* Partial routine creation then publish attempt

---

## **11\. Dependencies**

* User Management System  
* Class & Batch Management System  
* Notification System  
* Teacher Class Management Module  
* Student Class Access Module  
* Live Class Module  
* Room Configuration

---

## **12\. Assumptions**

* Admin is sole scheduling authority  
* Batch is the core scheduling unit  
* Teachers and students consume schedule externally  
* Physical classes are primary mode  
* Online is support layer

---

## **13\. UX Behavior Requirements (IMPORTANT)**

* Inline conflict detection while editing  
* Weekly grid \+ list view both available  
* Draft vs Published clearly distinguishable  
* Recurring edit modal:  
  * This occurrence  
  * This and future  
  * Remember selection option

---

**🧩 MODULE: TEACHER CLASS WORKSPACE**

## **1\. User Story**

### **Teacher User Story**

As a teacher,  
I want to view my assigned classes, open class details, manage class-specific materials, add instructions, access live class actions, and track linked class activities,  
So that I can prepare, conduct, and support my scheduled classes properly.

### **Admin User Story**

As an admin,  
I want teachers to manage their scheduled classes only through a controlled workspace without changing the official routine,  
So that class execution remains organized while scheduling authority stays centralized.

---

## **2\. Purpose**

The Teacher Class Workspace provides the teacher-facing operational workspace for all scheduled class occurrences assigned to that teacher. It allows teachers to prepare and manage class-specific instructional materials, notes, prerecorded support clips, live class entry, attendance shortcuts, and linked academic actions without modifying the official published schedule.

This module is class-occurrence-centric, not a general content library.

Additionally, the Teacher Class Workspace provides contextual assessment shortcuts so teachers can create, view, and monitor assessments related to a specific class occurrence, batch, or subject without leaving the classroom workflow. Assessment lifecycle ownership remains under the Assessment Management module.

---

## **3\. Scope**

### **✅ In Scope**

* teacher assigned class list  
* Today / Upcoming / Previous class grouping  
* class occurrence detail view  
* class-specific teacher instructions  
* class-specific material upload  
* class-specific attachment of prerecorded clip or support resource  
* support for uploaded files and external URLs where relevant to the class  
* live class launch shortcut  
* attendance shortcut  
* linked assessment shortcut  
* class update visibility  
* occurrence-specific notes  
* student-visible publication of class resources  
* viewing class history per assigned occurrence  
* linked assessment panel inside class detail  
* create assessment with prefilled class context  
* view linked assessments for class occurrence  
* pending submission summary for linked assessments  
* result publication status summary for linked assessments

### **❌ Out of Scope**

* official routine creation  
* official routine editing  
* conflict management for schedule  
* reusable content library  
* institution-wide content repository  
* standalone file storage ownership  
* assessment engine logic  
* live session engine logic

---

## **4\. Actors**

* Teacher  
* Admin  
* System

---

## **5\. Core Concepts**

### **5.1 Assigned Class Occurrence**

A teacher-facing effective class derived from:

* published class schedule  
* override-adjusted occurrence  
* assigned batch \+ subject \+ teacher \+ date/time \+ mode

### **5.2 Class Workspace**

The operational area for one scheduled class occurrence where the teacher manages:

* instructions  
* support materials  
* prerecorded support clip if any  
* live class access  
* attendance shortcut  
* linked assessment action

### **5.3 Class Resource**

A class-specific academic resource attached to a particular class occurrence or recurring class slot.

Examples:

* worksheet PDF  
* note  
* slide deck  
* external link  
* prerecorded clip  
* revision file  
* assignment brief

### **5.4 Resource Scope**

Class resources may be attached at:

* recurring class slot level  
* specific occurrence level

### **5.5 Student Visibility Rule**

A teacher may prepare a resource without immediately publishing it to students.  
Student access depends on publication/visibility flags.

### **5.6 Linked Assessment Context**

An optional relationship where an assessment is associated with:

* a specific scheduled class occurrence  
* recurring class slot  
* batch \+ subject context

This allows teachers to launch assessment actions directly from a class workspace while keeping assessment data under the Assessment Management module.

---

## **6\. Functional Requirements**

### **🔹 6.1 Assigned Class List**

* **FR-TCW-001:** System shall show teachers only their assigned published classes.  
* **FR-TCW-002:** System shall group classes into Today, Upcoming, and Previous.  
* **FR-TCW-003:** System shall resolve applicable schedule overrides before showing classes.  
* **FR-TCW-004:** System shall hide draft schedule items from teacher view.  
* **FR-TCW-005:** Each class item shall show batch, subject, date, time, and delivery mode.  
* **FR-TCW-006:** System shall visibly mark updated, overridden, or cancelled classes.

### **🔹 6.2 Class Detail View**

* **FR-TCW-007:** Teacher shall open a class detail page for each assigned class occurrence.  
* **FR-TCW-008:** Class detail shall show:  
  * batch  
  * subject  
  * class date  
  * start time  
  * end time  
  * teacher  
  * delivery mode  
  * room or live class reference if relevant  
* **FR-TCW-009:** For ON\_SITE classes, system shall show room/location details.  
* **FR-TCW-010:** For LIVE classes, system shall show live session status and teacher host action.  
* **FR-TCW-011:** For classes with attached prerecorded support clip, system shall show that clip in class detail.  
* **FR-TCW-012:** For previous classes, system shall show class-specific historical resources and linked recording if available.

### **🔹 6.3 Class Instructions & Notes**

* **FR-TCW-013:** Teacher shall add class instructions for students.  
* **FR-TCW-014:** Teacher shall add private teacher notes for class preparation.  
* **FR-TCW-015:** Teacher shall edit or remove their own instructions and notes.  
* **FR-TCW-016:** System shall support occurrence-specific instructions.  
* **FR-TCW-017:** System shall support recurring-slot instructions if needed.  
* **FR-TCW-018:** Student-facing instructions shall be separately marked from private teacher notes.

### **🔹 6.4 Class Resource Management**

* **FR-TCW-019:** Teacher shall upload class-specific resources.  
* **FR-TCW-020:** Teacher shall attach external links as class resources.  
* **FR-TCW-021:** Teacher shall attach prerecorded support clip to a class.  
* **FR-TCW-022:** A class resource may be:  
  * uploaded file  
  * external URL  
  * prerecorded support clip  
* **FR-TCW-023:** Teacher shall add title and optional description to each resource.  
* **FR-TCW-024:** Teacher shall decide whether the resource is student-visible.  
* **FR-TCW-025:** Teacher shall define whether the resource is for one class occurrence only or for recurring slot use.  
* **FR-TCW-026:** Teacher shall edit or remove their own class resources.  
* **FR-TCW-027:** Teacher shall reorder resources if multiple resources are attached.  
* **FR-TCW-028:** System shall support attaching resource type labels such as:  
  * worksheet  
  * note  
  * prerecorded clip  
  * support file  
  * revision material

### **🔹 6.5 Prerecorded Clip Support Within Class**

* **FR-TCW-029:** Teacher shall be able to attach a prerecorded clip directly to a class occurrence.  
* **FR-TCW-030:** Prerecorded clip may be:  
  * uploaded video file  
  * external video URL  
* **FR-TCW-031:** Prerecorded clip shall remain class-linked, not library-owned.  
* **FR-TCW-032:** Teacher shall control whether students can access the prerecorded clip.  
* **FR-TCW-033:** Student access condition may be extended later without changing the class-level ownership model.  
* **FR-TCW-034:** System shall show the prerecorded clip distinctly from normal attachments in class detail.

### **🔹 6.6 Live Class Actions**

* **FR-TCW-035:** Teacher shall access live class start/join controls from class detail.  
* **FR-TCW-036:** Live class controls shall be visible only for eligible live-mode class occurrences.  
* **FR-TCW-037:** Teacher shall see session readiness state where available.  
* **FR-TCW-038:** Teacher shall be redirected to Live Class Management for session execution.  
* **FR-TCW-039:** Teacher Workspace shall not itself implement live session engine.

### **🔹 6.7 Attendance Shortcut**

* **FR-TCW-040:** Teacher shall access attendance action from a class occurrence.  
* **FR-TCW-041:** Attendance shortcut shall open the correct batch/date context where possible.  
* **FR-TCW-042:** Teacher Workspace shall not own attendance data logic.

### **🔹 6.8 Linked Assessment Shortcut**

* **FR-TCW-043:** Teacher shall access assessment creation shortcut from class workspace where permitted.  
* **FR-TCW-044:** Assessment creation flow shall carry prefilled context where available:  
  * batch  
  * subject  
  * teacher  
  * linked schedule entry  
  * class date (optional)  
* **FR-TCW-045:** Teacher shall view assessments linked to the class occurrence.  
* **FR-TCW-046:** Teacher shall view assessment summary for linked items including:  
  * draft / published state  
  * submission count  
  * pending evaluation count  
  * result publication state  
* **FR-TCW-047:** Teacher shall open linked assessments inside Assessment Management module.  
* **FR-TCW-048:** Teacher Workspace shall not own assessment creation engine, submission engine, evaluation engine, or result engine.

### **🔹 6.9 Filtering & Navigation**

* **FR-TCW-049:** Teacher shall filter classes by subject.  
* **FR-TCW-050:** Teacher shall filter classes by batch.  
* **FR-TCW-051:** Teacher shall filter classes by delivery mode.  
* **FR-TCW-052:** Teacher shall search by batch or subject name.  
* **FR-TCW-053:** System shall support date-based navigation for previous and upcoming classes.

---

## **7\. Business Rules**

* Teacher Workspace consumes published effective schedule only.  
* Official schedule changes remain admin-owned.  
* Teachers may manage only classes assigned to them.  
* Resources attached here are class-linked, not reusable library assets.  
* Prerecorded clip attached here is part of class support, not a general content library item.  
* Student visibility must be explicitly controlled for attached resources.  
* Previous classes may retain historical resources and linked recordings.  
* Cancelled classes may remain visible historically with status marking.  
* Teacher notes and student instructions must be logically separated.  
* Live class functionality must redirect to Live Class Management module.  
* Assessment linkage from class workspace shall be optional.  
* A class occurrence may have zero, one, or multiple linked assessments.  
* Linked assessments remain owned by Assessment Management module.  
* Deleting or changing a schedule shall not automatically delete linked assessments unless explicitly configured.  
* Class workspace may display assessment summaries but not alter evaluation logic.

---

## **8\. User Flow / Process Flow**

### **8.1 Teacher Daily Class Flow**

1. Teacher logs in.  
2. Opens Teacher Class Workspace.  
3. System loads assigned published classes.  
4. Classes are grouped into:  
   * Today  
   * Upcoming  
   * Previous  
5. Teacher opens one class occurrence.  
6. Teacher reviews time, batch, subject, and mode.  
7. Teacher chooses one or more actions:  
   * add instruction  
   * upload support material  
   * attach prerecorded clip  
   * start live class  
   * open attendance  
   * open linked assessment

### **8.2 Attach Class Resource Flow**

1. Teacher opens class detail.  
2. Clicks Add Resource.  
3. Chooses resource type:  
   * file  
   * external link  
   * prerecorded clip  
4. Adds title and visibility.  
5. Selects scope:  
   * this occurrence only  
   * recurring class slot  
6. Saves.  
7. System validates and stores resource.  
8. Resource appears in class detail.

### **8.3 Attach Prerecorded Clip Flow**

1. Teacher opens a class occurrence.  
2. Clicks Attach Prerecorded Clip.  
3. Chooses:  
   * upload file  
   * external video URL  
4. Adds title/description.  
5. Sets student visibility.  
6. Saves.  
7. System marks the resource as class-level prerecorded support.  
8. Students can access it later if published.

### **8.4 Previous Class Review Flow**

1. Teacher opens Previous classes.  
2. Selects one historical class.  
3. Sees:  
   * past instructions  
   * attached resources  
   * linked recording if any  
   * attendance shortcut history reference if needed later

### **8.5 Create Assessment From Class Flow**

1. The teacher opens class details.  
2. Clicks Create Assessment.  
3. System opens Assessment module with prefilled:  
   * batch  
   * subject  
   * linked class reference  
4. The teacher selects the assessment type.  
5. Completes setup and saves/publishes.  
6. Linked assessment now appears in the class assessment panel.

### **8.6 Review Linked Assessments Flow**

1. The teacher opens class details.  
2. Open Assessment panel.  
3. Sees linked assessments with statuses.  
4. Opens selected items in the Assessment module for evaluation or management.

---

## **9\. Data Model**

### **📘 TeacherAssignedClassView**

| Field | Type | Description |
| ----- | ----- | ----- |
| schedule\_entry\_id | UUID | Schedule entry |
| batch\_id | UUID | Batch |
| batch\_name | String | Batch name |
| class\_id | UUID | Academic class |
| subject\_id | UUID | Subject |
| subject\_name | String | Subject |
| teacher\_id | UUID | Teacher |
| class\_date | Date | Effective occurrence date |
| start\_time | Time | Start |
| end\_time | Time | End |
| delivery\_mode | Enum | ON\_SITE / LIVE / HYBRID / SUPPORT\_ONLY |
| room\_name | String nullable | Room |
| live\_session\_ref | String nullable | Live session |
| effective\_status | Enum | NORMAL / UPDATED / CANCELLED / OVERRIDDEN |

### **📘 ClassInstruction**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| schedule\_entry\_id | UUID | Class slot |
| class\_date | Date nullable | Occurrence date |
| teacher\_id | UUID | Teacher |
| instruction\_type | Enum | STUDENT\_VISIBLE / TEACHER\_PRIVATE |
| content | Text | Note content |
| created\_at | Timestamp | Created time |
| updated\_at | Timestamp | Updated time |

### **📘 ClassResource**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| schedule\_entry\_id | UUID | Class slot |
| class\_date | Date nullable | Occurrence-specific ref |
| teacher\_id | UUID | Uploader |
| title | String | Resource title |
| description | Text nullable | Description |
| resource\_type | Enum | FILE / LINK / PRERECORDED\_CLIP |
| file\_asset\_id | UUID nullable | File reference |
| external\_url | String nullable | URL |
| visibility\_status | Enum | DRAFT / PUBLISHED / HIDDEN |
| apply\_scope | Enum | OCCURRENCE / RECURRING\_SLOT |
| display\_order | Integer | Ordering |
| created\_at | Timestamp | Created time |

### **📘 ClassAssessmentLinkView**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| schedule\_entry\_id | UUID | Linked class |
| assessment\_id | UUID | Assessment ref |
| assessment\_title | String | Title |
| assessment\_type | Enum | Type |
| status | Enum | DRAFT / PUBLISHED / CLOSED |
| submission\_count | Integer | Total submissions |
| pending\_evaluation\_count | Integer | Pending review |
| result\_status | Enum | NOT\_PUBLISHED / PUBLISHED |

---

## **10\. API Contracts**

### **Get Teacher Classes**

**GET** `/teacher/classes`

**Query:**

* `group=today|upcoming|previous`  
* `subject_id`  
* `batch_id`  
* `mode`  
* `date_from`  
* `date_to`

### **Get Class Detail**

**GET** `/teacher/classes/{schedule_entry_id}`

### **Add Class Instruction**

**POST** `/teacher/classes/{schedule_entry_id}/instructions`

{  
  "instruction\_type": "STUDENT\_VISIBLE",  
  "content": "Bring your math notebook tomorrow.",  
  "class\_date": "2026-04-20"  
}

### **Add Class Resource**

**POST** `/teacher/classes/{schedule_entry_id}/resources`

{  
  "title": "Worksheet Chapter 4",  
  "resource\_type": "FILE",  
  "file\_asset\_id": "uuid",  
  "visibility\_status": "PUBLISHED",  
  "apply\_scope": "OCCURRENCE",  
  "class\_date": "2026-04-20"  
}

### **Attach Prerecorded Clip**

**POST** `/teacher/classes/{schedule_entry_id}/prerecorded-clips`

{  
  "title": "Pre-recorded explanation",  
  "resource\_type": "PRERECORDED\_CLIP",  
  "external\_url": "https://example.com/video/abc",  
  "visibility\_status": "PUBLISHED",  
  "class\_date": "2026-04-20"  
}

### **Update Resource**

**PUT** `/teacher/class-resources/{id}`

### **Delete Resource**

**DELETE** `/teacher/class-resources/{id}`

### **Get Linked Assessments For Class**

**GET** `/teacher/classes/{schedule_entry_id}/assessments`

### **Open Prefilled Create Assessment Context**

**GET** `/teacher/classes/{schedule_entry_id}/assessment-context`

**Response example:**

{  
  "batch\_id": "uuid",  
  "subject\_id": "uuid",  
  "linked\_schedule\_entry\_id": "uuid"  
}

---

## **11\. UI Components**

### **Teacher Class List Screen**

* tab/group switcher: Today / Upcoming / Previous  
* subject filter  
* batch filter  
* mode filter  
* search  
* class cards/list rows

### **Class Card / Row**

Show:

* subject  
* batch  
* time  
* mode  
* status badge

Quick actions:

* open  
* start live  
* attendance  
* resources

### **Class Detail Screen**

Sections:

* class header summary  
* instructions panel  
* class resources panel  
* prerecorded clip panel  
* live class action panel  
* assessment shortcut panel  
* assessment panel  
* create assessment button  
* linked assessments list  
* pending evaluation badge  
* result publication status badge  
* attendance shortcut

### **Resource Add Modal / Form**

* resource type selector  
* title  
* description  
* upload or external URL field  
* student visibility switch  
* occurrence/recurring scope selector

---

## **12\. UX Flow**

### **Teacher UX**

The teacher should feel that this module is the operational home for each scheduled class.

The teacher should not feel like they are managing a generic content repository.

### **UX Priorities**

* next class visibility  
* quick classroom actions  
* class-specific support material management  
* previous class review  
* linked academic shortcuts

Teacher should be able to move from teaching workflow to assessment workflow in one click without re-entering batch/subject context.

---

## **13\. Validation Rules**

* teacher must be assigned to class  
* student-visible instruction content cannot be empty  
* class resource title required  
* resource must have `file_asset_id` or `external_url` depending on type  
* prerecorded clip must have valid source  
* external URL must be valid format  
* occurrence-specific resource must include `class_date` where applicable  
* teacher cannot modify another teacher’s resources unless admin override exists

---

## **14\. Edge Cases**

* schedule updated after teacher already attached resources  
* class cancelled after resources were published  
* live class reference missing  
* teacher attaches duplicate resource twice  
* prerecorded clip link later breaks  
* recurring-slot resource becomes unsuitable for one override occurrence  
* teacher reassigned after resource creation  
* previous class remains visible but one resource is archived  
* class has multiple linked assessments  
* linked assessment exists after schedule override  
* teacher reassigned after linked assessment creation  
* assessment published but class later cancelled  
* result published after class has moved to history

---

## **15\. Dependencies**

* Class Scheduling System  
* Live Class Management  
* Attendance Management  
* Assessment Management  
* Notification Management  
* Shared File Asset & Document Management  
* Authentication & Authorization

---

## **16\. Non-Functional Requirements**

* teacher class list should load quickly under normal usage  
* class detail should support mobile usage  
* file/link resource management should tolerate weak internet  
* class-linked resources must remain permission-safe  
* occurrence rendering must always respect schedule override truth

---

## **17\. Assumptions**

* teachers do not edit the official routine  
* prerecorded support clips are class-linked, not library-owned  
* many teachers may use mobile devices  
* one class may have both live and support resources

---

## **18\. Open Questions**

* should teachers be able to mark class as completed later?  
* should teacher private notes be visible to admin?  
* should student access to prerecorded clip be condition-based later?

---

# 

# 

# 

# **🧩 MODULE: STUDENT CLASS WORKSPACE**

## **1\. User Story**

### **Student User Story**

As a student,  
I want to view my scheduled classes, open class details, join live classes, read teacher instructions, access class-specific materials, and review previous class resources,  
So that I can attend and follow my classes properly from one organized space.

### **Admin User Story**

As an admin,  
I want students to consume classes from a controlled class-based workspace that follows the official published schedule,  
So that academic access remains structured and role-safe.

---

## **2\. Purpose**

The Student Class Workspace provides student-facing access to scheduled class occurrences based on the student’s assigned batch and effective published routine. It lets the student view Today, Upcoming, and Previous classes, access class-specific instructions and resources, join ongoing live sessions, and review previously published class materials including class-linked prerecorded support clips and linked class recordings.

This module is class-centric, not a generic content library.

Additionally, the Student Class Workspace may surface assessments that are directly linked to a class occurrence or relevant academic context, allowing students to discover and open class-related quizzes, assignments, and tests naturally from their classroom journey.

---

## **3\. Scope**

### **✅ In Scope**

* student class list by effective schedule  
* Today / Upcoming / Previous grouping  
* class occurrence detail  
* live class join access  
* teacher instruction display  
* class-specific materials  
* class-linked prerecorded clip access  
* previous class linked recording access  
* room/location display for on-site classes  
* batch-safe access control  
* class-level history view  
* linked class assessment visibility  
* open active assessment from class detail  
* assignment due indicator  
* result published indicator for linked assessments  
* continue attempt shortcut for active linked assessments

### **❌ Out of Scope**

* official routine editing  
* generic reusable content library  
* independent live session engine  
* assessment engine core  
* full attendance analytics  
* generic file storage ownership

---

## **4\. Actors**

* Student  
* System  
* Teacher (indirect contributor)  
* Admin (indirect owner of published schedule)

---

## **5\. Core Concepts**

### **5.1 Effective Class Occurrence**

A student-visible class occurrence resolved from:

* published routine  
* applicable override  
* current batch mapping  
* class date/time/mode context

### **5.2 Class Detail**

A student-facing page for one class occurrence showing:

* subject  
* teacher  
* time  
* mode  
* instructions  
* attached resources  
* prerecorded support clip if published  
* linked recording if available for previous class

### **5.3 Previous Class Resource Linkage**

A completed class may retain:

* student-visible materials  
* instructions  
* linked live recording  
* class-linked prerecorded support clip

### **5.4 Join Window**

For live classes, join action is controlled by current time and live session readiness.

### **5.5 Class Assessment Linkage**

A student-visible contextual relationship where an assessment is relevant to a class because it is:

* explicitly linked to that class occurrence  
* created for the same batch \+ subject  
* intended as post-class homework or quiz

Assessment ownership remains under Assessment Management.

---

## **6\. Functional Requirements**

### **🔹 6.1 Class List View**

* **FR-SCW-001:** System shall show student only classes belonging to their assigned batch.  
* **FR-SCW-002:** System shall group classes into Today, Upcoming, and Previous.  
* **FR-SCW-003:** System shall resolve schedule overrides before showing classes.  
* **FR-SCW-004:** System shall hide draft routines from students.  
* **FR-SCW-005:** System shall visibly mark cancelled or updated classes where policy requires.  
* **FR-SCW-006:** Each class item shall show subject, teacher, date/time, and mode.

### **🔹 6.2 Class Detail View**

* **FR-SCW-007:** Student shall open class detail for any visible class occurrence.  
* **FR-SCW-008:** Class detail shall show:  
  * subject  
  * teacher  
  * date  
  * time  
  * mode  
  * room/location if on-site  
* **FR-SCW-009:** Student shall see student-visible teacher instructions.  
* **FR-SCW-010:** Student shall see class-specific student-visible materials.  
* **FR-SCW-011:** Student shall see class-linked prerecorded clip if the teacher has published it.  
* **FR-SCW-012:** For previous classes, student shall see linked recording if available.  
* **FR-SCW-013:** Student shall not see teacher-private notes.

### **🔹 6.3 Live Class Access**

* **FR-SCW-014:** Student shall be able to join live class only during valid join window.  
* **FR-SCW-015:** For future live classes, system shall show not-started state.  
* **FR-SCW-016:** For ongoing live classes, system shall show join action.  
* **FR-SCW-017:** If live session reference is unavailable during valid class window, system shall show unavailable state.  
* **FR-SCW-018:** Student Workspace shall redirect to Live Class Management for actual live access flow.

### **🔹 6.4 Class Resource Access**

* **FR-SCW-019:** Student shall access class-specific resources published for them.  
* **FR-SCW-020:** Resource may be file, external link, or prerecorded clip.  
* **FR-SCW-021:** Student shall open/download supported resources according to permission.  
* **FR-SCW-022:** Student shall not access hidden or unpublished class resources.

### **🔹 6.5 Previous Class Review**

* **FR-SCW-023:** Completed classes shall move to Previous group after class end.  
* **FR-SCW-024:** Previous class detail shall remain accessible for learning review.  
* **FR-SCW-025:** Previous class detail may include:  
  * instructions  
  * materials  
  * class-linked prerecorded clip  
  * linked live recording  
* **FR-SCW-026:** Previous classes shall remain visible according to configured retention rules.

### **🔹 6.6 Linked Assessment Visibility**

* **FR-SCW-027:** Student shall see assessments linked to the class occurrence where eligible.  
* **FR-SCW-028:** Linked assessments may show status such as:  
  * UPCOMING  
  * ACTIVE  
  * SUBMITTED  
  * RESULT\_PUBLISHED  
  * CLOSED  
* **FR-SCW-029:** Student shall open active linked assessments in Assessment Management module.  
* **FR-SCW-030:** Student shall continue in-progress linked attempts where allowed.  
* **FR-SCW-031:** Student shall view linked result shortcut when published.  
* **FR-SCW-032:** Student shall not access assessments outside eligibility rules even if class-linked.

### **🔹 6.7 Navigation & Filtering**

* **FR-SCW-033:** Student shall navigate classes by Today, Upcoming, and Previous.  
* **FR-SCW-034:** Student shall filter by subject if enabled.  
* **FR-SCW-035:** Student shall filter by date range if enabled.  
* **FR-SCW-036:** Student shall search by subject name if enabled.

---

## **7\. Business Rules**

* Student Class Workspace is derived from published effective schedule only.  
* Batch membership determines student class visibility.  
* A student may only access classes belonging to their own batch.  
* Live join action must respect timing and session validity.  
* Class-linked prerecorded clips belong to the class, not to a separate content library.  
* Previous class review is allowed only for student-visible resources.  
* Student must never see teacher-private notes.  
* Schedule changes must be reflected in workspace rendering.  
* Assessment visibility from class workspace is optional and context-driven.  
* A class may have zero, one, or multiple linked assessments.  
* Student eligibility rules from Assessment module override class linkage visibility.  
* Class workspace only surfaces assessment status and navigation.  
* Attempt, submission, evaluation, and result logic remain under Assessment Management.

---

## **8\. User Flow / Process Flow**

### **8.1 Student Class Access Flow**

1. Student logs in.  
2. Opens Student Class Workspace.  
3. System resolves student batch.  
4. System loads effective published classes.  
5. Classes are grouped into:  
   * Today  
   * Upcoming  
   * Previous  
6. Student opens one class occurrence.  
7. Student sees class details and available actions.

### **8.2 Future Live Class Flow**

1. Student opens upcoming live class.  
2. System checks current time.  
3. Since class has not started yet:  
   * show schedule  
   * show not started state  
4. Join action remains disabled or unavailable.

### **8.3 Ongoing Live Class Flow**

1. Student opens class during active join window.  
2. System validates live session availability.  
3. Join button becomes active.  
4. Student is redirected to live class access flow.

### **8.4 Previous Class Review Flow**

1. Student opens Previous classes.  
2. Selects one class.  
3. System shows:  
   * teacher instructions  
   * class files  
   * class-linked prerecorded support clip if published  
   * linked recording if available

### **8.5 Open Class Assignment Flow**

1. Student opens class detail.  
2. Sees linked assignment in Assessment panel.  
3. Clicks Open Assessment.  
4. System redirects to Assessment module.  
5. Student completes submission.

### **8.6 View Result From Class Flow**

1. Student opens previous class detail.  
2. Sees linked assessment marked Result Published.  
3. Clicks View Result.  
4. System opens result screen in Assessment module.

---

## **9\. Data Model**

### **📘 StudentClassView**

| Field | Type | Description |
| ----- | ----- | ----- |
| schedule\_entry\_id | UUID | Class slot |
| class\_date | Date | Effective date |
| batch\_id | UUID | Batch |
| batch\_name | String | Batch |
| subject\_id | UUID | Subject |
| subject\_name | String | Subject |
| teacher\_name | String | Teacher |
| delivery\_mode | Enum | ON\_SITE / LIVE / HYBRID / SUPPORT\_ONLY |
| room\_name | String nullable | Room |
| start\_time | Time | Start |
| end\_time | Time | End |
| effective\_status | Enum | UPCOMING / ONGOING / PREVIOUS / CANCELLED |
| live\_session\_ref | String nullable | Live ref |

### **📘 StudentClassInstructionView**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| schedule\_entry\_id | UUID | Class slot |
| class\_date | Date nullable | Date |
| content | Text | Instruction |
| created\_at | Timestamp | Created time |

### **📘 StudentClassResourceView**

| Field | Type | Description |
| ----- | ----- | ----- |
| resource\_id | UUID | Resource ref |
| schedule\_entry\_id | UUID | Class slot |
| class\_date | Date nullable | Occurrence |
| title | String | Resource title |
| resource\_type | Enum | FILE / LINK / PRERECORDED\_CLIP |
| file\_asset\_id | UUID nullable | File ref |
| external\_url | String nullable | URL |
| visibility\_status | Enum | PUBLISHED |
| display\_order | Integer | Ordering |

### **📘 StudentClassRecordingView**

| Field | Type | Description |
| ----- | ----- | ----- |
| recording\_id | UUID | Recording ref |
| schedule\_entry\_id | UUID | Class slot |
| class\_date | Date | Date |
| title | String | Recording title |
| recording\_url | String | Playback URL |
| recording\_status | Enum | AVAILABLE / PROCESSING / UNAVAILABLE |

### **📘 StudentClassAssessmentView**

| Field | Type | Description |
| ----- | ----- | ----- |
| assessment\_id | UUID | Assessment ref |
| schedule\_entry\_id | UUID | Class ref |
| title | String | Assessment title |
| assessment\_type | Enum | Type |
| student\_status | Enum | UPCOMING / ACTIVE / IN\_PROGRESS / SUBMITTED / RESULT\_PUBLISHED / CLOSED |
| due\_at | Timestamp nullable | Due time |
| score\_available | Boolean | Result available |

---

## **10\. API Contracts**

### **Get Student Classes**

**GET** `/student/classes`

Query:

* `group=today|upcoming|previous`  
* `subject_id`  
* `date_from`  
* `date_to`

### **Get Class Detail**

**GET** `/student/classes/{schedule_entry_id}`

### **Get Class Resources**

**GET** `/student/classes/{schedule_entry_id}/resources`

### **Get Class Instructions**

**GET** `/student/classes/{schedule_entry_id}/instructions`

### **Get Class Recording**

**GET** `/student/classes/{schedule_entry_id}/recording`

### **Get Linked Assessments For Student Class**

**GET** `/student/classes/{schedule_entry_id}/assessments`

### **Open Linked Assessment**

**GET** `/student/classes/{schedule_entry_id}/assessments/{assessment_id}`  
(redirect/context endpoint)

---

## **11\. UI Components**

### **Student Class List Screen**

* tabs: Today / Upcoming / Previous  
* class cards/list  
* subject filter  
* date filter  
* search

### **Class Card**

Show:

* subject  
* teacher  
* time  
* mode  
* status  
* open/join button

### **Class Detail Screen**

Sections:

* class summary  
* instruction panel  
* resources panel  
* prerecorded clip panel  
* linked recording panel for previous classes  
* live class join area when relevant  
* assessment panel  
* assignment due badge  
* active quiz badge  
* continue attempt button  
* view result button

---

## **12\. UX Flow**

### **Student UX**

The student should feel that this is the place to manage real class participation and review.  
It should not feel like a generic media/content app.

### **UX Priorities**

* today’s class  
* ongoing live join  
* upcoming classes  
* previous class review  
* class-linked support resources

Students should naturally discover class-related homework, quizzes, and results from the class detail screen without needing to separately search the Assessment module.

---

## **13\. Validation Rules**

* student must belong to visible batch  
* student-visible resources only  
* join allowed only within valid window  
* hidden/cancelled rules must be respected  
* previous class recording shown only if available and allowed

---

## **14\. Edge Cases**

* batch changed mid-week  
* class cancelled after student saw it  
* live session missing  
* prerecorded support clip unpublished after prior access  
* previous class recording processing delayed  
* external link resource broken  
* override changes same-day time  
* class has multiple active assessments  
* assessment due after class date  
* class cancelled but assessment remains valid  
* student changed batch after assessment publication  
* result published after class moved to previous history  
* linked assessment closed before student opens class detail

---

## **15\. Dependencies**

* Class Scheduling System  
* Teacher Class Workspace  
* Live Class Management  
* Notification Management  
* Assessment Management  
* Shared File Asset & Document Management  
* Authentication & Authorization

---

## **16\. Non-Functional Requirements**

* student class list should load quickly  
* mobile responsiveness mandatory  
* weak internet conditions should still allow class data viewing  
* access control must prevent cross-batch leakage  
* previous class review must remain stable after schedule changes

---

## **17\. Assumptions**

* students access classes primarily by batch  
* prerecorded support clips remain class-linked  
* many students will use mobile devices  
* previous class review is important for coaching continuity

---

## **18\. Open Questions**

* should previous classes remain visible forever or by retention period?  
* should students later get conditional access to prerecorded clip?  
* should class resources show viewed/downloaded state later?

# 

# 

# 

# 

# **🧩 MODULE: LIVE CLASS MANAGEMENT**

## **1\. User Story**

### **Teacher User Story**

As a teacher,  
I want to start and host scheduled live classes securely from my class workspace,  
So that I can conduct online sessions without manually managing meeting logistics outside the LMS.

### **Student User Story**

As a student,  
I want to join only my scheduled live classes securely from my class workspace,  
So that I can attend online sessions easily and safely.

### **Admin User Story**

As an admin,  
I want live classes to follow scheduled academic context, capture attendance, and optionally return recordings to the class flow,  
So that online classes remain disciplined and trackable.

---

## **2\. Purpose**

The Live Class Management module manages the execution lifecycle of live online class sessions linked to scheduled class occurrences. It handles secure live session creation/reference, teacher host flow, student join flow, session state, attendance capture hooks, and post-class recording linkage.

This module owns live session execution, not class listing or class content management.

---

## **3\. Scope**

### **✅ In Scope**

* session creation/reference for scheduled live classes  
* teacher host access  
* student join access  
* secure participant validation  
* session lifecycle status  
* attendance join/leave tracking  
* session readiness state  
* recording reference linkage  
* provider integration abstraction

### **❌ Out of Scope**

* student class list  
* teacher assigned class list  
* generic content library  
* scheduling authoring  
* general class materials  
* assessment workflows

---

## **4\. Actors**

* Teacher  
* Student  
* Admin  
* System  
* External Meeting Provider

---

## **5\. Core Concepts**

### **5.1 Live Session**

A scheduled online class session attached to one class occurrence.

### **5.2 Session Roles**

* HOST (teacher)  
* PARTICIPANT (student)  
* ADMIN\_VIEW optional later

### **5.3 Session Lifecycle**

* UPCOMING  
* READY  
* LIVE  
* ENDED  
* CANCELLED

### **5.4 Join Window**

The allowed time range in which a student may join.

### **5.5 Recording Reference**

A reference to class recording returned after live class completion if provider supports it.

---

## **6\. Functional Requirements**

### **🔹 6.1 Session Creation & Binding**

* FR-LCM-001: System shall support live session creation or reference binding for scheduled live-mode classes.  
* FR-LCM-002: A live session shall be linked to one class occurrence.  
* FR-LCM-003: Teacher host access shall be linked to assigned class context.  
* FR-LCM-004: Student join access shall be restricted to eligible class participants.

### **🔹 6.2 Session Access**

* FR-LCM-005: Teacher shall start/join session as host.  
* FR-LCM-006: Student shall join session only through authorized join flow.  
* FR-LCM-007: System shall block unauthorized users from joining.  
* FR-LCM-008: Join tokens or equivalent secure access method shall be supported.  
* FR-LCM-009: System shall support time-based join eligibility.

### **🔹 6.3 Session Lifecycle**

* FR-LCM-010: System shall track session status.  
* FR-LCM-011: Session shall move to LIVE when class begins or host starts session.  
* FR-LCM-012: Session shall move to ENDED after closure.  
* FR-LCM-013: Cancelled session shall not allow join.  
* FR-LCM-014: Session readiness information shall be available to teacher/student workspaces.

### **🔹 6.4 Attendance Hooks**

* FR-LCM-015: System shall capture participant join time.  
* FR-LCM-016: System shall capture participant leave time.  
* FR-LCM-017: System shall capture attendance duration where possible.  
* FR-LCM-018: Attendance event data shall be consumable by Attendance module.  
* FR-LCM-019: Live class attendance tracking shall not replace manual attendance policy unless configured.

### **🔹 6.5 Recording Linkage**

* FR-LCM-020: If recording is available, system shall store recording reference.  
* FR-LCM-021: Recording reference shall be linkable back to class occurrence.  
* FR-LCM-022: Teacher/Admin may later expose recording through class workspace if policy allows.  
* FR-LCM-023: Live Class module shall not own reusable library publication logic.

### **🔹 6.6 Monitoring**

* FR-LCM-024: Admin shall be able to view active live sessions summary if required.  
* FR-LCM-025: System shall store essential session logs.  
* FR-LCM-026: System shall support failure state logging for session issues.

---

## **7\. Business Rules**

* Live session belongs to a scheduled class occurrence.  
* Only authorized teacher may host the session.  
* Only eligible students may join.  
* Join action must respect timing rules.  
* Session execution is separate from class listing UI.  
* Recording reference, if produced, belongs first to the class occurrence.  
* Live attendance data may support attendance records but does not automatically define final attendance policy unless configured.

---

## **8\. User Flow / Process Flow**

### **8.1 Teacher Live Class Flow**

1. Teacher opens class occurrence from Teacher Class Workspace.  
2. Clicks Start Live Class.  
3. System validates teacher and session state.  
4. Teacher enters host session flow.  
5. Session becomes LIVE.  
6. Session logs are tracked.  
7. Teacher ends class.  
8. Recording reference is stored if available.

### **8.2 Student Join Flow**

1. Student opens class occurrence from Student Class Workspace.  
2. System checks current time and session state.  
3. If join is allowed, Join button appears.  
4. Student joins through secure access.  
5. Join/leave timestamps are logged.

### **8.3 Recording Return Flow**

1. Live class ends.  
2. External provider returns recording reference if enabled.  
3. System stores reference.  
4. Previous class detail may later show linked recording if published for student visibility.

---

## **9\. Data Model**

### **📘 LiveSession**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| schedule\_entry\_id | UUID | Class slot |
| class\_date | Date | Occurrence date |
| provider | Enum | Provider |
| provider\_session\_ref | String | Provider ref |
| host\_teacher\_id | UUID | Host teacher |
| status | Enum | UPCOMING / READY / LIVE / ENDED / CANCELLED |
| start\_at | Timestamp | Planned start |
| end\_at | Timestamp | Planned end |

### **📘 LiveParticipantEvent**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| session\_id | UUID | Session |
| user\_id | UUID | User |
| role | Enum | HOST / PARTICIPANT |
| joined\_at | Timestamp | Join |
| left\_at | Timestamp nullable | Leave |
| duration\_sec | Integer nullable | Duration |

### **📘 LiveRecordingReference**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| session\_id | UUID | Session |
| recording\_ref | String | Recording reference |
| playback\_url | String nullable | Playback |
| status | Enum | PROCESSING / AVAILABLE / FAILED |

---

## **10\. API Contracts**

### **Get Live Session State**

`GET /live-classes/{schedule_entry_id}`

### **Start Live Session**

`POST /live-classes/{schedule_entry_id}/start`

### **Join Live Session**

`POST /live-classes/{schedule_entry_id}/join`

### **End Live Session**

`POST /live-classes/{schedule_entry_id}/end`

### **Get Recording Reference**

`GET /live-classes/{schedule_entry_id}/recording`

---

## **11\. UI Components**

### **Teacher Live Class Panel**

* session state badge  
* start class button  
* provider status  
* participant summary placeholder if available

### **Student Live Class Panel**

* not started state  
* join button  
* unavailable state  
* ended state

### **Admin Monitoring View**

* active live sessions list  
* session status  
* basic counts

---

## **12\. UX Flow**

### **Teacher UX**

Teacher should access live class from class workspace, not by hunting for a separate provider tool.

### **Student UX**

Student should join the right class from the right class detail screen with minimal confusion.

---

## **13\. Validation Rules**

* session must belong to scheduled live class  
* teacher must match assigned teacher  
* student must belong to eligible batch  
* join window must be valid  
* cancelled sessions cannot be joined  
* recording cannot be exposed unless reference exists

---

## **14\. Edge Cases**

* provider unavailable  
* teacher starts late  
* student joins after start  
* duplicate join attempts  
* provider recording delayed  
* session ends without recording  
* join button shown but token generation fails

---

## **15\. Dependencies**

* Class Scheduling System  
* Teacher Class Workspace  
* Student Class Workspace  
* Attendance Management  
* Notification Management  
* Authentication & Authorization

---

## **16\. Non-Functional Requirements**

* live access flow must be secure  
* session status should update reliably  
* student join flow should be mobile-friendly  
* provider failures should degrade gracefully  
* attendance event logging should remain consistent

---

## **17\. Assumptions**

* external meeting provider will be used initially  
* live classes are linked to scheduled classes  
* recording availability depends on provider capability

---

## **18\. Open Questions**

* which provider will be first integrated?  
* should teacher be able to start early?  
* should late student joins affect attendance logic?

---

# 

# 

# 

# **🧩 MODULE: ASSESSMENT MANAGEMENT**

## **1\. User Story**

### **Student User Story**

As a student,  
I want to take assigned quizzes, class tests, exams, and assignments, view results, and understand weak areas,  
So that I can measure my performance and improve continuously.

### **Teacher User Story**

As a teacher,  
I want to create and manage assessments for my batches/subjects, evaluate submissions, publish results, and review student performance,  
So that I can assess academic progress effectively.

### **Admin User Story**

As an admin,  
I want the LMS to support structured assessments, controlled publication, and meaningful progress tracking,  
So that the coaching center can maintain quality academic evaluation.

---

## **2\. Purpose**

The Assessment Management module manages the full academic assessment lifecycle including quizzes, class tests, assignments, scheduled exams, student submissions, evaluation, result publication, and performance analysis. It supports both objective and teacher-reviewed assessment formats and can be linked to class, batch, and subject context where relevant.

This module remains separate from class workspaces, but may be launched from them through contextual shortcuts.

---

## **3\. Scope**

### **✅ In Scope**

* assessment creation  
* quiz/practice/class test/exam/assignment support  
* batch and subject mapping  
* optional class-linked assessment context  
* question authoring  
* objective and written answer support  
* assignment file submission  
* student attempt flow  
* timer-based live exam support  
* auto evaluation for objective items  
* manual evaluation for written items  
* result publication  
* student result view  
* weak chapter/topic insights  
* batch comparison summaries

### **❌ Out of Scope**

* government board registration  
* AI proctoring  
* OCR handwritten answer interpretation  
* advanced plagiarism engine  
* public certificate generation

---

## **4\. Actors**

* Teacher  
* Student  
* Admin  
* System

---

## **5\. Core Concepts**

### **5.1 Assessment**

An academic evaluation unit.

Possible assessment types:

* PRACTICE\_QUIZ  
* CLASS\_TEST  
* ASSIGNMENT  
* LIVE\_EXAM  
* MONTHLY\_EXAM  
* MODEL\_TEST

### **5.2 Assessment Context**

Each assessment may be mapped to:

* batch  
* subject  
* class  
* chapter  
* optional linked class occurrence

### **5.3 Objective Assessment**

Automatically evaluated items:

* MCQ  
* true/false  
* single-answer type

### **5.4 Subjective Assessment**

Teacher-reviewed items:

* written answer  
* file submission  
* long response

### **5.5 Submission / Attempt**

Student participation record for one assessment.

### **5.6 Result Publication**

Teacher/Admin-controlled release of evaluated results to students.

---

## **6\. Functional Requirements**

### **🔹 6.1 Assessment Creation**

* FR-ASM-001: Teacher/Admin shall create assessments.  
* FR-ASM-002: Assessment shall support configurable type.  
* FR-ASM-003: Assessment shall support mapping to batch.  
* FR-ASM-004: Assessment shall support mapping to subject.  
* FR-ASM-005: Assessment shall support optional chapter/topic mapping.  
* FR-ASM-006: Assessment shall support optional linked class occurrence context.  
* FR-ASM-007: Assessment shall support title, instructions, total marks, and duration where relevant.  
* FR-ASM-008: System shall support draft and published states.

### **🔹 6.2 Question & Submission Design**

* FR-ASM-009: Assessment shall support objective questions.  
* FR-ASM-010: Assessment shall support subjective questions.  
* FR-ASM-011: Assignment-type assessment shall support file submission.  
* FR-ASM-012: Assessment shall support question marks allocation.  
* FR-ASM-013: System shall support answer key for objective questions.  
* FR-ASM-014: System shall support teacher review marks for subjective responses.

### **🔹 6.3 Student Attempt Flow**

* FR-ASM-015: Student shall view assigned published assessments.  
* FR-ASM-016: Student shall start assessments within allowed window.  
* FR-ASM-017: Time-bound assessments shall show timer.  
* FR-ASM-018: System shall auto-submit when timer ends if applicable.  
* FR-ASM-019: Assignment submission shall support file upload where configured.  
* FR-ASM-020: System shall prevent duplicate final submission beyond allowed attempts.  
* FR-ASM-021: System shall save attempt/submission state according to configuration.

### **🔹 6.4 Evaluation**

* FR-ASM-022: Objective questions shall support automatic evaluation.  
* FR-ASM-023: Subjective answers shall support teacher evaluation.  
* FR-ASM-024: Teacher shall add awarded marks and optional remarks.  
* FR-ASM-025: Teacher shall evaluate uploaded assignment submissions.  
* FR-ASM-026: System shall maintain evaluation state.  
* FR-ASM-027: System shall support pending evaluation count for teacher dashboard/workspace.

### **🔹 6.5 Result Management**

* FR-ASM-028: System shall calculate total score.  
* FR-ASM-029: System shall calculate percentage where applicable.  
* FR-ASM-030: System shall support result publication.  
* FR-ASM-031: Students shall see only published results.  
* FR-ASM-032: System shall support result summary by assessment and student.  
* FR-ASM-033: System shall support rank or comparative summary if configured.

### **🔹 6.6 Performance Insights**

* FR-ASM-034: System shall provide student performance summary.  
* FR-ASM-035: System shall support weak chapter/topic insight where assessment mapping allows.  
* FR-ASM-036: System shall support recent trend summary for dashboard usage.  
* FR-ASM-037: System shall support batch-level performance summary for teacher/admin.

### **🔹 6.7 Class Workspace Linkage**

* FR-ASM-038: Teacher Class Workspace may open assessment creation with pre-filled batch/subject/class context.  
* FR-ASM-039: Student Class Workspace may show linked active assessments when relevant.  
* FR-ASM-040: Assessment module shall remain the source of truth for all evaluation logic.

---

## **7\. Business Rules**

* Assessment is independent from class schedule, but may be linked contextually to class.  
* Students may access only assessments assigned to their batch/role context.  
* Unpublished assessments are not visible to students.  
* Unpublished results are not visible to students.  
* Objective auto-evaluation and subjective manual evaluation may coexist in one assessment.  
* Assignment submission files are owned logically by Assessment module.  
* Linked class context is optional and should not be mandatory for every assessment.  
* Timer rules apply only to timed assessments.  
* Weak-area analysis depends on correct academic tagging.

---

## **8\. User Flow / Process Flow**

### **8.1 Teacher Create Assessment Flow**

1. Teacher opens Assessment module or opens assessment shortcut from class workspace.  
2. Selects assessment type.  
3. Enters title and instructions.  
4. Selects batch, subject, and optional chapter/topic.  
5. Adds questions or submission requirements.  
6. Sets timing and publication state.  
7. Saves draft or publishes.

### **8.2 Student Attempt Flow**

1. Student opens assigned assessments.  
2. Selects one active assessment.  
3. Reads instructions.  
4. Starts attempt.  
5. Answers questions or uploads assignment file.  
6. Submits.  
7. System stores submission.

### **8.3 Teacher Evaluation Flow**

1. Teacher opens pending evaluations.  
2. Reviews submissions.  
3. Objective items are pre-evaluated where possible.  
4. Teacher evaluates subjective items.  
5. Saves final marks.  
6. Publishes result.

### **8.4 Student Result Flow**

1. Student opens results.  
2. Sees published result only.  
3. Reviews marks and summary.  
4. May see weak chapter/topic insight if available.

---

## **9\. Data Model**

### **📘 Assessment**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| title | String | Assessment title |
| assessment\_type | Enum | Type |
| batch\_id | UUID | Batch |
| subject\_id | UUID | Subject |
| chapter\_id | UUID nullable | Chapter |
| linked\_schedule\_entry\_id | UUID nullable | Linked class |
| total\_marks | Decimal | Total |
| duration\_minutes | Integer nullable | Duration |
| status | Enum | DRAFT / PUBLISHED / CLOSED |
| created\_by | UUID | Creator |

### **📘 AssessmentQuestion**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| assessment\_id | UUID | Assessment |
| question\_type | Enum | OBJECTIVE / SUBJECTIVE / FILE\_SUBMISSION |
| question\_text | Text | Question |
| marks | Decimal | Marks |
| answer\_key | JSON nullable | Answer key |

### **📘 AssessmentAttempt**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| assessment\_id | UUID | Assessment |
| student\_id | UUID | Student |
| started\_at | Timestamp | Start |
| submitted\_at | Timestamp nullable | Submit time |
| status | Enum | IN\_PROGRESS / SUBMITTED / EVALUATED |
| total\_score | Decimal nullable | Score |

### **📘 AssessmentAnswer**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| attempt\_id | UUID | Attempt |
| question\_id | UUID | Question |
| answer\_data | JSON nullable | Answer |
| file\_asset\_id | UUID nullable | File upload |
| awarded\_marks | Decimal nullable | Marks |

### **📘 AssessmentResult**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| assessment\_id | UUID | Assessment |
| student\_id | UUID | Student |
| score | Decimal | Marks |
| percentage | Decimal nullable | Percentage |
| rank | Integer nullable | Rank |
| published | Boolean | Published |

---

## **10\. API Contracts**

### **Create Assessment**

`POST /assessments`

{  
  "title": "Class Test \- Algebra",  
  "assessment\_type": "CLASS\_TEST",  
  "batch\_id": "uuid",  
  "subject\_id": "uuid",  
  "linked\_schedule\_entry\_id": "uuid",  
  "total\_marks": 20  
}

### **Get Student Assessments**

`GET /student/assessments`

### **Start Assessment**

`POST /assessments/{id}/start`

### **Submit Assessment**

`POST /assessments/{id}/submit`

### **Evaluate Attempt**

`POST /assessments/{id}/evaluate/{attempt_id}`

### **Publish Result**

`POST /assessments/{id}/publish-results`

### **Get Student Result**

`GET /student/results/{assessment_id}`

---

## **11\. UI Components**

### **Teacher Assessment Screen**

* assessment list  
* create assessment button  
* batch/subject filters  
* pending evaluation list  
* publish result action

### **Assessment Create/Edit Form**

* assessment type selector  
* batch selector  
* subject selector  
* optional linked class selector  
* total marks  
* duration  
* instructions  
* question builder / submission config

### **Student Assessment Screen**

* upcoming/active/completed tabs  
* assessment cards  
* start action  
* result view action

### **Assessment Attempt Screen**

* question list  
* timer if timed  
* save/submit action  
* upload field for file submission

### **Result Screen**

* score summary  
* percentage  
* rank if enabled  
* remarks if available

---

## **12\. UX Flow**

### **Teacher UX**

Teacher should be able to create assessments either directly from the Assessment module or contextually from class workspace.

### **Student UX**

Student should experience a clean, predictable assessment flow:

* see assigned assessment  
* enter at the right time  
* submit safely  
* view result when published

---

## **13\. Validation Rules**

* batch required  
* subject required  
* title required  
* total marks must be positive  
* timed assessment must have valid duration  
* student must belong to eligible batch  
* objective answers must match expected structure  
* file submission must reference valid uploaded asset  
* result cannot be student-visible before publication

---

## **14\. Edge Cases**

* timer expires during upload  
* student refreshes browser mid-attempt  
* teacher publishes before finishing all evaluations  
* linked class exists but assessment remains valid after schedule changes  
* assignment file missing after submission  
* duplicate submission attempt  
* ranking tie  
* student batch changes after assignment publication

---

## **15\. Dependencies**

* Teacher Class Workspace  
* Student Class Workspace  
* Dashboard System  
* Notification Management  
* Shared File Asset & Document Management  
* Authentication & Authorization  
* Class, Batch & Subject Context Management

---

## **16\. Non-Functional Requirements**

* timed attempts must remain stable  
* submission flow should tolerate weak internet as much as possible  
* evaluation data must be consistent  
* student result access must be permission-safe  
* batch-size scale should be handled for normal coaching operations

---

## **17\. Assumptions**

* assessments are batch and subject driven  
* some assessments may be linked to class context  
* written and file-based submissions are needed in coaching workflows  
* many students will attempt assessments from mobile devices

---

## **18\. Open Questions**

* how many attempts allowed per assessment type?  
* should students view solutions immediately or only after result publication?  
* should class-linked assessments automatically appear inside class detail?

# **🧩 MODULE: ATTENDANCE MANAGEMENT SYSTEM**

---

# **1\. Purpose**

Manage student attendance through manual roll call and machine integration, provide batch-wise operational control for teachers/admins, and enable students to track their attendance with summarized insights.

---

# **2\. Scope / Responsibilities**

## **✅ In Scope**

* Manual attendance (single \+ bulk)  
* Machine-based attendance ingestion  
* Attendance override system  
* Batch-wise roll call UI  
* Attendance summary (Day \+ Month)  
* Attendance logs (Admin only)  
* Student self-view attendance  
* Guardian notification (manual trigger)

## **❌ Out of Scope**

* Payroll calculation  
* Fine generation for absence  
* Device configuration

---

# **3\. Actors**

* Admin  
* Teacher  
* Student  
* System (machine ingestion, calculations, notifications)

---

# **4\. Functional Requirements (FR)**

## **🔹 Roll Call**

* FR-ATT-001: System shall allow Admin/Teacher to select batch and date  
* FR-ATT-002: System shall load student list for selected batch  
* FR-ATT-003: System shall allow marking attendance per student (Present/Absent/Late/Excused)  
* FR-ATT-004: System shall support bulk attendance actions  
* FR-ATT-005: System shall allow quick toggle (Present/Absent) per student  
* FR-ATT-006: System shall allow saving attendance in one action  
* FR-ATT-007: System shall allow sending SMS (single/bulk)  
* FR-ATT-008: System shall show machine-derived attendance if available  
* FR-ATT-009: System shall allow manual override of machine attendance  
* FR-ATT-010: System shall log override action

## **🔹 Summary**

* FR-ATT-011: System shall provide Day view summary  
* FR-ATT-012: System shall provide Month view summary  
* FR-ATT-013: System shall allow filtering by batch  
* FR-ATT-014: System shall allow filtering by custom date range  
* FR-ATT-015: System shall calculate attendance percentage dynamically

## **🔹 Logs (Admin Only)**

* FR-ATT-016: System shall allow viewing attendance logs  
* FR-ATT-017: System shall support filtering logs by batch and date range  
* FR-ATT-018: System shall show source (Manual/Machine)  
* FR-ATT-019: System shall show override history

## **🔹 Student Self View**

* FR-ATT-020: Student shall view own attendance summary  
* FR-ATT-021: Student shall filter by batch and date range  
* FR-ATT-022: System shall show monthly summary by default  
* FR-ATT-023: System shall show attendance list in descending order

---

# **5\. Business Logic / Rules**

* One attendance record per student per batch per day  
* Machine attendance is default if exists  
* Manual attendance overrides machine record  
* Attendance cannot be marked for future dates  
* Student must belong to batch to mark attendance  
* Attendance % \= Present / Total Class Days  
* SMS sending is optional and manual  
* Teacher can access only assigned batches  
* Admin can access all batches

---

# **6\. User Flow / Process Flow**

## **🔹 Roll Call Flow**

1. User opens Attendance → Roll Call  
2. Selects:  
   * Batch  
   * Date  
3. System loads student list  
4. User:  
   * toggles Present/Absent  
   * uses bulk actions  
5. Clicks Save  
6. System:  
   * validates data  
   * saves records  
   * logs overrides (if any)  
7. User optionally sends SMS

---

## **🔹 Summary Flow**

1. User opens Summary tab  
2. Selects:  
   * Day or Month view  
   * Batch  
   * Date range  
3. System loads data  
4. Displays:  
   * Day → list view  
   * Month → matrix grid

---

## **🔹 Student Flow**

1. Student opens My Attendance  
2. System loads current month data  
3. Student filters if needed  
4. Student views:  
   * summary cards  
   * detailed list

---

# **7\. Data Model (EXTENDED)**

## **7.1 Entities**

* AttendanceDailyRecord  
* MachineAttendanceData  
* AttendanceSummary  
* AttendanceLog  
* NotificationLog

---

## **7.2 Schema**

### **📘 AttendanceDailyRecord**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| student\_id | UUID | Student |
| batch\_id | UUID | Batch |
| date | Date | Attendance date |
| status | Enum | PRESENT/ABSENT/LATE/EXCUSED |
| source | Enum | MACHINE/MANUAL |
| first\_entry | Timestamp | Machine first entry |
| last\_exit | Timestamp | Machine last exit |
| overridden | Boolean | override flag |
| overridden\_by | UUID | user id |
| note | String | remark |

---

### **📘 MachineAttendanceData**

* id  
* machine\_user\_id  
* timestamp  
* raw\_payload  
* processed\_flag

---

### **📘 AttendanceSummary**

* student\_id  
* batch\_id  
* month  
* year  
* total\_classes  
* present  
* absent  
* late  
* percentage

---

### **📘 AttendanceLog**

* id  
* student\_id  
* batch\_id  
* date  
* source  
* updated\_by  
* updated\_at  
* action\_type (CREATE/UPDATE/OVERRIDE)

---

# **8\. API Contracts**

## **Roll Call**

POST /attendance/student/bulk/{batchId}/{date}

Request:

{  
  "records": \[  
    {  
      "student\_id": "uuid",  
      "status": "PRESENT",  
      "note": "optional"  
    }  
  \]  
}

---

GET /attendance/student?batchId=\&date=

---

POST /attendance/student/send-sms

---

## **Summary**

GET /attendance/batch/{batchId}/day

GET /attendance/batch/{batchId}/month

---

## **Logs (Admin)**

GET /attendance/logs

---

## **Student**

GET /attendance/student/my-summary

GET /attendance/student/my-records

---

# **9\. UI Components**

---

## **🧩 Screen: Roll Call**

Components:

* Batch Selector (dropdown)  
* Date Picker  
* Load Button

### **Table**

Columns:

* Checkbox  
* Roll No  
* Student Name  
* Status (Toggle switch)  
* Source (Machine/Manual)  
* First Entry  
* Last Exit  
* Note (editable)  
* Action (SMS)

### **Top Actions**

* Mark All Present  
* Mark All Absent  
* Bulk Action (Selected)  
* Save Button  
* Send SMS

---

## **🧩 Screen: Summary**

### **Tabs:**

* Day  
* Month

---

### **Day View**

Components:

* Batch Selector  
* Date Range Picker

Table Columns:

* Roll No  
* Student Name  
* Status  
* Source  
* First Entry  
* Last Exit  
* Note

---

### **Month View (Matrix Grid)**

Rows:

* Students

Columns:

* 1–31 days

Cell values:

* P / A / L / E

Left Sticky Columns:

* Roll No  
* Student Name  
* Present Count  
* Absent Count  
* %

---

## **🧩 Screen: Logs (Admin only)**

Components:

* Batch Selector  
* Date Range Filter

Table:

* Date  
* Student Name  
* Status  
* Source  
* Override  
* Updated By  
* Updated At

---

## 

## **🧩 Screen: Student – My Attendance**

### **Summary Cards:**

* Attendance %  
* Present  
* Absent  
* Late  
* Total Classes

### **Filters:**

* Batch Selector  
* Custom Date Range

### **Table:**

Columns:

* Date  
* Day  
* Status  
* Source  
* Note

(Default current month, sorted DESC)

---

# **10\. UX Flow**

## **Roll Call UX**

1. Select batch \+ date  
2. Click Load  
3. List appears  
4. Toggle status quickly  
5. Use bulk if needed  
6. Click Save  
7. Optional SMS

---

## **Month View UX**

1. Select Month  
2. System loads matrix  
3. Scroll horizontally for dates  
4. View patterns visually

---

## **Student UX**

1. Opens page  
2. Sees summary  
3. Scrolls list  
4. Applies filter if needed

---

# **11\. Validation Rules**

* batchId required  
* date required  
* cannot mark future date  
* student must belong to batch  
* status must be valid enum  
* duplicate record must update existing  
* SMS only if phone exists

---

# **12\. Edge Cases**

* Machine \+ manual conflict  
* Student removed from batch mid-month  
* Partial bulk save  
* Network failure during save  
* Duplicate submission  
* Missing machine data  
* SMS failure  
* Late entry crossing date boundary

---

# **13\. Dependencies**

* User Management  
* Class & Batch Module  
* Notification System  
* Machine Integration

---

# **14\. Non-Functional Requirements**

* Load time \< 2 seconds  
* Must support 1000+ students per batch  
* Mobile responsive (teacher usage)  
* High consistency (no duplicate attendance)  
* Audit logging mandatory

---

# **15\. Assumptions**

* Timezone \= Bangladesh (GMT+6)  
* Teacher uses mobile device often  
* Machine integration is available  
* SMS provider configured

---

# **16\. Open Questions**

* Do we allow attendance lock after certain days?  
* Do we allow re-edit after a summary is generated?  
* Should SMS be auto-triggered later?

# **🧩 MODULE: PAYMENT & BILLING MANAGEMENT SYSTEM**

## **1\. User Story**

### **Student / Guardian User Story**

As a student or guardian,  
I want to view dues, pay fees online or manually, and receive receipts/reminders,  
So that I can keep payments updated without confusion.

### **Accountant User Story**

As an accountant,  
I want to collect fees, verify payments, manage dues, and generate receipts/reports,  
So that coaching finances remain accurate and efficient.

### **Admin User Story**

As an admin,  
I want to configure fee structures, discounts, fines, and billing cycles,  
So that all student billing runs in a disciplined and scalable manner.

---

# **2\. Purpose**

Manage student billing lifecycle including fee setup, invoice generation, due tracking, online/manual payment collection, reminders, receipts, waivers, fines, and payment history.

---

# **3\. Scope**

## **✅ In Scope**

* Admission fee billing  
* Monthly tuition billing  
* One-time charges  
* Installments  
* Fine / late fee  
* Discounts / waivers  
* Online payments  
* Manual cash/MFS/bank payments  
* Receipt generation  
* Due reminders  
* Payment history  
* Student billing dashboard

## **❌ Out of Scope (Phase 1\)**

* Full ERP inventory billing  
* Complex taxation engine  
* Multi-branch revenue consolidation (can add later)

---

# **4\. Actors**

* Admin  
* Accountant  
* Student  
* Guardian  
* System  
* Payment Gateway

---

# **5\. Functional Requirements**

## **🔹 5.1 Fee Structure Setup**

* FR-PAY-001: Configure admission fee.  
* FR-PAY-002: Configure monthly fee by class/batch.  
* FR-PAY-003: Configure one-time charges.  
* FR-PAY-004: Configure installment plans.  
* FR-PAY-005: Configure due dates.  
* FR-PAY-006: Configure late fine rules.

## **🔹 5.2 Billing Engine**

* FR-PAY-007: Generate monthly invoices automatically.  
* FR-PAY-008: Generate manual invoices when needed.  
* FR-PAY-009: Apply discounts/waivers.  
* FR-PAY-010: Recalculate due balance.

## **🔹 5.3 Payment Collection**

* FR-PAY-011: Accept online payment.  
* FR-PAY-012: Record cash payment.  
* FR-PAY-013: Record MFS payment.  
* FR-PAY-014: Record bank transfer.  
* FR-PAY-015: Verify pending manual payments.

## **🔹 5.4 Receipts**

* FR-PAY-016: Generate receipt number.  
* FR-PAY-017: Printable/downloadable receipt.  
* FR-PAY-018: Send receipt notification.

## **🔹 5.5 Reminder & Due Management**

* FR-PAY-019: Send pre-due reminder.  
* FR-PAY-020: Send overdue reminder.  
* FR-PAY-021: Apply late fine automatically if enabled.

## **🔹 5.6 Student Portal**

* FR-PAY-022: Student sees outstanding dues.  
* FR-PAY-023: Student sees payment history.  
* FR-PAY-024: Student downloads receipts.

---

# **6\. Business Rules**

* One student may have multiple invoices.  
* Partial payment allowed if configured.  
* Overpayment becomes advance balance or refundable.  
* Fine starts after due date.  
* Deleted payments require approval.  
* Receipt numbers must be unique.  
* Bangladesh currency \= BDT default.

---

# **7\. User Flow**

## **🔹 Monthly Billing Flow**

1. System runs billing cycle  
2. Creates invoices  
3. Sends reminders  
4. Student pays  
5. Receipt generated

## **🔹 Manual Collection Flow**

1. Accountant opens student profile  
2. Selects unpaid invoice  
3. Records payment  
4. Receipt printed

---

# **8\. Data Model**

## **📘 FeePlan**

| Field | Type |
| ----- | ----- |
| id | UUID |
| name | String |
| class\_id | UUID |
| batch\_id | UUID |
| amount | Decimal |
| cycle | Enum |

## **📘 Invoice**

| Field | Type |
| ----- | ----- |
| id | UUID |
| student\_id | UUID |
| invoice\_no | String |
| amount | Decimal |
| fine | Decimal |
| discount | Decimal |
| due\_date | Date |
| status | Enum |

## **📘 Payment**

| Field | Type |
| ----- | ----- |
| id | UUID |
| invoice\_id | UUID |
| amount | Decimal |
| method | Enum |
| txn\_ref | String |
| paid\_at | Timestamp |

---

# **9\. API Contracts**

## **Create Fee Plan**

`POST /billing/fee-plans`

## **Generate Invoice**

`POST /billing/run-monthly`

## **Collect Payment**

`POST /billing/payments`

{  
  "invoice\_id": "uuid",  
  "amount": 3000,  
  "method": "MFS"  
}

## **Student Dues**

`GET /student/billing`

---

# **10\. UI Components**

## **Accountant**

* Invoice list  
* Collect payment modal  
* Receipt print  
* Due tracker

## **Student**

* Outstanding dues card  
* Pay now button  
* History table

## **Admin**

* Fee setup  
* Fine rules  
* Discount approvals

---

# **11\. Validation Rules**

* amount \> 0  
* payment \<= due unless overpay allowed  
* due date required  
* valid payment method  
* invoice must exist

---

# **12\. Edge Cases**

* Duplicate gateway callback  
* Partial payment mismatch  
* Wrong manual amount entered  
* Refund request  
* Student transferred batch mid-month  
* Reminder sent after payment

---

# **13\. Dependencies**

* Admission Module  
* User Management  
* Notification Module  
* Accounting Module

---

# **14\. Non-Functional Requirements**

* Payment posting atomic  
* Receipt generation \< 2 sec  
* Audit logs mandatory  
* Mobile payment UX friendly

---

# **15\. Assumptions**

* Manual payments common in Bangladesh.  
* bKash/Nagad/Rocket future integrations possible.

---

# **16\. Open Questions**

* Advance monthly payment allowed?  
* Auto suspend student on unpaid dues?  
* Refund workflow phase 2?

---

# 

# 

# 

# 

# **🧩 MODULE: PAYROLL MANAGEMENT SYSTEM**

## **1\. User Story**

### **Teacher / Employee User Story**

As a teacher or employee,  
I want accurate salary calculation and payslips,  
So that I receive fair and transparent payments.

### **Accountant User Story**

As an accountant,  
I want to process payroll quickly using attendance and configured salary rules,  
So that monthly salary operations become efficient.

---

# **2\. Purpose**

Manage teacher and employee compensation including salary setup, attendance linkage, class-based payments, deductions, bonuses, payslips, and payroll disbursement history.

---

# **3\. Scope**

## **✅ In Scope**

* Fixed monthly salary  
* Per class payment  
* Hybrid salary model  
* Attendance deductions  
* Overtime  
* Bonus / allowance  
* Payroll run  
* Payslip generation  
* Payment history

## **❌ Out of Scope**

* Income tax filing automation  
* Bank bulk transfer integration phase 1

---

# **4\. Actors**

* Accountant  
* Admin  
* Teacher  
* Employee  
* System

---

# **5\. Functional Requirements**

## **🔹 5.1 Salary Setup**

* FR-PRO-001: Configure salary type.  
* FR-PRO-002: Configure monthly amount.  
* FR-PRO-003: Configure per class rate.  
* FR-PRO-004: Configure deductions.

## **🔹 5.2 Payroll Calculation**

* FR-PRO-005: Pull attendance summary.  
* FR-PRO-006: Pull class count if class-rate model.  
* FR-PRO-007: Add bonus/overtime.  
* FR-PRO-008: Compute net payable.

## **🔹 5.3 Payroll Processing**

* FR-PRO-009: Run payroll monthly.  
* FR-PRO-010: Review before finalization.  
* FR-PRO-011: Lock finalized payroll.  
* FR-PRO-012: Generate payslips.

## **🔹 5.4 Payment Recording**

* FR-PRO-013: Mark salary paid.  
* FR-PRO-014: Record method/ref no.

---

# **6\. Business Rules**

* One payroll record per user per month.  
* Finalized payroll edits require reversal flow.  
* Absent deduction rule configurable.  
* Negative payable not allowed.  
* Teacher class count from completed classes if configured.

---

# **7\. User Flow**

1. Accountant opens payroll month  
2. System calculates draft payroll  
3. Accountant reviews adjustments  
4. Finalize payroll  
5. Payslips generated  
6. Mark paid

---

# **8\. Data Model**

## **📘 CompensationPlan**

| Field | Type |
| ----- | ----- |
| id | UUID |
| user\_id | UUID |
| salary\_type | Enum |
| monthly\_amount | Decimal |
| per\_class\_rate | Decimal |

## **📘 PayrollRun**

| Field | Type |
| ----- | ----- |
| id | UUID |
| month | Integer |
| year | Integer |
| status | Enum |

## **📘 PayrollItem**

| Field | Type |
| ----- | ----- |
| id | UUID |
| payroll\_run\_id | UUID |
| user\_id | UUID |
| gross | Decimal |
| deduction | Decimal |
| bonus | Decimal |
| net | Decimal |

---

# **9\. API Contracts**

## **Run Payroll**

`POST /payroll/run`

## **Finalize Payroll**

`POST /payroll/{id}/finalize`

## **Mark Paid**

`POST /payroll/items/{id}/pay`

---

# **10\. UI Components**

* Payroll month selector  
* Draft payroll grid  
* Adjustment modal  
* Finalize button  
* Payslip download

---

# **11\. Validation Rules**

* month/year required  
* duplicate payroll month blocked  
* net \>= 0  
* finalized requires confirmation

---

# **12\. Edge Cases**

* Attendance corrected after finalize  
* Teacher joins mid-month  
* Employee resigned mid-month  
* Duplicate payment marking

---

# **13\. Dependencies**

* Attendance Module  
* Teacher Class Module  
* Accounting Module  
* User Management

---

# **14\. Non-Functional Requirements**

* Payroll for 1000 staff \< acceptable batch runtime  
* Accurate decimal math  
* Full audit trail

---

# **15\. Assumptions**

* Some teachers paid per class in coaching model.  
* Mixed salary structures common.

---

# **16\. Open Questions**

* Advance salary loan deduction later?  
* Tax slip later?

---

# 

# 

# **🧩 MODULE: ACCOUNTING & FINANCIAL TRACKING SYSTEM**

## **1\. User Story**

### **Accountant User Story**

As an accountant,  
I want to track income, expenses, balances, and summaries,  
So that I maintain clear financial control of the coaching center.

### **Owner / Admin User Story**

As an owner,  
I want monthly profitability visibility,  
So that I can make business decisions confidently.

---

# **2\. Purpose**

Provide financial visibility by tracking revenue, expenses, balances, payment inflow, payroll outflow, and management reports.

---

# **3\. Scope**

## **✅ In Scope**

* Income ledger  
* Expense ledger  
* Fee revenue sync  
* Payroll expense sync  
* Cash/bank balances  
* Monthly summary  
* Yearly summary  
* Profit/loss overview  
* Basic reporting

## **❌ Out of Scope**

* Full double-entry ERP accounting phase 1  
* Tax/VAT filing automation

---

# **4\. Actors**

* Accountant  
* Admin  
* Owner  
* System

---

# **5\. Functional Requirements**

## **🔹 5.1 Revenue Tracking**

* FR-ACC-001: Sync student payments as income.  
* FR-ACC-002: Support manual income entries.

## **🔹 5.2 Expense Tracking**

* FR-ACC-003: Record rent.  
* FR-ACC-004: Record utilities.  
* FR-ACC-005: Record payroll outflow.  
* FR-ACC-006: Record misc expenses.

## **🔹 5.3 Reports**

* FR-ACC-007: Monthly income summary.  
* FR-ACC-008: Monthly expense summary.  
* FR-ACC-009: Profit/loss summary.  
* FR-ACC-010: Yearly trend report.

## **🔹 5.4 Balances**

* FR-ACC-011: Track cash balance.  
* FR-ACC-012: Track bank balance.

---

# **6\. Business Rules**

* Every ledger entry timestamped.  
* Linked module entries should reference source transaction.  
* Soft delete preferred with reversal records.  
* Revenue recognized on successful payment posting.

---

# **7\. User Flow**

1. Fees collected → income ledger  
2. Salary paid → expense ledger  
3. Accountant adds other expenses  
4. Owner views reports

---

# **8\. Data Model**

## **📘 LedgerEntry**

| Field | Type |
| ----- | ----- |
| id | UUID |
| type | Enum |
| category | String |
| amount | Decimal |
| source\_module | String |
| ref\_id | UUID |
| entry\_date | Date |

## **📘 AccountBalance**

| Field | Type |
| ----- | ----- |
| id | UUID |
| account\_name | String |
| balance | Decimal |

---

# **9\. API Contracts**

## **Add Expense**

`POST /finance/expenses`

## **Summary**

`GET /finance/summary?month=4&year=2026`

## **Ledger**

`GET /finance/ledger`

---

# **10\. UI Components**

* KPI cards (income/expense/profit)  
* Ledger table  
* Expense add form  
* Filters  
* Export report

---

# **11\. Validation Rules**

* amount \> 0  
* category required  
* date required

---

# **12\. Edge Cases**

* Duplicate sync from payment module  
* Wrong expense category  
* Backdated entries  
* Deleted linked payment

---

# **13\. Dependencies**

* Payment Module  
* Payroll Module  
* User Management

---

# **14\. Non-Functional Requirements**

* Reports \< 3 sec for common periods  
* Accurate balances  
* Exportable CSV/PDF later

---

# **15\. Assumptions**

* Coaching center needs management-level reporting first.  
* Full enterprise accounting can come later.

---

# **16\. Open Questions**

* Need branch-wise accounting later?  
* Need approval workflow for expenses?  
* Need budget vs actual reporting later?

# **🧩 MODULE: NOTIFICATION MANAGEMENT SYSTEM**

## **1\. User Story**

### **Student User Story**

As a student,  
I want to receive relevant alerts about classes, exams, results, fees, and notices,  
So that I do not miss important academic and administrative updates.

### **Teacher User Story**

As a teacher,  
I want students and guardians to receive timely updates related to classes, assessments, and notices,  
So that communication remains smooth and organized.

### **Admin User Story**

As an admin,  
I want the system to send targeted notifications automatically and manually,  
So that the coaching center can maintain disciplined communication at scale.

### **Guardian User Story**

As a guardian,  
I want to receive important alerts regarding attendance, exams, fees, and notices,  
So that I can monitor and support my child properly.

---

## **2\. Purpose**

Manage all event-driven and manual communication delivery across the LMS including in-app notifications, notice alerts, reminders, announcements, SMS/email hooks, read tracking, and delivery logs.

---

## **3\. Scope**

### **✅ In Scope**

* In-app notifications  
* Notice publication alerts  
* Class schedule notifications  
* Class reminder notifications  
* Live class reminders  
* Assessment/exam reminders  
* Result publication alerts  
* Payment due reminders  
* Overdue payment alerts  
* Admission workflow notifications  
* Attendance alerts  
* Bulk announcements  
* Role/audience based targeting  
* Read/unread tracking  
* Delivery logs  
* Scheduled notifications  
* Template-based messages

### **❌ Out of Scope (Phase 1\)**

* WhatsApp integration  
* AI-generated message optimization  
* Two-way chat inbox  
* full campaign automation

---

## **4\. Actors**

* Admin  
* Teacher  
* Student  
* Guardian  
* Accountant  
* System

---

## **5\. Core Concepts**

### **5.1 Notification**

A short communication item sent to one or more users and linked to an event or content page.

### **5.2 Notification Channel**

Supported channels:

* IN\_APP  
* SMS  
* EMAIL  
* PUSH (future-ready)

### **5.3 Trigger Event**

An action in another module that creates a notification request.

Examples:

* notice published  
* exam scheduled  
* result published  
* payment overdue  
* attendance marked

### **5.4 Audience Resolution**

System determines intended recipients based on:

* role  
* class  
* batch  
* user id  
* guardian mapping  
* teacher assignment

---

## **6\. Functional Requirements**

### **🔹 6.1 Event Notifications**

* FR-NOT-001: System shall create notification on configured business events.  
* FR-NOT-002: System shall support notifications from Notice Board module.  
* FR-NOT-003: System shall support notifications from Class Scheduling module.  
* FR-NOT-004: System shall support notifications from Live Class module.  
* FR-NOT-005: System shall support notifications from Assessment module.  
* FR-NOT-006: System shall support notifications from Payment module.  
* FR-NOT-007: System shall support notifications from Admission module.  
* FR-NOT-008: System shall support notifications from Attendance module.

### **🔹 6.2 Manual Notifications**

* FR-NOT-009: Admin shall send manual announcements.  
* FR-NOT-010: Admin shall target users by role/class/batch.  
* FR-NOT-011: Teacher may send limited batch-specific announcements if permitted.

### **🔹 6.3 Delivery Channels**

* FR-NOT-012: System shall support in-app notification delivery.  
* FR-NOT-013: System shall support SMS delivery through provider integration.  
* FR-NOT-014: System shall support email delivery.  
* FR-NOT-015: System shall support channel selection per notification type.

### **🔹 6.4 Notification Center**

* FR-NOT-016: User shall view own notifications.  
* FR-NOT-017: System shall show read/unread status.  
* FR-NOT-018: User shall mark notification as read.  
* FR-NOT-019: User shall open linked module/content from notification.

### **🔹 6.5 Scheduling & Retry**

* FR-NOT-020: System shall schedule reminders before configured event time.  
* FR-NOT-021: Failed delivery attempts shall be logged.  
* FR-NOT-022: Retry logic shall be supported for retryable failures.

### **🔹 6.6 Templates & Logs**

* FR-NOT-023: Admin shall manage notification templates.  
* FR-NOT-024: System shall store delivery log per recipient.  
* FR-NOT-025: System shall store failure reason when delivery fails.

---

## **7\. Business Rules**

* Every user sees only their own notifications.  
* Notice publication may create one or more recipient notifications.  
* In-app notification is default mandatory channel.  
* SMS/email are configurable by event type.  
* Duplicate notifications for same event/user should be prevented where needed.  
* Guardian notifications use mapped guardian contact.  
* Read state is per recipient.  
* Expired notices may remain in history but not as active alerts.

---

## **8\. Data Model**

### **📘 NotificationEvent**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| source\_module | Enum | NOTICE/CLASS/ASSESSMENT/PAYMENT/etc |
| source\_entity\_id | UUID | Original entity id |
| event\_type | String | RESULT\_PUBLISHED, NOTICE\_PUBLISHED |
| triggered\_at | Timestamp | Event time |

### **📘 Notification**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| event\_id | UUID | Related event |
| title | String | Short title |
| body | Text | Message body |
| channel | Enum | IN\_APP/SMS/EMAIL |
| deep\_link | String | Target page |
| scheduled\_at | Timestamp | Scheduled send time |
| sent\_at | Timestamp | Sent time |
| status | Enum | PENDING/SENT/FAILED |

### **📘 NotificationRecipient**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| notification\_id | UUID | Notification ref |
| user\_id | UUID | Recipient |
| guardian\_id | UUID | Optional guardian ref |
| delivery\_status | Enum | PENDING/SENT/FAILED/READ |
| read\_at | Timestamp | Read time |
| failure\_reason | String | Failure cause |

### **📘 NotificationTemplate**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| code | String | Template key |
| channel | Enum | IN\_APP/SMS/EMAIL |
| title\_template | String | Template title |
| body\_template | Text | Template body |
| is\_active | Boolean | Active flag |

---

## **9\. API Contracts**

### **Get My Notifications**

`GET /notifications/me`

### **Mark Read**

`POST /notifications/{id}/read`

### **Send Manual Announcement**

`POST /notifications/manual`

{  
  "title": "Tomorrow class rescheduled",  
  "message": "Class 10 Math batch will start at 5 PM",  
  "target": {  
    "roles": \["STUDENT"\],  
    "batch\_ids": \["uuid"\]  
  },  
  "channels": \["IN\_APP", "SMS"\]  
}

### **Get Delivery Logs**

`GET /notifications/logs`

---

## **10\. UI Components**

### **User Notification Center**

* notification bell  
* unread badge  
* notification list  
* mark read action  
* deep link open

### **Admin Notification Console**

* send announcement form  
* target selector  
* channel selector  
* template selector  
* delivery log table

---

## **11\. Validation Rules**

* title required  
* at least one target required  
* at least one channel required  
* SMS only if valid phone exists  
* email only if valid email exists  
* scheduled time must be valid future datetime

---

## **12\. Edge Cases**

* user has no phone/email  
* SMS provider failure  
* duplicate publish event  
* notice audience changed after publish  
* user deactivated before send  
* event triggered but source entity deleted  
* reminder scheduled after event already passed

---

## **13\. Dependencies**

* Notice Board Module  
* Admission Module  
* Attendance Module  
* Class Scheduling Module  
* Live Class Module  
* Assessment Module  
* Payment Module  
* Authentication Module

---

## **14\. Non-Functional Requirements**

* in-app notification creation near real-time  
* delivery logs must be audit-ready  
* bulk announcement should support large batch delivery  
* retry-safe event processing required

---

## **15\. Assumptions**

* in-app notification is mandatory first channel  
* SMS is important in Bangladesh context  
* guardians are primary contacts for many students

---

## **16\. Open Questions**

* Should teachers send direct batch announcements?  
* Should guardians have separate notification inbox?  
* Should notice acknowledgements be tracked?

---

# **🧩 MODULE: NOTICE BOARD MANAGEMENT SYSTEM**

## **1\. User Story**

### **Student User Story**

As a student,  
I want to view official notices relevant to my class, batch, exams, holidays, and academic activities,  
So that I stay updated and do not miss important announcements.

### **Parent / Guardian User Story**

As a guardian,  
I want to receive and read important coaching notices related to my child,  
So that I can support attendance, payments, exams, and schedules properly.

### **Teacher User Story**

As a teacher,  
I want to view institutional notices and publish teacher-authorized batch notices when permitted,  
So that academic communication remains organized.

### **Admin User Story**

As an admin,  
I want to create, publish, schedule, target, and archive notices centrally,  
So that the coaching center can manage official communication professionally.

---

# **2\. Purpose**

Provide a centralized digital notice board for official announcements, academic updates, administrative communication, schedules, holidays, results, and batch-specific notices with audience targeting, attachments, publishing workflow, and archive history.

---

# **3\. Scope**

## **✅ In Scope**

* Notice creation  
* Draft / publish / archive lifecycle  
* Audience targeting  
* Role/class/batch filtering  
* Priority / pinned notices  
* Notice categories  
* Attachments (PDF/image/document)  
* Scheduled publish  
* Expiry date  
* Notice listing page  
* Notice detail page  
* Search & filtering  
* Notice archive/history  
* Optional acknowledgement tracking  
* Trigger notification alerts

## **❌ Out of Scope (Phase 1\)**

* Comment threads under notices  
* Two-way discussion board  
* AI notice writing assistant  
* Multilingual auto-translation  
* Public website notice sync (future possible)

---

# **4\. Actors**

* Admin  
* Teacher (limited publish if allowed)  
* Student  
* Parent / Guardian  
* Accountant (read relevant notices)  
* System

---

# **5\. Core Concepts**

## **5.1 Notice**

An official long-form announcement with title, content, target audience, active dates, and optional attachments.

## **5.2 Audience Scope**

Notice may target:

* All users  
* Students only  
* Teachers only  
* Guardians only  
* Accountant only  
* Specific class  
* Specific batch  
* Specific role combinations

## **5.3 Priority Levels**

* NORMAL  
* IMPORTANT  
* URGENT

## **5.4 Pinning**

Pinned notices remain top of list until unpinned or expired.

## **5.5 Lifecycle Status**

* DRAFT  
* SCHEDULED  
* PUBLISHED  
* EXPIRED  
* ARCHIVED

---

# **6\. Functional Requirements**

---

## **🔹 6.1 Notice Creation**

* FR-NB-001: Admin shall create notice drafts.  
* FR-NB-002: Teacher may create notices if permission enabled.  
* FR-NB-003: Notice shall support rich text content.  
* FR-NB-004: Notice shall support attachments.  
* FR-NB-005: Notice shall support category selection.  
* FR-NB-006: Notice shall support priority level.

---

## **🔹 6.2 Audience Targeting**

* FR-NB-007: Notice shall target by role.  
* FR-NB-008: Notice shall target by class.  
* FR-NB-009: Notice shall target by batch.  
* FR-NB-010: Notice shall support all-user broadcast.  
* FR-NB-011: System shall resolve guardian recipients from student mapping.

---

## **🔹 6.3 Publishing Workflow**

* FR-NB-012: Draft notices may be saved.  
* FR-NB-013: Notice may be published immediately.  
* FR-NB-014: Notice may be scheduled for future publish.  
* FR-NB-015: Notice may have expiry date/time.  
* FR-NB-016: Expired notices shall no longer appear in active lists.  
* FR-NB-017: Notice may be archived manually.

---

## **🔹 6.4 Visibility & Display**

* FR-NB-018: Users shall only view notices intended for them.  
* FR-NB-019: Pinned notices shall appear first.  
* FR-NB-020: Urgent notices shall be visually highlighted.  
* FR-NB-021: Notice detail page shall show title, body, date, attachment.

---

## **🔹 6.5 Search & Filter**

* FR-NB-022: Users may filter by category.  
* FR-NB-023: Users may search by title/content.  
* FR-NB-024: Users may filter by date range.  
* FR-NB-025: Admin may filter by creator/status/audience.

---

## **🔹 6.6 Acknowledgement (Optional)**

* FR-NB-026: Admin may require acknowledgement.  
* FR-NB-027: User may mark “Read & Acknowledged”.  
* FR-NB-028: Admin may view pending acknowledgements.

---

## **🔹 6.7 Notification Integration**

* FR-NB-029: Publishing a notice may trigger notifications.  
* FR-NB-030: System shall deep link notification to notice detail page.

---

# **7\. Business Rules**

* Only authorized publishers can create notices.  
* Draft notices are invisible to end users.  
* Published notices visible only to intended audience.  
* Expired notices removed from active board automatically.  
* Pinned notices sorted above normal notices.  
* Urgent notices may bypass normal ordering.  
* Attachment file type restrictions configurable.  
* Notice edit after publish should maintain version history if enabled.  
* Guardian notices derive from linked student relationships.

---

# **8\. User Flow / Process Flow**

---

## **🔹 Admin Publish Flow**

1. Admin opens Notice Board  
2. Clicks Create Notice  
3. Adds title/content  
4. Uploads attachment  
5. Selects audience  
6. Sets priority / schedule  
7. Clicks Publish  
8. System shows active notice  
9. Notification manager sends alerts

---

## **🔹 Student View Flow**

1. Student logs in  
2. Opens Notice Board  
3. Sees active notices  
4. Opens detail  
5. Downloads attachment if available

---

## **🔹 Scheduled Notice Flow**

1. Admin saves scheduled notice  
2. System waits until publish time  
3. Status becomes PUBLISHED  
4. Notifications triggered

---

# **9\. Data Model**

---

## **📘 Notice**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| title | String | Notice title |
| slug | String | URL key |
| content | Text | Notice body |
| category | Enum | GENERAL/EXAM/HOLIDAY/FEE/etc |
| priority | Enum | NORMAL/IMPORTANT/URGENT |
| status | Enum | DRAFT/PUBLISHED/etc |
| is\_pinned | Boolean | Top notice |
| publish\_at | Timestamp | Publish time |
| expire\_at | Timestamp | Expiry time |
| created\_by | UUID | Creator |
| updated\_by | UUID | Last editor |
| created\_at | Timestamp | Created time |

---

## **📘 NoticeAudience**

| Field | Type |
| ----- | ----- |
| id | UUID |
| notice\_id | UUID |
| role | Enum |
| class\_id | UUID nullable |
| batch\_id | UUID nullable |

---

## **📘 NoticeAttachment**

| Field | Type |
| ----- | ----- |
| id | UUID |
| notice\_id | UUID |
| file\_name | String |
| file\_url | String |
| file\_type | String |

---

## **📘 NoticeReadLog**

| Field | Type |
| ----- | ----- |
| id | UUID |
| notice\_id | UUID |
| user\_id | UUID |
| viewed\_at | Timestamp |
| acknowledged\_at | Timestamp nullable |

---

# **10\. API Contracts**

---

## **Create Notice**

`POST /notices`

{  
  "title": "Monthly Exam Schedule",  
  "content": "Exam starts from 5 May.",  
  "category": "EXAM",  
  "priority": "IMPORTANT",  
  "is\_pinned": true  
}

---

## **Publish Notice**

`POST /notices/{id}/publish`

---

## **Schedule Notice**

`POST /notices/{id}/schedule`

{  
  "publish\_at": "2026-04-25T09:00:00"  
}

---

## **My Notices**

`GET /notices/me`

---

## **Notice Detail**

`GET /notices/{id}`

---

## **Mark Read**

`POST /notices/{id}/read`

---

## **Acknowledge Notice**

`POST /notices/{id}/acknowledge`

---

# **11\. UI Components**

---

## **🧩 Admin Notice Console**

* Notice list table  
* Create button  
* Draft / Published tabs  
* Filters  
* Audience selector  
* Rich text editor  
* Attachment uploader  
* Schedule picker  
* Publish button

---

## **🧩 Student / Teacher Notice Board**

* Pinned notices section  
* Latest notices feed  
* Category filters  
* Search bar  
* Notice detail drawer/page

---

## **🧩 Guardian View**

* Important notices only feed  
* Exam / fee / holiday categories

---

# **12\. UX Flow**

---

## **Student UX**

1. Bell icon shows new notice  
2. Opens notice board  
3. Pinned urgent notice first  
4. Opens details  
5. Downloads file

---

## **Admin UX**

1. Create notice in \<2 min flow  
2. Preview audience reach  
3. Publish confidently

---

# **13\. Validation Rules**

* title required  
* content required  
* at least one audience required  
* publish\_at required for scheduled notices  
* expire\_at \> publish\_at  
* attachment size configurable  
* allowed file types only

---

# **14\. Edge Cases**

* Scheduled publish time passed during edit  
* Audience removed after publish  
* Expired pinned notice still cached  
* Duplicate notices created  
* Broken attachment link  
* User deactivated after read log  
* Notice edited after users already read it

---

# **15\. Dependencies**

* Notification Management  
* Authentication Module  
* User Management  
* Class & Batch Module  
* File Storage Module

---

# **16\. Non-Functional Requirements**

* Notice feed load \< 2 sec  
* Bulk audience publish scalable  
* Attachment delivery reliable  
* Mobile responsive mandatory  
* Audit trail for published notices

---

# **17\. Assumptions**

* Notice board replaces many paper notice processes.  
* Students mainly access via mobile.  
* Guardians prefer short important notices.

---

# **18\. Open Questions**

* Should teachers publish notices directly or approval required?  
* Need multilingual Bangla/English notices?  
* Need website public notice sync later?  
* Need mandatory acknowledgement for fee/exam notices only?

# **🧩 MODULE: DASHBOARD SYSTEM**

## **1\. Purpose**

The Dashboard System provides a **role-based operational home screen** for each logged-in user. It acts as the first interaction point after authentication and presents the most important summaries, urgent alerts, today’s planner items, pending tasks, and quick actions by consuming data from authoritative modules such as class scheduling, attendance, assessments, payments, payroll, accounting, notices, and notifications.

The dashboard does **not** own business data.  
It owns:

* summary aggregation  
* widget presentation  
* role-based visibility  
* action prioritization  
* urgent alert surfacing  
* deep-link navigation into source modules

This module ensures that users do not need to manually open many screens to understand what requires attention today.

---

## **2\. Scope**

### **✅ In Scope**

* Role-based dashboard landing page after login  
* Dashboard for Student  
* Dashboard for Teacher  
* Dashboard for Admin  
* Dashboard for Accountant  
* Summary cards  
* Quick actions  
* Today planner widgets  
* Pending work queues  
* Recent updates widgets  
* Alert and exception sections  
* Deep links to source modules  
* Mobile responsive dashboard layout  
* Refresh and summary aggregation logic  
* Empty state handling  
* Widget ordering rules  
* Source module failure handling for widgets

### **❌ Out of Scope**

* Full custom drag-and-drop dashboard builder  
* User-configurable widget marketplace  
* Cross-branch BI analytics engine  
* AI-generated personalized dashboard insights  
* Guardian dashboard in Phase 1  
* Public website dashboard

---

## **3\. Actors**

* Student  
* Teacher  
* Admin  
* Accountant  
* System

---

## **4\. User Story**

### **Student User Story**

As a student,  
I want to see my classes, assessments, attendance, dues, results, notices, and urgent actions in one place,  
So that I can understand what I need to do today without opening many different modules.

### **Teacher User Story**

As a teacher,  
I want to see my class schedule, pending attendance tasks, upcoming live classes, pending evaluation work, notices, and quick teaching actions in one place,  
So that I can run my academic work efficiently and not miss operational tasks.

### **Admin User Story**

As an admin,  
I want to see institution-level academic operations, pending admissions, fee status, live class activity, notices, and urgent system exceptions from one dashboard,  
So that I can manage the coaching center proactively.

### **Accountant User Story**

As an accountant,  
I want to see collections, dues, pending verification tasks, payroll work, expense summaries, and balance status from one dashboard,  
So that I can run daily financial operations without switching across multiple modules.

---

## **5\. Design Principle**

The Dashboard System must follow this rule:

### **Dashboard owns presentation. Source modules own truth.**

Examples:

* attendance percentage comes from Attendance Module  
* today’s classes come from Scheduling / Student Class Access / Teacher Class Management  
* upcoming exams come from Assessment Module  
* payment due amount comes from Payment & Billing Module  
* payroll queue comes from Payroll Module  
* current balance comes from Accounting Module  
* notices and notification alerts come from Notice Board and Notification Module

The dashboard must never calculate business truth independently when the source module already defines it.

---

## **6\. Core Concepts**

### **6.1 Dashboard**

A role-specific home screen shown immediately after successful login.

### **6.2 Widget**

A reusable dashboard block showing one of the following:

* KPI summary  
* planner item  
* pending work queue  
* alert  
* recent update feed  
* quick action group

### **6.3 Summary Card**

A compact numeric tile showing one high-value metric, such as:

* Attendance %  
* Upcoming Exams  
* Pending Dues  
* Today’s Classes  
* Outstanding Dues  
* Payroll Pending

### **6.4 Planner Widget**

A time-oriented or task-oriented dashboard block used to help the user understand what is happening next.

Examples:

* Today’s Classes  
* Upcoming Classes  
* Next Live Class  
* Upcoming Exams  
* Payment Queue  
* Admissions Queue

### **6.5 Quick Action**

A short path to a high-frequency workflow.

Examples:

* Join Live Class  
* Take Attendance  
* Review Admissions  
* Collect Payment  
* Create Notice

### **6.6 Alert**

A system-prioritized message requiring user awareness or action.

Alert levels:

* INFO  
* WARNING  
* CRITICAL

### **6.7 Work Queue**

A list of pending items requiring action.

Examples:

* Pending Admissions  
* Pending Evaluation  
* Pending Manual Payment Verification  
* Salary Payment Pending

### **6.8 Deep Link**

A route that opens the source workflow directly from a dashboard widget.

Example:

* clicking “Pending Attendance” opens Attendance Roll Call  
* clicking “Pending Dues” opens Student Billing Details  
* clicking “Pending Admissions” opens Admission Review List

---

## **7\. Role-Based Dashboard Definition**

---

# **7.1 Student Dashboard**

## **7.1.1 Dashboard Goal**

The Student Dashboard shall function as:

**planner \+ urgent actions \+ academic summary \+ alerts**

The screen must answer:

* What do I have today?  
* What is coming next?  
* Is anything urgent?  
* Am I falling behind?  
* Where should I go now?

## **7.1.2 Student Dashboard Sections**

### **A. Greeting & Context Header**

Show:

* student name  
* current batch/class  
* current date  
* optional short greeting  
* current active academic session if applicable

### **B. Top Summary Cards**

The system shall show the following student cards:

1. **Attendance Percentage**  
   * current month or configured academic period  
   * value only, with deep link to My Attendance  
2. **Upcoming Exams Count**  
   * total upcoming assigned assessments not yet completed  
3. **Pending Due Amount**  
   * outstanding payable amount from billing module  
4. **Unread Notifications Count**  
   * count from notification center  
5. **Latest Result Summary**  
   * latest published result score/grade  
   * if no published result exists, show empty state

### **C. Quick Action Bar**

The system shall show role-relevant actions only.

Student quick actions:

* Join Live Class  
* Open Today’s Class  
* Start Ongoing Exam  
* View Results  
* Pay Due  
* Open Notice Board

Rules:

* Join Live Class appears highlighted only if a live class is currently joinable  
* Start Ongoing Exam appears highlighted only during active exam window  
* Pay Due appears visually emphasized if due exists  
* hidden actions must not render empty placeholders

### **D. Today Planner Section**

This is the primary student widget.

#### **Today’s Classes**

For each class show:

* subject  
* teacher name  
* start time  
* end time  
* delivery mode  
* room name or live state if applicable  
* action button:  
  * Open  
  * Join  
  * View Details

If no class exists today:

* show friendly zero state

### **E. Upcoming Section**

Show limited items, not full history.

#### **Upcoming Classes**

Show next 3–5 upcoming class items:

* date  
* time  
* subject  
* mode

#### **Upcoming Exams**

Show nearest upcoming assessments:

* assessment title  
* subject  
* scheduled start  
* duration  
* status badge

### **F. Academic Summary Section**

Show a compact academic overview only.

Widgets:

* latest result summary  
* recent performance trend  
* weak subjects / weak chapters if available from assessment module  
* continue learning card for recorded class continuation

### **G. Alerts Section**

Show only high-value alerts:

* fee due  
* overdue payment  
* exam starting soon  
* live class starting soon  
* result published  
* low attendance warning  
* new urgent notice

### **H. Recent Updates Section**

Show:

* recent notifications  
* latest notices  
* recent published results  
* latest teacher instruction if linked to current academic activity

## **7.1.3 Student Dashboard Behavior Rules**

* dashboard must prioritize actionable items before passive information  
* today’s classes must appear above historical summaries  
* student must never see another student’s data  
* exam actions must respect assessment timing rules  
* dues must reflect billing truth only  
* if a source module is unavailable, widget must show “temporarily unavailable” rather than wrong data

---

# **7.2 Teacher Dashboard**

## **7.2.1 Dashboard Goal**

The Teacher Dashboard shall function as:

**teaching operations center \+ pending tasks \+ readiness panel**

It must answer:

* What classes do I have today?  
* What needs action now?  
* Which live/attendance/evaluation tasks are pending?  
* Which academic groups need attention?

## **7.2.2 Teacher Dashboard Sections**

### **A. Greeting & Role Header**

Show:

* teacher name  
* today’s date  
* optional subject expertise summary or assigned batch count  
* unread urgent notifications badge

### **B. Top Summary Cards**

Show the following:

1. **Today’s Classes Count**  
2. **Pending Attendance Tasks**  
3. **Pending Evaluations**  
4. **Upcoming Live Classes Count**  
5. **Unread Notices / Notifications**

### **C. Quick Action Bar**

Teacher quick actions:

* Take Attendance  
* Start Live Class  
* Upload Material  
* Create Assessment  
* Review Submissions  
* View Schedule

Rules:

* Start Live Class should highlight nearest live session when valid  
* Take Attendance should deep link to the correct batch/date if one immediate class is in context  
* Review Submissions should show badge count if pending

### **D. Today’s Teaching Planner**

This is the core teacher widget.

For each class show:

* batch name  
* subject  
* time  
* delivery mode  
* room or live state  
* status:  
  * normal  
  * updated  
  * overridden  
  * cancelled  
* actions:  
  * open class detail  
  * start live class  
  * upload material  
  * take attendance

### **E. Next Live Class Focus Widget**

If the teacher has an upcoming live class:

* show class title  
* time remaining  
* batch  
* session readiness state  
* start button when allowed

If no upcoming live class:

* widget collapses or shows next scheduled class instead

### **F. Pending Work Section**

Show operational pending items:

* attendance not yet marked  
* written assessments pending evaluation  
* recently changed class schedule  
* upcoming class without uploaded material if detectable  
* urgent notice awaiting review

### **G. Student Performance Insight Section**

Keep this concise, operational, and not too analytics-heavy.

May show:

* weak batch/topic summary  
* low-performing assessment alerts  
* absent pattern warning  
* low engagement batch alert

### **H. Alerts Section**

Teacher-facing alerts:

* schedule changed  
* class cancelled  
* live session issue  
* urgent notice published  
* evaluation deadline approaching  
* class override applied

## **7.2.3 Teacher Dashboard Behavior Rules**

* teacher must see only assigned academic data  
* draft schedules must never appear  
* dashboard must prefer next actionable class over historical data  
* pending evaluation count must exclude already finalized submissions  
* material upload suggestions must not block use of the dashboard  
* if live session reference is missing, alert must link to class detail rather than provider directly

---

# **7.3 Admin Dashboard**

## **7.3.1 Dashboard Goal**

The Admin Dashboard shall function as:

**institution operations dashboard**

It must answer:

* What needs intervention today?  
* Which approvals are pending?  
* What is happening across classes, live sessions, fees, notices, and attendance?  
* Are there any operational exceptions or risks?

## **7.3.2 Admin Dashboard Sections**

### **A. Top KPI Cards**

The admin dashboard shall show:

1. **Active Students**  
2. **Active Teachers**  
3. **Pending Admissions**  
4. **Today’s Classes Count**  
5. **Pending Dues Count or Summary**  
6. **Active Live Sessions Count**

### **B. Quick Action Bar**

Admin quick actions:

* Review Admissions  
* Create Notice  
* Open Schedule Manager  
* Manage Students  
* Manage Teachers  
* Send Announcement

Optional later:

* Verify Payment  
* Create Live Session  
* Open Attendance Logs

### **C. Pending Admissions Queue**

Show the most immediate pending admission items:

* applicant name  
* class  
* batch  
* phone  
* submitted date  
* status  
* actions:  
  * review  
  * approve  
  * reject

### **D. Today’s Academic Operations Section**

Show:

* total classes today  
* upcoming live classes  
* cancelled/overridden class indicators  
* attendance exception summary  
* recorded class publish or linkage anomalies if later required

### **E. Payment & Billing Snapshot**

Show:

* due student count  
* overdue student count  
* today’s collected amount if available  
* pending manual verification count if available  
* deep link to payment module

### **F. Alerts & Exceptions Section**

This is one of the most important admin areas.

Possible admin alerts:

* schedule conflicts unresolved  
* class override created  
* live session issue  
* attendance exceptions  
* notification delivery failures  
* overdue fee spike  
* batch low attendance risk  
* missing teacher assignment anomaly

### **G. Notice & Communication Snapshot**

Show:

* recently published notices  
* unread high-priority notices  
* recent announcement activity  
* failed delivery summary from notification system if available

### **H. Academic Summary Snapshot**

Show compact institution-level academic status:

* upcoming exams count  
* result publication pending count  
* batch-level weak performance warning if later supported  
* quick view of class execution pace

## **7.3.3 Admin Dashboard Behavior Rules**

* admin dashboard must prioritize pending approvals and exceptions above passive totals  
* actions must open source workflow in one click  
* critical issues should appear before summary cards on mobile if necessary  
* admin must see system-wide scope within permission limits  
* dashboard must not replace full management screens

---

# **7.4 Accountant Dashboard**

## **7.4.1 Dashboard Goal**

The Accountant Dashboard shall function as:

**finance operations dashboard**

It must answer:

* What money is due?  
* What has been collected?  
* What requires verification?  
* What payroll work is pending?  
* What is today’s financial attention area?

## **7.4.2 Accountant Dashboard Sections**

### **A. Top KPI Cards**

The system shall show:

1. **This Month Collection**  
2. **Outstanding Dues**  
3. **Pending Manual Payment Verifications**  
4. **Payroll Pending**  
5. **This Month Expenses**  
6. **Current Balance**

### **B. Quick Action Bar**

Accountant quick actions:

* Collect Payment  
* Verify Payment  
* Run Payroll  
* Mark Salary Paid  
* Add Expense  
* View Financial Summary

### **C. Payment Queue Section**

Show high-priority payment work:

* overdue invoices  
* today due invoices  
* pending manual verification items  
* failed or mismatched transactions  
* students with repeated due status

Each row may show:

* student name  
* batch  
* invoice no  
* amount  
* due date  
* status  
* action

### **D. Payroll Queue Section**

Show:

* current payroll cycle status  
* draft payroll pending review  
* finalized payroll awaiting payment marking  
* salary exceptions  
* teachers/employees with missing configuration if detectable

### **E. Financial Summary Section**

Show concise management summary:

* income vs expense for current month  
* payroll outflow  
* recent large expenses  
* current balance status  
* simple trend widget if supported

### **F. Alerts Section**

Show:

* overdue invoice spike  
* duplicate payment mismatch  
* manual payment awaiting verification too long  
* payroll not run for current period  
* low balance warning if rule defined later

## **7.4.3 Accountant Dashboard Behavior Rules**

* transaction queues must appear above historical charts  
* accountant dashboard must emphasize unresolved items first  
* finance numbers must come only from payment/payroll/accounting truth sources  
* salary and expense actions must respect permissions  
* dashboard must not expose unauthorized student academic data

---

## **8\. Functional Requirements**

### **8.1 General Framework**

* FR-DB-001: System shall redirect authenticated users to role-based dashboard after login.  
* FR-DB-002: System shall determine dashboard layout based on authenticated role.  
* FR-DB-003: System shall render only widgets allowed for that role.  
* FR-DB-004: System shall hide widgets when source module permission is absent.  
* FR-DB-005: System shall support desktop and mobile responsive dashboard rendering.  
* FR-DB-006: System shall load alert, summary, planner, and quick-action zones separately to reduce full-page failure.  
* FR-DB-007: System shall support deep links from widgets to source modules.  
* FR-DB-008: System shall gracefully show empty state when no data exists.  
* FR-DB-009: System shall gracefully show temporary unavailable state when source data cannot be loaded.  
* FR-DB-010: System shall support configurable widget ordering by role.  
* FR-DB-011: System shall not allow end users to access data outside their permission scope through widget APIs.  
* FR-DB-012: System shall support summary caching for performance-sensitive widgets.  
* FR-DB-013: System shall refresh time-sensitive widgets periodically or on manual refresh.  
* FR-DB-014: System shall show unread count badges where relevant.  
* FR-DB-015: System shall support alert severity ordering.

### **8.2 Student Dashboard Functional Requirements**

* FR-DB-STU-001: System shall show attendance percentage card.  
* FR-DB-STU-002: System shall show upcoming exams count card.  
* FR-DB-STU-003: System shall show pending due amount card.  
* FR-DB-STU-004: System shall show unread notifications count card.  
* FR-DB-STU-005: System shall show latest result summary card.  
* FR-DB-STU-006: System shall show today’s classes planner widget.  
* FR-DB-STU-007: System shall show upcoming classes widget.  
* FR-DB-STU-008: System shall show upcoming exams widget.  
* FR-DB-STU-009: System shall show recent academic summary widget.  
* FR-DB-STU-010: System shall show weak subjects/chapters if available from assessment module.  
* FR-DB-STU-011: System shall show continue learning recorded content widget when applicable.  
* FR-DB-STU-012: System shall show quick actions relevant to active student workflows.  
* FR-DB-STU-013: System shall highlight joinable live class when current time allows joining.  
* FR-DB-STU-014: System shall highlight ongoing exam when assessment attempt window is active.  
* FR-DB-STU-015: System shall show urgent alerts such as fee due, exam soon, live class soon, result published, low attendance, and urgent notice.  
* FR-DB-STU-016: System shall never show non-student or cross-student data.

### **8.3 Teacher Dashboard Functional Requirements**

* FR-DB-TEA-001: System shall show today’s classes count card.  
* FR-DB-TEA-002: System shall show pending attendance tasks card.  
* FR-DB-TEA-003: System shall show pending evaluations card.  
* FR-DB-TEA-004: System shall show upcoming live classes card.  
* FR-DB-TEA-005: System shall show unread notices/notifications card.  
* FR-DB-TEA-006: System shall show today’s teaching planner widget.  
* FR-DB-TEA-007: System shall show next live class focus widget when applicable.  
* FR-DB-TEA-008: System shall show pending work queue.  
* FR-DB-TEA-009: System shall show class updates, override alerts, and cancellation alerts.  
* FR-DB-TEA-010: System shall support quick actions for attendance, live class, material upload, assessment creation, submission review, and schedule open.  
* FR-DB-TEA-011: System shall display only assigned teacher data.  
* FR-DB-TEA-012: System shall not show unpublished draft schedule items.

### **8.4 Admin Dashboard Functional Requirements**

* FR-DB-ADM-001: System shall show active students count card.  
* FR-DB-ADM-002: System shall show active teachers count card.  
* FR-DB-ADM-003: System shall show pending admissions count card.  
* FR-DB-ADM-004: System shall show today’s classes count card.  
* FR-DB-ADM-005: System shall show pending dues summary card.  
* FR-DB-ADM-006: System shall show active live sessions count card.  
* FR-DB-ADM-007: System shall show admissions queue widget.  
* FR-DB-ADM-008: System shall show today’s academic operations widget.  
* FR-DB-ADM-009: System shall show communication and notice snapshot widget.  
* FR-DB-ADM-010: System shall show alerts and exceptions widget.  
* FR-DB-ADM-011: System shall show payment and billing snapshot widget.  
* FR-DB-ADM-012: System shall support quick actions for admissions, notice creation, schedule opening, student management, teacher management, and announcement sending.  
* FR-DB-ADM-013: System shall provide one-click navigation from admissions queue to approval flow.  
* FR-DB-ADM-014: System shall surface operational exceptions ahead of passive summaries where severity is high.

### **8.5 Accountant Dashboard Functional Requirements**

* FR-DB-ACC-001: System shall show monthly collections card.  
* FR-DB-ACC-002: System shall show outstanding dues card.  
* FR-DB-ACC-003: System shall show pending manual verification card.  
* FR-DB-ACC-004: System shall show payroll pending card.  
* FR-DB-ACC-005: System shall show monthly expenses card.  
* FR-DB-ACC-006: System shall show current balance card.  
* FR-DB-ACC-007: System shall show payment queue widget.  
* FR-DB-ACC-008: System shall show payroll queue widget.  
* FR-DB-ACC-009: System shall show finance summary widget.  
* FR-DB-ACC-010: System shall show finance alerts widget.  
* FR-DB-ACC-011: System shall support quick actions for payment collection, payment verification, payroll run, marking salary paid, adding expense, and viewing financial summary.  
* FR-DB-ACC-012: System shall display only finance-authorized data to accountant role.

### **8.6 Alerts & Priority Requirements**

* FR-DB-ALT-001: System shall classify dashboard alerts by severity.  
* FR-DB-ALT-002: Critical alerts shall appear before normal informational widgets.  
* FR-DB-ALT-003: Expired alerts shall no longer appear in active alert sections.  
* FR-DB-ALT-004: Duplicate alerts should be collapsed or minimized when they refer to the same user-facing issue.  
* FR-DB-ALT-005: Alert clicks shall open the correct workflow or detail page.  
* FR-DB-ALT-006: Dashboard shall support role-specific alert definitions.

---

## **9\. Business Rules**

* Dashboard is the first screen after successful login unless a forced workflow redirect is required.  
* Dashboard must not independently create or mutate core business data.  
* Every summary displayed must come from a valid source module or trusted cache derived from that module.  
* Widgets must be permission-aware.  
* A user may only view dashboard data related to their own role and access scope.  
* Student dashboard is action-first, not report-first.  
* Teacher dashboard is operations-first, not analytics-first.  
* Admin dashboard is exception-first and approvals-first.  
* Accountant dashboard is queue-first and transaction-first.  
* Critical alerts take precedence over lower-priority widgets.  
* Quick actions must only be shown when the underlying workflow exists in the system.  
* A hidden, disabled, or not-yet-implemented module must not expose fake dashboard actions.  
* Empty widgets must show helpful messages instead of blank containers.  
* Mobile layout may reorder sections so urgent items appear sooner.  
* Dashboard caching must not display materially incorrect stale values for highly time-sensitive actions like join live class or start exam.

---

## **10\. User Flow / Process Flow**

### **10.1 General Login to Dashboard Flow**

1. User submits valid login credentials.  
2. Authentication module validates credentials and returns role-based session.  
3. System identifies dashboard type for role.  
4. Dashboard layout configuration is loaded.  
5. System requests summary data from source modules or dashboard cache.  
6. Critical alerts are resolved first.  
7. Quick actions are resolved second.  
8. Planner/work queue widgets are resolved.  
9. Secondary summary widgets are loaded.  
10. User lands on dashboard and may open a linked workflow directly.

---

### **10.2 Student Dashboard Flow**

1. Student logs in.  
2. System loads student summary cards:  
   * attendance  
   * upcoming exams  
   * dues  
   * unread notifications  
   * latest result  
3. System loads today’s classes.  
4. System checks whether:  
   * any live class is currently joinable  
   * any exam is currently ongoing  
5. If yes, highlight the relevant action.  
6. System loads upcoming classes and exams.  
7. System loads urgent alerts.  
8. Student clicks:  
   * Join Live Class, or  
   * Start Exam, or  
   * View Result, or  
   * Pay Due, or  
   * Notice Board

#### **Student UX Intent**

The student should understand today’s academic reality within a few seconds:

* what to attend  
* what is urgent  
* what is due  
* what is next

---

### **10.3 Teacher Dashboard Flow**

1. Teacher logs in.  
2. System loads counts:  
   * today’s classes  
   * pending attendance  
   * pending evaluations  
   * upcoming live classes  
   * unread notices  
3. System loads today’s schedule list.  
4. System identifies nearest upcoming live class.  
5. System loads pending work queue:  
   * attendance pending  
   * evaluations pending  
   * recent schedule changes  
6. System loads urgent teacher alerts.  
7. Teacher clicks quick action:  
   * Take Attendance  
   * Start Live Class  
   * Upload Material  
   * Create Assessment  
   * Review Submissions

#### **Teacher UX Intent**

The teacher should immediately see:

* today’s academic workload  
* what must be completed before class  
* what is pending after class  
* what changed since the last login

---

### **10.4 Admin Dashboard Flow**

1. Admin logs in.  
2. System loads institution KPI cards.  
3. System loads pending admissions queue.  
4. System loads today’s operations:  
   * classes  
   * live sessions  
   * attendance exceptions  
5. System loads payment snapshot.  
6. System loads notice/communication snapshot.  
7. System loads critical alerts and exceptions.  
8. Admin opens target workflow from dashboard.

#### **Admin UX Intent**

The admin should be able to answer:

* what is pending  
* what is going wrong  
* what requires decision now

---

### **10.5 Accountant Dashboard Flow**

1. Accountant logs in.  
2. System loads top finance cards.  
3. System loads payment queue.  
4. System loads payroll queue.  
5. System loads expense and financial summary.  
6. System loads finance alerts.  
7. Accountant uses quick action to open immediate finance workflow.

#### **Accountant UX Intent**

The accountant should be able to answer:

* what must be collected  
* what must be verified  
* what must be paid  
* what financial issue requires attention

---

## **11\. Data Model**

### **📘 DashboardLayout**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| role | Enum | STUDENT / TEACHER / ADMIN / ACCOUNTANT |
| layout\_code | String | Layout key |
| is\_default | Boolean | Default role layout |
| created\_at | Timestamp | Created time |
| updated\_at | Timestamp | Updated time |

### **📘 DashboardWidgetConfig**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| layout\_id | UUID | Dashboard layout reference |
| widget\_code | String | Unique widget identifier |
| widget\_type | Enum | CARD / LIST / ACTION / ALERT / FEED |
| title | String | Widget label |
| display\_order | Integer | Render order |
| is\_active | Boolean | Active status |
| min\_role | Enum | Minimum role scope if needed |
| refresh\_interval\_sec | Integer | Auto refresh interval |
| source\_module | String | Owning source module |
| source\_endpoint | String | Data endpoint key |
| deep\_link\_route | String | Default navigation route |

### **📘 DashboardAlert**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| role | Enum | Target role |
| user\_id | UUID nullable | Nullable for role-wide alerts |
| alert\_code | String | Unique alert key |
| title | String | Alert title |
| body | Text | Alert message |
| severity | Enum | INFO / WARNING / CRITICAL |
| deep\_link | String | Navigation route |
| status | Enum | ACTIVE / DISMISSED / EXPIRED |
| created\_at | Timestamp | Created time |
| expires\_at | Timestamp nullable | Auto-hide time |

### **📘 DashboardSnapshotCache**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| role | Enum | Role |
| user\_id | UUID | User |
| widget\_code | String | Widget key |
| payload\_json | JSON | Cached rendered data |
| generated\_at | Timestamp | Cache generation time |
| expires\_at | Timestamp | Cache expiry |

### **📘 DashboardActionConfig**

| Field | Type | Description |
| ----- | ----- | ----- |
| id | UUID | Primary key |
| role | Enum | Target role |
| action\_code | String | JOIN\_LIVE / TAKE\_ATTENDANCE etc |
| label | String | Button label |
| icon | String | UI icon key |
| route | String | Destination route |
| is\_active | Boolean | Visibility flag |
| display\_order | Integer | Order |

---

## **12\. API Contracts**

### **12.1 Get My Dashboard**

`GET /dashboard/me`

**Purpose**  
Returns fully aggregated dashboard payload for authenticated user.

**Response Example**

{  
  "role": "STUDENT",  
  "header": {  
    "display\_name": "Rahim",  
    "context\_label": "Class 10 • Morning Batch"  
  },  
  "summary\_cards": \[  
    {  
      "code": "ATTENDANCE\_PERCENTAGE",  
      "label": "Attendance",  
      "value": "92%",  
      "deep\_link": "/attendance/my"  
    }  
  \],  
  "quick\_actions": \[  
    {  
      "code": "JOIN\_LIVE\_CLASS",  
      "label": "Join Live Class",  
      "enabled": true,  
      "highlighted": true,  
      "route": "/student/classes/live/123"  
    }  
  \],  
  "widgets": \[\],  
  "alerts": \[\]  
}

### **12.2 Get Widget Data**

`GET /dashboard/widgets/{widgetCode}`

**Purpose**  
Refresh or lazy-load one widget only.

### **12.3 Refresh Dashboard**

`POST /dashboard/refresh`

**Purpose**  
Force refresh user dashboard snapshot.

### **12.4 Dismiss Alert**

`POST /dashboard/alerts/{id}/dismiss`

**Purpose**  
Dismiss user-dismissible dashboard alert.

### **12.5 Get Dashboard Layout Metadata**

`GET /dashboard/layout`

**Purpose**  
Returns widget ordering and layout definition for current role.

---

## **13\. UI Components**

## **13.1 Shared Dashboard Shell**

All dashboards shall include:

* top header  
* greeting/context strip  
* summary card row  
* quick action row  
* alert zone  
* widget grid / stacked sections  
* responsive spacing  
* deep-link controls

---

## **13.2 Student Dashboard UI Components**

### **Header Area**

* greeting text  
* student name  
* class/batch label  
* notification bell

### **Summary Cards Row**

* Attendance %  
* Upcoming Exams  
* Pending Dues  
* Unread Notifications  
* Latest Result

### **Quick Action Row**

* Join Live Class  
* Open Today’s Class  
* Start Ongoing Exam  
* View Results  
* Pay Due  
* Notice Board

### **Main Widgets**

#### **Today’s Classes Widget**

Layout:

* list cards  
* each row shows subject, teacher, time, mode, action button

#### **Upcoming Exams Widget**

Layout:

* compact vertical list

#### **Upcoming Classes Widget**

Layout:

* compact vertical list

#### **Academic Status Widget**

* latest result  
* recent trend indicator  
* weak chapter hint

#### **Continue Learning Widget**

* thumbnail/title  
* resume button  
* subject

#### **Alerts & Updates Widget**

* urgent notice  
* result published  
* due reminder  
* live class reminder

---

## **13.3 Teacher Dashboard UI Components**

### **Header Area**

* greeting  
* teacher name  
* today date  
* urgent badge

### **Summary Cards**

* Today’s Classes  
* Pending Attendance  
* Pending Evaluation  
* Upcoming Live Classes  
* Unread Notices

### **Quick Actions**

* Take Attendance  
* Start Live Class  
* Upload Material  
* Create Assessment  
* Review Submissions  
* View Schedule

### **Main Widgets**

#### **Today’s Teaching Planner**

* schedule rows  
* batch  
* subject  
* time  
* mode  
* room/live state  
* status badge

#### **Next Live Class Widget**

* countdown  
* title  
* start button

#### **Pending Tasks Widget**

* attendance pending  
* evaluation pending  
* schedule changes  
* notice requiring attention

#### **Academic Insight Widget**

* weak batch/topic summary  
* low score signal  
* absence pattern signal

#### **Teacher Alerts Widget**

* class override  
* live link issue  
* urgent notice  
* evaluation deadline

---

## **13.4 Admin Dashboard UI Components**

### **Header Area**

* admin greeting  
* institution context  
* notification badge

### **KPI Cards**

* Active Students  
* Active Teachers  
* Pending Admissions  
* Today’s Classes  
* Pending Dues  
* Active Live Sessions

### **Quick Actions**

* Review Admissions  
* Create Notice  
* Open Schedule  
* Manage Students  
* Manage Teachers  
* Send Announcement

### **Main Widgets**

#### **Admissions Queue Widget**

* applicant name  
* class/batch  
* date  
* action buttons

#### **Today’s Operations Widget**

* today’s classes  
* upcoming live classes  
* attendance exceptions  
* cancellations/overrides

#### **Finance Snapshot Widget**

* due count  
* overdue count  
* collected summary

#### **Communication Snapshot Widget**

* latest notices  
* recent announcements  
* failed notification summary

#### **Alerts & Exceptions Widget**

* high-severity issues  
* schedule conflict warning  
* notification failure  
* attendance risk  
* live session issue

---

## **13.5 Accountant Dashboard UI Components**

### **Header Area**

* accountant greeting  
* financial day context if needed

### **KPI Cards**

* This Month Collection  
* Outstanding Dues  
* Pending Verification  
* Payroll Pending  
* This Month Expenses  
* Current Balance

### **Quick Actions**

* Collect Payment  
* Verify Payment  
* Run Payroll  
* Mark Salary Paid  
* Add Expense  
* View Summary

### **Main Widgets**

#### **Payment Queue Widget**

* overdue invoices  
* today due items  
* manual verification items  
* mismatch items

#### **Payroll Queue Widget**

* payroll cycle status  
* pending salary marking  
* payroll exceptions

#### **Finance Summary Widget**

* income vs expense  
* payroll outflow  
* expense snapshot  
* current balance trend

#### **Finance Alerts Widget**

* overdue spike  
* duplicate payment mismatch  
* verification pending too long  
* payroll delay warning

---

## **14\. UX Flow**

### **14.1 Widget Priority Order**

Dashboard sections must be ordered by practical use priority:

1. Critical alerts  
2. Immediate actions  
3. Today planner / pending queue  
4. Core summary cards  
5. Recent updates / secondary information

### **14.2 Student UX Rule**

Student dashboard must feel like:  
**today plan \+ urgent schoolwork \+ financial reminder**

### **14.3 Teacher UX Rule**

Teacher dashboard must feel like:  
**today’s academic operations \+ pending work**

### **14.4 Admin UX Rule**

Admin dashboard must feel like:  
**institution control room**

### **14.5 Accountant UX Rule**

Accountant dashboard must feel like:  
**daily finance operations board**

### **14.6 Mobile UX Requirements**

* cards stack vertically  
* quick actions remain easy to tap  
* urgent alerts appear before lower sections  
* planner items show compact rows  
* no wide table should block basic usage  
* overflow lists must provide “view all” action rather than overloading dashboard

---

## **15\. Validation Rules**

* role must be resolved before dashboard payload generation  
* widget must belong to current role layout  
* hidden modules must not expose active quick actions  
* deep links must map to valid routes  
* dashboard metric values must be validated before render  
* stale cached joinable-action data must not remain active beyond safe time threshold  
* negative values must not be shown for non-negative metrics like dues count, exam count, attendance percentage  
* student widgets must require authenticated student context  
* teacher widgets must require assigned teacher context  
* admin widgets must require admin permission  
* accountant widgets must require finance permission

---

## **16\. Edge Cases**

* user logs in but all widgets return no data  
* source module is slow or temporarily unavailable  
* student has no class today  
* student has dues but no payable online method enabled  
* teacher has class today but live session ref missing  
* teacher pending evaluation widget returns stale count  
* admin sees admission count but no actionable records due to permission mismatch  
* accountant sees payroll pending but payroll period not yet initialized  
* alert points to deleted entity  
* user role changed after previous session  
* cache contains outdated quick action state  
* no notices / no notifications / no upcoming assessments  
* batch change causes student dashboard items to shift immediately  
* large morning login burst causes partial widget load

---

## **17\. Dependencies**

* Authentication & Authorization Module  
* User Management Module  
* Profile Management Module  
* Class Scheduling System  
* Teacher Class Management System  
* Student Class Access System  
* Attendance Management System  
* Assessment, Exam & Quiz Management System  
* Live Class Management System  
* Recorded Class / Content Management System  
* Payment & Billing Management System  
* Payroll Management System  
* Accounting & Financial Tracking System  
* Notification Management System  
* Notice Board Management System

---

## **18\. Non-Functional Requirements**

* dashboard initial load target: under 2 seconds for standard conditions  
* critical alert zone must render before lower-priority secondary widgets where feasible  
* dashboard must support morning concurrent login spikes  
* widgets must support graceful partial-load rendering  
* source aggregation must be cache-aware  
* role isolation must be strict  
* dashboard must be fully mobile responsive  
* time-sensitive widgets must avoid materially stale data  
* quick actions must remain tappable on low-resolution Android devices common in Bangladesh usage scenarios  
* dashboard must be usable under unstable network conditions with retry-friendly widget loading  
* all dashboard data access must be audit-safe and permission controlled

---

## **19\. Assumptions**

* dashboard is the default landing screen after login  
* most students and teachers will access through mobile  
* the dashboard should reduce clicks, not replace modules  
* users need urgent action visibility more than full analytical detail on landing  
* modules already defined in the SRS remain the authority for business truth  
* guardian dashboard can be added later using the same widget framework

---

## **20\. Open Questions**

* should users be allowed limited personalization later, such as pinning one widget higher?  
* should student dashboard include motivational messaging later?  
* should teacher dashboard include class readiness scoring later?  
* should admin dashboard include branch segmentation in future?  
* should accountant dashboard support downloadable mini-summary directly from the home screen?

---

