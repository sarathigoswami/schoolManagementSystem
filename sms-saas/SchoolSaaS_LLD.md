# School Management SaaS Platform
## Low-Level Design (LLD) Document

**Version:** 1.0  
**Date:** December 2, 2025  
**Author:** Chief Software Architect  
**Classification:** Architecture & Implementation  
**Related Document:** HLD v1.0

---

## Executive Summary

This Low-Level Design (LLD) document provides detailed implementation specifications for the School Management SaaS Platform defined in the HLD. It translates architectural decisions into concrete code patterns, database schemas, API contracts, and system workflows that engineering teams will use for implementation.

**Document Scope:**
- Detailed module-level designs (StudentParent, Teacher, Examination, Admin, Leadership, TenantConfig)
- Complete database schema with migration strategies
- API endpoint specifications (REST + GraphQL)
- Inter-module communication patterns (Event Bus, Saga patterns)
- Security implementation details (RLS, encryption, token management)
- Performance optimization strategies
- Testing strategies and test data models
- Operational runbooks

---

## 1. Core Module Architecture & Design Patterns

### 1.1 Module Communication & Dependencies

#### Module Dependency Graph

```
┌─────────────────────────────────────────────────────────┐
│                    TenantContext Module                 │
│  (Cross-cutting: Tenant ID extraction, RLS setup)      │
└────────┬────────────────────────────────────────────────┘
         │ (All modules depend on TenantContext)
         │
    ┌────▼─────────────────────────────────────────────┐
    │              Core Modules (Spring Modulith)      │
    ├───────────────────────────────────────────────────┤
    │                                                   │
    │  ┌─────────────────┐      ┌──────────────────┐  │
    │  │ StudentParent   │      │ Teacher Module   │  │
    │  │ Module          │────→ │ (read-only)      │  │
    │  │ - Dashboards    │      │ - Class list     │  │
    │  │ - Grades        │      │ - Attendance     │  │
    │  │ - Fees          │      │                  │  │
    │  │ - Assignments   │      └──────────────────┘  │
    │  └────────┬────────┘                             │
    │           │                                      │
    │           ▼ (publishes: StudentGradePublished)  │
    │      ┌──────────────────┐                        │
    │      │   Event Bus      │                        │
    │      │  (Apache Kafka)  │                        │
    │      └────────┬─────────┘                        │
    │               │                                  │
    │  ┌────────────┼────────────┬────────────────┐   │
    │  ▼            ▼            ▼                ▼   │
    │ ┌──────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ │
    │ │Exam  │ │Teacher  │ │Admin     │ │Leadership│ │
    │ │Module│ │Module   │ │Module    │ │Module    │ │
    │ │      │ │(listen) │ │(listen)  │ │(listen)  │ │
    │ └──────┘ └─────────┘ └──────────┘ └──────────┘ │
    │                                                   │
    │  ┌──────────────────────────────────────────┐   │
    │  │   TenantConfig Module                    │   │
    │  │   - Feature Flags                        │   │
    │  │   - Subscription Settings                │   │
    │  │   - Branding Config                      │   │
    │  └──────────────────────────────────────────┘   │
    │                                                   │
    └───────────────────────────────────────────────────┘
```

#### Module Definition via Spring Modulith

```java
// src/main/java/com/schoolsaas/StudentParentModule.java
package com.schoolsaas.studentparent;

import org.springframework.modulith.core.Named;
import org.springframework.modulith.ArchitectureTests;

@Named("studentparent")
public interface StudentParentModule {
    // Marker interface for module definition
}

// src/main/java/com/schoolsaas/studentparent/api/StudentParentApi.java
package com.schoolsaas.studentparent.api;

import org.springframework.modulith.core.ModuleExposed;

@ModuleExposed
public interface StudentParentApi {
    // Public API exposed to other modules
    Dashboard getDashboard(UUID schoolId, UUID studentId);
    List<Grade> getGrades(UUID schoolId, UUID studentId);
    List<Fee> getFees(UUID schoolId, UUID studentId);
}

// Implementation (internal to module)
@Service
class StudentParentService implements StudentParentApi {
    // Implementation details
}
```

#### Event-Driven Communication Pattern

```java
// Event definition (shared across modules)
// src/main/java/com/schoolsaas/event/StudentGradePublishedEvent.java
@Data
@AllArgsConstructor
public class StudentGradePublishedEvent {
    UUID schoolId;
    UUID examId;
    UUID studentId;
    BigDecimal marksObtained;
    String gradeLetterr;
    LocalDateTime publishedAt;
    
    // Kafka topic: student-grade-published
}

// Publisher (Examination Module)
@Service
@RequiredArgsConstructor
public class ResultPublishingService {
    private final ApplicationEventPublisher eventPublisher;
    private final ResultRepository resultRepository;
    
    @Transactional
    public void publishExamResults(UUID examId) {
        List<Grade> grades = resultRepository.findByExamId(examId);
        
        for (Grade grade : grades) {
            // Persist to database
            gradeRepository.save(grade);
            
            // Publish event (async via Kafka)
            StudentGradePublishedEvent event = new StudentGradePublishedEvent(
                grade.getSchoolId(),
                examId,
                grade.getStudentId(),
                grade.getMarksObtained(),
                grade.getGradeLetterr,
                LocalDateTime.now()
            );
            
            eventPublisher.publishEvent(event);
            
            // Also publish to WebSocket for real-time UI updates
            webSocketService.notifyStudentResultAvailable(
                grade.getStudentId(),
                grade
            );
        }
    }
}

// Listeners (Teacher, Admin, Leadership modules)
@Component
@RequiredArgsConstructor
public class TeacherResultListener {
    private final NotificationService notificationService;
    private final TeacherDashboardService dashboardService;
    
    @EventListener
    @Async("asyncExecutor")  // Non-blocking
    public void onStudentGradePublished(StudentGradePublishedEvent event) {
        // Update teacher's dashboard analytics
        dashboardService.incrementResultsPublishedCount(
            event.getSchoolId(),
            event.getExamId()
        );
        
        // Send notification: "Results published for Class X"
        notificationService.notifyTeachers(
            event.getSchoolId(),
            "exam_results_published",
            event
        );
    }
}

// Spring Cloud Stream configuration
@Configuration
public class KafkaConfiguration {
    @Bean
    public Function<StudentGradePublishedEvent, StudentGradePublishedEvent> 
        processGradePublished() {
        return event -> {
            // Transform or enrich event if needed
            return event;
        };
    }
}
```

#### Inter-Module Method Invocation (Synchronous)

```java
// When immediate response needed (e.g., validate teacher exists before saving grade)
@Service
@RequiredArgsConstructor
public class GradingService {
    private final TeacherApi teacherApi;  // Injected from Teacher Module
    private final GradeRepository gradeRepository;
    
    @Transactional
    public Grade saveGrade(UUID schoolId, UUID teacherId, GradeCreateRequest req) {
        // Validate teacher exists in this school (sync call)
        Teacher teacher = teacherApi.getTeacherOrThrow(schoolId, teacherId);
        
        if (teacher == null) {
            throw new TeacherNotFoundException("Teacher not found");
        }
        
        Grade grade = new Grade();
        grade.setSchoolId(schoolId);
        grade.setTeacherId(teacherId);
        grade.setMarksObtained(req.getMarks());
        
        return gradeRepository.save(grade);
    }
}

// Exposed API from Teacher Module
@ModuleExposed
public interface TeacherApi {
    @Nullable
    Teacher getTeacher(UUID schoolId, UUID teacherId);
    
    default Teacher getTeacherOrThrow(UUID schoolId, UUID teacherId) {
        return getTeacher(schoolId, teacherId);
    }
}
```

---

## 2. StudentParent Module (Detailed Design)

### 2.1 Module Structure

```
studentparent/
├── api/
│   ├── StudentParentController.java      (REST endpoints)
│   ├── StudentParentApi.java              (exposed interface)
│   └── StudentParentGraphQL.java          (GraphQL resolvers)
├── service/
│   ├── DashboardService.java
│   ├── GradingService.java
│   ├── FeeService.java
│   └── AssignmentService.java
├── repository/
│   ├── StudentRepository.java
│   ├── GradeRepository.java
│   ├── FeeRepository.java
│   └── AssignmentRepository.java
├── entity/
│   ├── Student.java
│   ├── Grade.java
│   ├── Fee.java
│   ├── FeePayment.java
│   └── Assignment.java
├── dto/
│   ├── StudentDashboardDTO.java
│   ├── GradeDTO.java
│   └── FeePaymentDTO.java
├── event/
│   ├── StudentGradePublishedEvent.java
│   └── StudentGradeListener.java
└── exception/
    └── StudentParentExceptions.java
```

### 2.2 Entity Design with Tenant Isolation

```java
// studentparent/entity/Student.java
@Entity
@Table(name = "students", indexes = {
    @Index(name = "idx_school_enrollment", columnList = "school_id,enrollment_number"),
    @Index(name = "idx_school_status", columnList = "school_id,status")
})
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Student {
    @Id
    private UUID id;
    
    @Column(nullable = false)
    private UUID schoolId;  // Tenant boundary - CRITICAL
    
    @Column(nullable = false, unique = true)
    private String enrollmentNumber;
    
    @Column(nullable = false)
    private String firstName;
    
    @Column(nullable = false)
    private String lastName;
    
    @Column(nullable = false)
    private LocalDate dateOfBirth;
    
    @Enumerated(EnumType.STRING)
    private Gender gender;  // M, F, OTHER
    
    @Column(unique = true)
    private String email;
    
    @Column
    private String phoneNumber;
    
    @Column
    private String addressLine1;
    
    @Column
    private String city;
    
    @Column
    private String state;
    
    @Column
    private String country;
    
    @Column(nullable = false)
    private LocalDate admissionDate;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private StudentStatus status;  // ACTIVE, INACTIVE, GRADUATED, TRANSFERRED
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
    
    @Column(nullable = false)
    private LocalDateTime updatedAt = LocalDateTime.now();
    
    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
    
    @Transient
    public int getAge() {
        return Period.between(dateOfBirth, LocalDate.now()).getYears();
    }
}

// Grade entity with RLS
@Entity
@Table(name = "grades", indexes = {
    @Index(name = "idx_grades_exam_student", columnList = "school_id,exam_id,student_id"),
    @Index(name = "idx_grades_student", columnList = "school_id,student_id")
})
@Data
@NoArgsConstructor
public class Grade {
    @Id
    private UUID id = UUID.randomUUID();
    
    @Column(nullable = false)
    private UUID schoolId;  // Tenant boundary
    
    @Column(nullable = false)
    private UUID studentId;
    
    @Column(nullable = false)
    private UUID examId;
    
    @Column(nullable = false)
    private UUID subjectId;
    
    @Column(nullable = false)
    private BigDecimal marksObtained;
    
    @Column(nullable = false)
    private BigDecimal totalMarks;
    
    @Column(length = 1)
    private String gradeLetter;  // A, B, C, D, F
    
    @Column
    private BigDecimal gradePoint;
    
    @Column
    private String remarks;
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
    
    @Column(nullable = false)
    private LocalDateTime updatedAt = LocalDateTime.now();
    
    @Transient
    public String getGradeCategory() {
        if (marksObtained == null || totalMarks == null) return "PENDING";
        BigDecimal percentage = marksObtained.divide(totalMarks, 2, RoundingMode.HALF_UP)
                                             .multiply(BigDecimal.valueOf(100));
        
        return switch (percentage.intValue()) {
            case int p when p >= 90 -> "EXCELLENT";
            case int p when p >= 75 -> "VERY_GOOD";
            case int p when p >= 60 -> "GOOD";
            case int p when p >= 45 -> "SATISFACTORY";
            default -> "NEEDS_IMPROVEMENT";
        };
    }
}
```

### 2.3 Repository Layer with Tenant Filtering

```java
// studentparent/repository/StudentRepository.java
@Repository
public interface StudentRepository extends JpaRepository<Student, UUID> {
    
    // Auto-filter by school_id from TenantContext
    @Query("""
        SELECT s FROM Student s 
        WHERE s.schoolId = :schoolId 
          AND s.status = 'ACTIVE'
        ORDER BY s.firstName, s.lastName
    """)
    List<Student> findActiveStudents(@Param("schoolId") UUID schoolId);
    
    // With pagination
    @Query("""
        SELECT s FROM Student s 
        WHERE s.schoolId = :schoolId
        ORDER BY s.createdAt DESC
    """)
    Page<Student> findAllBySchoolId(
        @Param("schoolId") UUID schoolId,
        Pageable pageable
    );
    
    // Find by enrollment number (unique per school)
    @Query("""
        SELECT s FROM Student s 
        WHERE s.schoolId = :schoolId 
          AND s.enrollmentNumber = :enrollmentNumber
    """)
    Optional<Student> findByEnrollmentNumber(
        @Param("schoolId") UUID schoolId,
        @Param("enrollmentNumber") String enrollmentNumber
    );
    
    // Count for analytics
    @Query("""
        SELECT COUNT(s) FROM Student s 
        WHERE s.schoolId = :schoolId 
          AND s.status = 'ACTIVE'
    """)
    long countActiveStudents(@Param("schoolId") UUID schoolId);
}

// Grade repository with RLS + efficient querying
@Repository
public interface GradeRepository extends JpaRepository<Grade, UUID> {
    
    @Query("""
        SELECT g FROM Grade g 
        WHERE g.schoolId = :schoolId 
          AND g.examId = :examId 
          AND g.studentId = :studentId
    """)
    Optional<Grade> findByExamAndStudent(
        @Param("schoolId") UUID schoolId,
        @Param("examId") UUID examId,
        @Param("studentId") UUID studentId
    );
    
    // Get all grades for a student (transcript)
    @Query("""
        SELECT g FROM Grade g 
        WHERE g.schoolId = :schoolId 
          AND g.studentId = :studentId
        ORDER BY g.createdAt DESC
    """)
    List<Grade> findStudentTranscript(
        @Param("schoolId") UUID schoolId,
        @Param("studentId") UUID studentId
    );
    
    // Bulk fetch grades for exam (result publishing)
    @Query("""
        SELECT g FROM Grade g 
        WHERE g.schoolId = :schoolId 
          AND g.examId = :examId
        ORDER BY g.studentId
    """)
    List<Grade> findByExamIdOptimized(
        @Param("schoolId") UUID schoolId,
        @Param("examId") UUID examId
    );
}
```

### 2.4 Service Layer with Business Logic

```java
// studentparent/service/DashboardService.java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class DashboardService {
    private final StudentRepository studentRepository;
    private final GradeRepository gradeRepository;
    private final FeeRepository feeRepository;
    private final AssignmentRepository assignmentRepository;
    private final CacheManager cacheManager;
    
    public StudentDashboardDTO getDashboard(UUID schoolId, UUID studentId) {
        // 1. Fetch student basic info
        Student student = studentRepository.findById(studentId)
            .filter(s -> s.getSchoolId().equals(schoolId))
            .orElseThrow(() -> new StudentNotFoundException(studentId));
        
        // 2. Fetch recent grades (with cache hit)
        List<Grade> recentGrades = gradeRepository.findStudentTranscript(schoolId, studentId)
            .stream()
            .limit(5)
            .toList();
        
        // 3. Fetch pending fees
        List<Fee> pendingFees = feeRepository.findPendingFees(schoolId, studentId);
        
        // 4. Fetch recent assignments
        List<Assignment> pendingAssignments = assignmentRepository.findPendingAssignments(schoolId, studentId);
        
        // 5. Calculate statistics
        BigDecimal averageScore = calculateAverageScore(recentGrades);
        long attendancePercentage = calculateAttendance(schoolId, studentId);
        
        // 6. Build response DTO
        return StudentDashboardDTO.builder()
            .student(StudentDTO.from(student))
            .recentGrades(recentGrades.stream()
                .map(GradeDTO::from)
                .toList())
            .averageScore(averageScore)
            .attendancePercentage(attendancePercentage)
            .pendingFees(pendingFees.stream()
                .map(FeeDTO::from)
                .toList())
            .totalFeesOutstanding(pendingFees.stream()
                .map(Fee::getAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add))
            .pendingAssignments(pendingAssignments.stream()
                .map(AssignmentDTO::from)
                .toList())
            .build();
    }
    
    private BigDecimal calculateAverageScore(List<Grade> grades) {
        if (grades.isEmpty()) return BigDecimal.ZERO;
        
        BigDecimal total = grades.stream()
            .map(Grade::getMarksObtained)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        return total.divide(
            BigDecimal.valueOf(grades.size()),
            2,
            RoundingMode.HALF_UP
        );
    }
    
    private long calculateAttendance(UUID schoolId, UUID studentId) {
        // Query attendance table: (PRESENT + LATE) / (PRESENT + ABSENT + LATE)
        return attendanceRepository.calculateAttendancePercentage(schoolId, studentId);
    }
}

// Fee payment service with idempotency
@Service
@RequiredArgsConstructor
@Transactional
public class FeePaymentService {
    private final FeePaymentRepository feePaymentRepository;
    private final FeeRepository feeRepository;
    private final PaymentGatewayClient paymentGatewayClient;
    private final EventPublisher eventPublisher;
    private final IdempotencyKeyRepository idempotencyKeyRepository;
    
    public FeePaymentDTO initiatePayment(
        UUID schoolId,
        UUID studentId,
        FeePaymentRequest request) {
        
        // Check for duplicate (idempotent via idempotency key)
        Optional<FeePayment> existing = idempotencyKeyRepository
            .findByKeyAndTenantId(request.getIdempotencyKey(), schoolId)
            .flatMap(ik -> feePaymentRepository.findById(ik.getPaymentId()));
        
        if (existing.isPresent()) {
            return FeePaymentDTO.from(existing.get());
        }
        
        // Fetch fee to be paid
        Fee fee = feeRepository.findByIdAndSchoolId(request.getFeeId(), schoolId)
            .orElseThrow(() -> new FeeNotFoundException(request.getFeeId()));
        
        // Call payment gateway (Stripe/Razorpay)
        PaymentInitiationResponse gatewayResponse = paymentGatewayClient.initiatePayment(
            PaymentRequest.builder()
                .amount(fee.getAmount())
                .currency("INR")
                .customerId(studentId.toString())
                .metadata(Map.of(
                    "school_id", schoolId.toString(),
                    "fee_id", fee.getId().toString()
                ))
                .build()
        );
        
        // Create payment record (PENDING status)
        FeePayment payment = FeePayment.builder()
            .id(UUID.randomUUID())
            .schoolId(schoolId)
            .feeId(fee.getId())
            .studentId(studentId)
            .amountPaid(fee.getAmount())
            .paymentGatewayRef(gatewayResponse.getPaymentIntentId())
            .status(PaymentStatus.INITIATED)
            .paymentMethod(request.getPaymentMethod())
            .build();
        
        feePaymentRepository.save(payment);
        
        // Store idempotency key
        idempotencyKeyRepository.save(IdempotencyKey.builder()
            .key(request.getIdempotencyKey())
            .tenantId(schoolId)
            .paymentId(payment.getId())
            .build());
        
        // Publish event
        eventPublisher.publishEvent(new FeePaymentInitiatedEvent(
            schoolId, payment.getId(), fee.getId()
        ));
        
        return FeePaymentDTO.from(payment);
    }
    
    @EventListener
    public void onPaymentWebhook(PaymentWebhookEvent webhook) {
        // Handle Stripe/Razorpay webhook
        FeePayment payment = feePaymentRepository
            .findByPaymentGatewayRef(webhook.getPaymentIntentId())
            .orElseThrow();
        
        if (webhook.getStatus().equals("succeeded")) {
            payment.setStatus(PaymentStatus.SUCCESS);
            payment.setTransactionId(webhook.getTransactionId());
            
            // Update fee status
            Fee fee = feeRepository.findById(payment.getFeeId()).orElseThrow();
            fee.setPaymentStatus(FeePaymentStatus.PAID);
            feeRepository.save(fee);
            
            // Publish success event
            eventPublisher.publishEvent(new FeePaymentSucceededEvent(
                payment.getSchoolId(),
                payment.getStudentId(),
                payment.getFeeId()
            ));
        } else if (webhook.getStatus().equals("failed")) {
            payment.setStatus(PaymentStatus.FAILED);
            eventPublisher.publishEvent(new FeePaymentFailedEvent(
                payment.getSchoolId(),
                payment.getStudentId(),
                webhook.getFailureReason()
            ));
        }
        
        feePaymentRepository.save(payment);
    }
}
```

### 2.5 REST API Endpoints

```java
// studentparent/api/StudentParentController.java
@RestController
@RequestMapping("/api/v1/students")
@RequiredArgsConstructor
@Validated
public class StudentParentController {
    private final DashboardService dashboardService;
    private final FeePaymentService feePaymentService;
    private final StudentRepository studentRepository;
    
    @GetMapping("/me/dashboard")
    @PreAuthorize("hasRole('STUDENT')")
    public ResponseEntity<StudentDashboardDTO> getDashboard(
        @RequestAttribute("schoolId") UUID schoolId,
        @RequestAttribute("userId") UUID userId) {
        
        StudentDashboardDTO dashboard = dashboardService.getDashboard(schoolId, userId);
        return ResponseEntity.ok(dashboard);
    }
    
    @GetMapping("/me/grades")
    @PreAuthorize("hasRole('STUDENT')")
    public ResponseEntity<Page<GradeDTO>> getGrades(
        @RequestAttribute("schoolId") UUID schoolId,
        @RequestAttribute("userId") UUID userId,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
        
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        Page<Grade> grades = gradeRepository.findAllByStudentAndSchool(
            schoolId, userId, pageable
        );
        
        return ResponseEntity.ok(grades.map(GradeDTO::from));
    }
    
    @PostMapping("/me/fees/{feeId}/pay")
    @PreAuthorize("hasRole('STUDENT') or hasRole('PARENT')")
    public ResponseEntity<FeePaymentDTO> initiatePayment(
        @RequestAttribute("schoolId") UUID schoolId,
        @RequestAttribute("userId") UUID userId,
        @PathVariable UUID feeId,
        @Valid @RequestBody FeePaymentRequest request) {
        
        request.setIdempotencyKey(
            UUID.randomUUID().toString()  // Client should provide stable key
        );
        
        FeePaymentDTO payment = feePaymentService.initiatePayment(
            schoolId, userId, request
        );
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(payment);
    }
    
    @GetMapping("/me/fees/summary")
    @PreAuthorize("hasRole('STUDENT') or hasRole('PARENT')")
    public ResponseEntity<FeesSummaryDTO> getFeesSummary(
        @RequestAttribute("schoolId") UUID schoolId,
        @RequestAttribute("userId") UUID userId) {
        
        FeesSummaryDTO summary = feeService.getFeesSummary(schoolId, userId);
        return ResponseEntity.ok(summary);
    }
    
    // Parent portal: list children
    @GetMapping("/children")
    @PreAuthorize("hasRole('PARENT')")
    public ResponseEntity<List<StudentDTO>> getChildren(
        @RequestAttribute("schoolId") UUID schoolId,
        @RequestAttribute("userId") UUID parentId) {
        
        List<Student> children = studentRepository.findChildrenOfParent(schoolId, parentId);
        return ResponseEntity.ok(children.stream().map(StudentDTO::from).toList());
    }
}
```

### 2.6 GraphQL Schema (Complementary API)

```graphql
# schema.graphql
type Student {
  id: ID!
  enrollmentNumber: String!
  firstName: String!
  lastName: String!
  email: String
  dateOfBirth: Date!
  status: StudentStatus!
  averageGrade: Float
  attendancePercentage: Int
  createdAt: DateTime!
}

type Grade {
  id: ID!
  exam: Exam!
  subject: Subject!
  marksObtained: Float!
  totalMarks: Float!
  gradeLetter: String
  gradeCategory: String!
  remarks: String
  publishedAt: DateTime!
}

type Dashboard {
  student: Student!
  recentGrades: [Grade!]!
  averageScore: Float!
  attendancePercentage: Int!
  pendingFees: [Fee!]!
  totalFeesOutstanding: Float!
  pendingAssignments: [Assignment!]!
}

input FeePaymentInput {
  feeId: ID!
  paymentMethod: PaymentMethod!
  idempotencyKey: String!
}

type Query {
  me: Student!
  dashboard: Dashboard!
  grades(first: 10, after: String): GradeConnection!
  fees(status: FeeStatus): [Fee!]!
  children: [Student!]! # Parent query
}

type Mutation {
  initiatePayment(input: FeePaymentInput!): FeePayment!
}

type Subscription {
  gradePublished: Grade!  # WebSocket: subscribe to grade updates
  feePaymentStatusChanged: FeePayment!
}
```

---

## 3. Examination Module (High-Concurrency Design)

### 3.1 Conflict Detection Algorithm

```java
// examination/service/ExamSchedulingService.java
@Service
@RequiredArgsConstructor
@Transactional
public class ExamSchedulingService {
    private final ExamScheduleRepository scheduleRepository;
    private final ConflictDetectionEngine conflictDetectionEngine;
    
    public void scheduleExam(UUID schoolId, ExamScheduleRequest request) 
        throws ConflictDetectionException {
        
        // Step 1: Detect room conflicts
        List<ExamSchedule> roomConflicts = conflictDetectionEngine.detectRoomConflicts(
            schoolId,
            request.getScheduledDate(),
            request.getStartTime(),
            request.getEndTime(),
            request.getRoomNumber()
        );
        
        if (!roomConflicts.isEmpty()) {
            throw new RoomConflictException(
                "Room " + request.getRoomNumber() + 
                " already booked for conflicting exams: " +
                roomConflicts.stream()
                    .map(e -> e.getSubject().getSubjectName())
                    .collect(Collectors.joining(", "))
            );
        }
        
        // Step 2: Detect student conflicts (same student in 2 exams at same time)
        List<ExamSchedule> studentConflicts = conflictDetectionEngine.detectStudentConflicts(
            schoolId,
            request.getScheduledDate(),
            request.getStartTime(),
            request.getEndTime(),
            request.getClassId()  // get students of this class
        );
        
        if (!studentConflicts.isEmpty()) {
            throw new StudentConflictException(
                "Students already have scheduled exams at this time"
            );
        }
        
        // Step 3: Detect teacher conflicts (same teacher can't invigilate 2 exams)
        List<ExamSchedule> teacherConflicts = conflictDetectionEngine.detectTeacherConflicts(
            schoolId,
            request.getScheduledDate(),
            request.getStartTime(),
            request.getEndTime(),
            request.getInvigilatorIds()
        );
        
        if (!teacherConflicts.isEmpty()) {
            throw new TeacherConflictException(
                "Assigned teachers have conflicting exam duties"
            );
        }
        
        // Step 4: Create schedule
        ExamSchedule schedule = new ExamSchedule();
        schedule.setId(UUID.randomUUID());
        schedule.setSchoolId(schoolId);
        schedule.setScheduledDate(request.getScheduledDate());
        schedule.setStartTime(request.getStartTime());
        schedule.setEndTime(request.getEndTime());
        schedule.setRoomNumber(request.getRoomNumber());
        
        scheduleRepository.save(schedule);
    }
}

// Conflict detection engine (optimized for large datasets)
@Component
@RequiredArgsConstructor
public class ConflictDetectionEngine {
    private final ExamScheduleRepository scheduleRepository;
    private final StudentEnrollmentRepository enrollmentRepository;
    
    public List<ExamSchedule> detectRoomConflicts(
        UUID schoolId,
        LocalDate date,
        LocalTime startTime,
        LocalTime endTime,
        String roomNumber) {
        
        // Use database-level query with time overlap logic
        return scheduleRepository.findConflictingSchedules(
            schoolId, date, startTime, endTime, roomNumber
        );
    }
    
    public List<ExamSchedule> detectStudentConflicts(
        UUID schoolId,
        LocalDate date,
        LocalTime startTime,
        LocalTime endTime,
        UUID classId) {
        
        // Get students in class
        List<UUID> studentIds = enrollmentRepository.findStudentIdsByClassId(classId);
        
        // Find existing exams for these students at same time
        return scheduleRepository.findStudentExamConflicts(
            schoolId, date, startTime, endTime, studentIds
        );
    }
}

// Database queries optimized for conflict detection
@Repository
public interface ExamScheduleRepository extends JpaRepository<ExamSchedule, UUID> {
    
    @Query("""
        SELECT es FROM ExamSchedule es
        WHERE es.schoolId = :schoolId
          AND es.scheduledDate = :date
          AND es.roomNumber = :roomNumber
          AND (
            (es.startTime < :endTime AND es.endTime > :startTime)
          )
    """)
    List<ExamSchedule> findConflictingSchedules(
        @Param("schoolId") UUID schoolId,
        @Param("date") LocalDate date,
        @Param("startTime") LocalTime startTime,
        @Param("endTime") LocalTime endTime,
        @Param("roomNumber") String roomNumber
    );
    
    @Query(value = """
        SELECT es.* FROM exam_schedules es
        WHERE es.school_id = :schoolId
          AND es.scheduled_date = :date
          AND es.start_time < :endTime
          AND es.end_time > :startTime
          AND EXISTS (
            SELECT 1 FROM enrollments e
            WHERE e.student_id = ANY(:studentIds)
              AND e.class_id IN (
                SELECT DISTINCT ec.class_id FROM enrollments ec
                WHERE ec.student_id = ANY(:studentIds)
              )
          )
    """, nativeQuery = true)
    List<ExamSchedule> findStudentExamConflicts(
        @Param("schoolId") UUID schoolId,
        @Param("date") LocalDate date,
        @Param("startTime") LocalTime startTime,
        @Param("endTime") LocalTime endTime,
        @Param("studentIds") UUID[] studentIds
    );
}
```

### 3.2 Result Publishing (High-Concurrency Pattern)

```java
// examination/service/ResultProcessingService.java
@Service
@RequiredArgsConstructor
@Transactional
public class ResultProcessingService {
    private final GradeRepository gradeRepository;
    private final ExamRepository examRepository;
    private final ResultCachingService cachingService;
    private final ApplicationEventPublisher eventPublisher;
    private final ResultPublishingTaskExecutor taskExecutor;
    private final ResultPublishingAuditLog auditLog;
    
    // Publish results for an entire exam (can be 10K+ students)
    @Async("resultProcessingExecutor")
    @Transactional
    public void publishExamResults(UUID schoolId, UUID examId) {
        
        long startTime = System.currentTimeMillis();
        
        try {
            // Step 1: Validate exam ready for publication
            Exam exam = examRepository.findById(examId)
                .orElseThrow(() -> new ExamNotFoundException(examId));
            
            if (!exam.isReadyForPublication()) {
                throw new InvalidExamStatusException("Exam not ready");
            }
            
            // Step 2: Pre-compute all grades in batches
            // (avoid loading all 10K records into memory at once)
            int batchSize = 500;
            int totalGrades = (int) gradeRepository.countByExamIdAndSchoolId(examId, schoolId);
            
            for (int offset = 0; offset < totalGrades; offset += batchSize) {
                List<Grade> gradeBatch = gradeRepository.findByExamIdWithPagination(
                    schoolId, examId, offset, batchSize
                );
                
                // Cache each grade immediately
                for (Grade grade : gradeBatch) {
                    cachingService.cacheStudentResult(
                        schoolId, examId, grade.getStudentId(), grade
                    );
                }
                
                // Publish batch events
                publishGradeBatchEvent(schoolId, examId, gradeBatch);
            }
            
            // Step 3: Mark exam as published
            exam.setPublishedAt(LocalDateTime.now());
            examRepository.save(exam);
            
            // Step 4: Publish aggregate event
            eventPublisher.publishEvent(new ExamResultsPublishedEvent(
                schoolId, examId, totalGrades
            ));
            
            long duration = System.currentTimeMillis() - startTime;
            auditLog.logSuccess(schoolId, examId, totalGrades, duration);
            
        } catch (Exception e) {
            auditLog.logFailure(schoolId, examId, e.getMessage());
            throw new ResultPublishingException(e);
        }
    }
    
    private void publishGradeBatchEvent(UUID schoolId, UUID examId, List<Grade> grades) {
        List<StudentGradePublishedEvent> events = grades.stream()
            .map(g -> new StudentGradePublishedEvent(
                schoolId, examId, g.getStudentId(),
                g.getMarksObtained(), g.getGradeLetterr, LocalDateTime.now()
            ))
            .toList();
        
        // Batch publish to Kafka (more efficient)
        events.forEach(eventPublisher::publishEvent);
    }
}

// Result caching service (99% cache hit on result day)
@Service
@RequiredArgsConstructor
public class ResultCachingService {
    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;
    
    public void cacheStudentResult(UUID schoolId, UUID examId, UUID studentId, Grade grade) {
        String cacheKey = "result:exam:" + examId + ":student:" + studentId;
        
        try {
            String gradeJson = objectMapper.writeValueAsString(grade);
            redisTemplate.opsForValue().set(
                cacheKey,
                gradeJson,
                Duration.ofDays(1)  // Expire after 1 day
            );
        } catch (JsonProcessingException e) {
            log.error("Failed to cache grade", e);
        }
    }
    
    public Optional<Grade> getStudentResult(UUID examId, UUID studentId) {
        String cacheKey = "result:exam:" + examId + ":student:" + studentId;
        String cached = redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            try {
                return Optional.of(objectMapper.readValue(cached, Grade.class));
            } catch (JsonProcessingException e) {
                log.error("Failed to deserialize cached grade", e);
            }
        }
        return Optional.empty();
    }
}

// Executor configuration for async result processing
@Configuration
public class ResultProcessingExecutorConfig {
    @Bean(name = "resultProcessingExecutor")
    public Executor resultProcessingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);           // 10 concurrent exam publications
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("result-processor-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }
}
```

### 3.3 Admit Card Generation (Batch Processing)

```java
// examination/service/AdmitCardGenerationService.java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class AdmitCardGenerationService {
    private final PdfGenerationService pdfGenerationService;
    private final FileStorageService fileStorageService;
    private final ExamScheduleRepository scheduleRepository;
    private final StudentRepository studentRepository;
    
    // Generate admit cards for all students in exam (batch PDF generation)
    public void generateAdmitCardsForExam(UUID schoolId, UUID examId) {
        
        List<ExamSchedule> schedules = scheduleRepository.findByExamId(examId);
        
        // Group by class for efficient processing
        Map<UUID, List<ExamSchedule>> schedulesByClass = schedules.stream()
            .collect(Collectors.groupingBy(ExamSchedule::getClassId));
        
        for (Map.Entry<UUID, List<ExamSchedule>> entry : schedulesByClass.entrySet()) {
            UUID classId = entry.getKey();
            List<ExamSchedule> classSchedules = entry.getValue();
            
            // Get students in class
            List<Student> students = studentRepository.findStudentsByClassId(classId);
            
            // Generate admit card PDF for each student
            for (Student student : students) {
                generateAdmitCardPdf(schoolId, student, classSchedules);
            }
        }
    }
    
    private void generateAdmitCardPdf(UUID schoolId, Student student, List<ExamSchedule> schedules) {
        // Build HTML from template
        AdmitCardData data = AdmitCardData.builder()
            .studentName(student.getFirstName() + " " + student.getLastName())
            .enrollmentNumber(student.getEnrollmentNumber())
            .schedules(schedules.stream()
                .map(s -> AdmitCardData.ScheduleInfo.builder()
                    .subject(s.getSubject().getSubjectName())
                    .date(s.getScheduledDate().format(DateTimeFormatter.ofPattern("dd-MMM-yyyy")))
                    .time(s.getStartTime().format(DateTimeFormatter.ofPattern("HH:mm")) + " - " +
                          s.getEndTime().format(DateTimeFormatter.ofPattern("HH:mm")))
                    .roomNumber(s.getRoomNumber())
                    .build())
                .toList())
            .build();
        
        String html = renderTemplate("admit-card.html", data);
        
        // Generate PDF
        byte[] pdfBytes = pdfGenerationService.generatePdfFromHtml(html);
        
        // Store in MinIO
        String filename = "admit-cards/" + schoolId + "/exam-" + 
                         student.getId() + "-" + student.getEnrollmentNumber() + ".pdf";
        fileStorageService.uploadFile(filename, pdfBytes, "application/pdf");
    }
}
```

---

## 4. Authentication & Security Layer

### 4.1 JWT Token Generation & Validation

```java
// security/service/JwtTokenProvider.java
@Service
@RequiredArgsConstructor
public class JwtTokenProvider {
    private final JwtProperties jwtProperties;
    private final UserDetailsService userDetailsService;
    
    public String generateToken(Authentication authentication) {
        UserPrincipal principal = (UserPrincipal) authentication.getPrincipal();
        
        Instant now = Instant.now();
        Instant expiryTime = now.plus(jwtProperties.getExpirationTime());
        
        return Jwts.builder()
            .subject(principal.getUsername())
            .claim("user_id", principal.getUserId().toString())
            .claim("school_id", principal.getSchoolId().toString())
            .claim("role", principal.getRole())
            .claim("permissions", principal.getPermissions())
            .claim("mfa_verified", principal.isMfaVerified())
            .claim("session_id", UUID.randomUUID().toString())
            .issuedAt(Date.from(now))
            .expiration(Date.from(expiryTime))
            .signWith(getSigningKey(), SignatureAlgorithm.HS512)
            .compact();
    }
    
    public String generateRefreshToken(UUID userId, UUID schoolId) {
        Instant now = Instant.now();
        Instant expiryTime = now.plus(jwtProperties.getRefreshExpirationTime());
        
        return Jwts.builder()
            .subject(userId.toString())
            .claim("school_id", schoolId.toString())
            .claim("token_type", "REFRESH")
            .issuedAt(Date.from(now))
            .expiration(Date.from(expiryTime))
            .signWith(getSigningKey(), SignatureAlgorithm.HS512)
            .compact();
    }
    
    public Claims validateToken(String token) throws JwtException {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
    }
    
    public UUID getUserIdFromToken(String token) {
        Claims claims = validateToken(token);
        return UUID.fromString(claims.get("user_id", String.class));
    }
    
    public UUID getSchoolIdFromToken(String token) {
        Claims claims = validateToken(token);
        return UUID.fromString(claims.get("school_id", String.class));
    }
    
    private Key getSigningKey() {
        byte[] decodedKey = Decoders.BASE64.decode(jwtProperties.getSecretKey());
        return Keys.hmacShaKeyFor(decodedKey);
    }
}

// JWT properties from application.yml
@Configuration
@ConfigurationProperties(prefix = "security.jwt")
@Data
public class JwtProperties {
    private String secretKey;
    private Duration expirationTime = Duration.ofHours(24);
    private Duration refreshExpirationTime = Duration.ofDays(7);
}
```

### 4.2 Spring Security Configuration with Tenant Context

```java
// security/config/SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {
    private final JwtTokenProvider jwtTokenProvider;
    private final TenantContextInterceptor tenantContextInterceptor;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF (stateless API)
            .csrf().disable()
            
            // Set session policy
            .sessionManagement()
                .sessionFixationProtection(SessionFixationProtectionStrategy.MIGRATE_SESSION)
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)
                .and()
            .and()
            
            // Authorization rules
            .authorizeRequests()
                .antMatchers("/auth/login", "/auth/register", "/health").permitAll()
                .antMatchers("/api/**").authenticated()
                .anyRequest().authenticated()
            .and()
            
            // Add JWT filter
            .addFilterBefore(
                jwtAuthenticationFilter(),
                UsernamePasswordAuthenticationFilter.class
            )
            
            // Security headers
            .headers()
                .contentSecurityPolicy("default-src 'self'; script-src 'self' cdn.jsdelivr.net")
                .xssProtection()
                .frameOptions().deny()
                .httpStrictTransportSecurity()
                    .includeSubDomains(true)
                    .maxAgeInSeconds(63072000);
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter(jwtTokenProvider, tenantContextInterceptor);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // Cost factor 12 (slower, more secure)
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) 
        throws Exception {
        return config.getAuthenticationManager();
    }
}

// JWT Authentication Filter
@Component
@Slf4j
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenProvider jwtTokenProvider;
    private final TenantContextInterceptor tenantContextInterceptor;
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain) throws ServletException, IOException {
        
        try {
            String jwt = getJwtFromRequest(request);
            
            if (jwt != null && jwtTokenProvider.validateToken(jwt)) {
                
                // Extract claims
                UUID userId = jwtTokenProvider.getUserIdFromToken(jwt);
                UUID schoolId = jwtTokenProvider.getSchoolIdFromToken(jwt);
                Claims claims = jwtTokenProvider.validateToken(jwt);
                
                // Set tenant context (CRITICAL for RLS)
                TenantContext.setCurrentSchool(schoolId);
                request.setAttribute("schoolId", schoolId);
                request.setAttribute("userId", userId);
                
                // Create authentication
                UserPrincipal principal = new UserPrincipal(
                    userId,
                    schoolId,
                    claims.getSubject(),
                    (String) claims.get("role"),
                    (List<String>) claims.get("permissions")
                );
                
                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        principal, null, principal.getAuthorities()
                    );
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
                
                log.debug("JWT authentication successful for user: {} in school: {}", 
                    userId, schoolId);
            }
        } catch (JwtException e) {
            log.error("JWT authentication failed: {}", e.getMessage());
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid token");
            return;
        }
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            // Clean up tenant context
            TenantContext.clear();
        }
    }
    
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### 4.3 Row-Level Security (RLS) Enforcement

```java
// security/config/TenantContext.java
@Slf4j
public class TenantContext {
    private static final ThreadLocal<UUID> schoolId = new ThreadLocal<>();
    
    public static void setCurrentSchool(UUID school) {
        if (school == null) {
            throw new IllegalArgumentException("School ID cannot be null");
        }
        schoolId.set(school);
        log.debug("Tenant context set to: {}", school);
    }
    
    public static UUID getCurrentSchool() {
        UUID school = schoolId.get();
        if (school == null) {
            throw new TenantNotSetException("Tenant context not set for current request");
        }
        return school;
    }
    
    public static void clear() {
        schoolId.remove();
    }
    
    // For testing: optional get without throwing
    public static Optional<UUID> getCurrentSchoolOptional() {
        return Optional.ofNullable(schoolId.get());
    }
}

// JPA interceptor to enforce RLS at database level
@Configuration
public class RlsConfiguration {
    
    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer() {
        return props -> {
            props.put("hibernate.enable_lazy_load_no_trans", true);
            props.put("org.hibernate.dialect.PostgreSQL94Dialect", 
                "org.hibernate.dialect.PostgreSQL10Dialect");
        };
    }
}

// PostgreSQL row-level security setup (run on migration)
@Component
@Slf4j
public class RlsInitializer {
    
    public void initializeRls(DataSource dataSource) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            
            // Enable RLS
            stmt.execute("""
                CREATE POLICY IF NOT EXISTS student_isolation ON students
                AS PERMISSIVE
                FOR ALL
                USING (school_id = current_setting('app.current_school_id')::uuid);
            """);
            
            stmt.execute("""
                ALTER TABLE students ENABLE ROW LEVEL SECURITY;
            """);
            
            // Similar for other tables
            stmt.execute("""
                ALTER TABLE grades ENABLE ROW LEVEL SECURITY;
            """);
            
            stmt.execute("""
                ALTER TABLE fees ENABLE ROW LEVEL SECURITY;
            """);
            
            log.info("RLS policies initialized successfully");
        }
    }
}

// Set PostgreSQL context variable before each database operation
@Aspect
@Component
@Slf4j
@RequiredArgsConstructor
public class TenantAwarenessAspect {
    private final EntityManager entityManager;
    
    @Before("execution(* com.schoolsaas..repository.*.*(..))")
    public void setTenantContext() {
        UUID schoolId = TenantContext.getCurrentSchoolOptional()
            .orElseThrow(() -> new TenantNotSetException("No tenant context"));
        
        // Set PostgreSQL session variable
        entityManager.createNativeQuery(
            "SET app.current_school_id = :schoolId"
        ).setParameter("schoolId", schoolId.toString())
         .executeUpdate();
        
        log.debug("Set PostgreSQL session variable: app.current_school_id = {}", schoolId);
    }
}
```

---

## 5. Data Access Layer (Query Optimization)

### 5.1 Pagination & Search

```java
// common/repository/TenantBaseRepository.java
@NoRepositoryBean
public interface TenantBaseRepository<T, ID> extends JpaRepository<T, ID> {
    
    // Base pagination with school filtering
    @Query("""
        SELECT t FROM #{#entityName} t 
        WHERE t.schoolId = :schoolId
    """)
    Page<T> findAllBySchool(
        @Param("schoolId") UUID schoolId,
        Pageable pageable
    );
}

// Advanced search with multiple criteria
@Repository
public interface StudentRepository extends TenantBaseRepository<Student, UUID> {
    
    @Query("""
        SELECT s FROM Student s
        WHERE s.schoolId = :schoolId
          AND (
            LOWER(s.firstName) LIKE LOWER(CONCAT('%', :keyword, '%'))
            OR LOWER(s.lastName) LIKE LOWER(CONCAT('%', :keyword, '%'))
            OR LOWER(s.enrollmentNumber) LIKE LOWER(CONCAT('%', :keyword, '%'))
          )
          AND s.status = :status
        ORDER BY s.firstName, s.lastName
    """)
    Page<Student> searchStudents(
        @Param("schoolId") UUID schoolId,
        @Param("keyword") String keyword,
        @Param("status") StudentStatus status,
        Pageable pageable
    );
    
    // Specialized query for analytics
    @Query("""
        SELECT new map(
            s.id as id,
            s.firstName as firstName,
            s.lastName as lastName,
            AVG(g.marksObtained) as averageScore,
            COUNT(DISTINCT a.attendanceDate) as attendanceCount
        )
        FROM Student s
        LEFT JOIN Grade g ON g.studentId = s.id
        LEFT JOIN Attendance a ON a.studentId = s.id
        WHERE s.schoolId = :schoolId
        GROUP BY s.id, s.firstName, s.lastName
        ORDER BY AVG(g.marksObtained) DESC
    """)
    Page<Map<String, Object>> getStudentAnalytics(
        @Param("schoolId") UUID schoolId,
        Pageable pageable
    );
}
```

### 5.2 N+1 Query Prevention (Fetch Strategies)

```java
// Entity eager loading specification
@Entity
@Table(name = "grades")
@Data
public class Grade {
    @Id
    private UUID id;
    
    // Use EAGER loading only when necessary
    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "exam_id")
    private Exam exam;
    
    @ManyToOne(fetch = FetchType.LAZY)  // Lazy by default
    @JoinColumn(name = "student_id")
    private Student student;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "subject_id")
    private Subject subject;
}

// Use EntityGraph to avoid N+1
@Repository
public interface GradeRepository extends JpaRepository<Grade, UUID> {
    
    @EntityGraph(attributePaths = {"exam", "student", "subject"})
    @Query("""
        SELECT g FROM Grade g
        WHERE g.schoolId = :schoolId
          AND g.examId = :examId
    """)
    List<Grade> findByExamIdWithDetails(
        @Param("schoolId") UUID schoolId,
        @Param("examId") UUID examId
    );
    
    // Alternative: use JOIN FETCH in JPQL
    @Query("""
        SELECT DISTINCT g
        FROM Grade g
        JOIN FETCH g.exam e
        JOIN FETCH g.student s
        JOIN FETCH g.subject subj
        WHERE g.schoolId = :schoolId
          AND g.examId = :examId
    """)
    List<Grade> findByExamIdWithFetch(
        @Param("schoolId") UUID schoolId,
        @Param("examId") UUID examId
    );
}
```

### 5.3 Query Performance Monitoring

```java
// performance/QueryExecutionInterceptor.java
@Component
@RequiredArgsConstructor
@Slf4j
public class QueryExecutionInterceptor implements StatementInspector {
    private final MeterRegistry meterRegistry;
    
    @Override
    public String inspect(String sql) {
        long startTime = System.nanoTime();
        
        return new String(sql) {
            @Override
            public String toString() {
                long duration = System.nanoTime() - startTime;
                
                // Log slow queries (> 1 second)
                if (duration > 1_000_000_000) {
                    log.warn("SLOW QUERY ({}ms): {}", 
                        duration / 1_000_000, 
                        sql
                    );
                }
                
                // Record metric
                meterRegistry.timer("db.query.duration")
                    .record(duration, TimeUnit.NANOSECONDS);
                
                return super.toString();
            }
        };
    }
}

// Hibernate configuration
@Configuration
public class HibernateConfiguration {
    
    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer(
        MeterRegistry meterRegistry) {
        return props -> {
            props.put("hibernate.session.events.log", true);
            props.put("hibernate.generate_statistics", true);
            props.put("hibernate.use_sql_comments", true);
        };
    }
}
```

---

## 6. Event Processing & Message Queue

### 6.1 Kafka Configuration

```yaml
# application-kafka.yml
spring:
  kafka:
    bootstrap-servers: kafka:9092
    producer:
      acks: all                    # Wait for all replicas
      retries: 3
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      linger-ms: 10               # Batch messages
      batch-size: 16384
    consumer:
      group-id: school-saas-consumer
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      max-poll-records: 500
    listener:
      ack-mode: MANUAL_IMMEDIATE  # Manual acknowledgment
      concurrency: 5

cloud:
  stream:
    kafka:
      binder:
        brokers: kafka:9092
    bindings:
      studentGradePublished:
        destination: student-grade-published
        content-type: application/json
        group: grade-processors
```

### 6.2 Event Publishing & Listening

```java
// event/EventPublishingService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class EventPublishingService {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishEvent(String topic, String key, Object payload) {
        kafkaTemplate.send(topic, key, payload)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event to {}: {}", topic, ex.getMessage());
                } else {
                    log.debug("Event published to {} with offset {}", 
                        topic, 
                        result.getRecordMetadata().offset()
                    );
                }
            });
    }
}

// Listener with error handling and retry
@Component
@RequiredArgsConstructor
@Slf4j
public class StudentGradePublishedListener {
    private final TeacherNotificationService notificationService;
    private final StudentGradePublishedListener.RetryableTask retryableTask;
    
    @KafkaListener(
        topics = "student-grade-published",
        groupId = "teacher-notifiers",
        concurrency = "5"
    )
    public void onStudentGradePublished(
        StudentGradePublishedEvent event,
        @Header(name = "kafka_receivedPartition") int partition,
        @Header(name = "kafka_offset") long offset) {
        
        try {
            log.info("Processing grade published event for student: {} in exam: {}",
                event.getStudentId(), event.getExamId());
            
            // Send notification asynchronously
            retryableTask.sendTeacherNotification(event);
            
        } catch (Exception e) {
            log.error("Error processing grade published event: {}", e.getMessage());
            // Kafka will retry based on configuration
            throw e;
        }
    }
    
    // Retry logic with exponential backoff
    @Component
    @RequiredArgsConstructor
    public static class RetryableTask {
        private final TeacherNotificationService notificationService;
        
        @Retryable(
            maxAttempts = 3,
            backoff = @Backoff(delay = 1000, multiplier = 2.0)
        )
        public void sendTeacherNotification(StudentGradePublishedEvent event) {
            notificationService.notifyTeachersOfPublishedGrades(event);
        }
        
        @Recover
        public void recover(Exception e, StudentGradePublishedEvent event) {
            log.error("Failed to send teacher notification after retries: {}", 
                e.getMessage());
            // Log to dead letter queue or send alert
        }
    }
}

// Dead letter queue handler
@Component
@RequiredArgsConstructor
@Slf4j
public class GradePublishedDltHandler {
    private final AlertingService alertingService;
    
    @DltHandler
    public void handle(StudentGradePublishedEvent event, @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage) {
        log.error("Event sent to DLT: {} - Error: {}", event, exceptionMessage);
        
        // Alert operations team
        alertingService.alert(
            "Grade publication failed for exam: " + event.getExamId(),
            AlertSeverity.HIGH
        );
    }
}
```

---

## 7. Caching Strategy

### 7.1 Multi-Level Caching

```java
// caching/CachingConfiguration.java
@Configuration
@EnableCaching
@RequiredArgsConstructor
public class CachingConfiguration {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.create(connectionFactory);
    }
}

// Tenant-aware cache decorator
@Service
@RequiredArgsConstructor
public class TenantAwareCacheService {
    private final CacheManager cacheManager;
    
    public <T> T getOrCompute(
        String cacheKey,
        Callable<T> loader,
        Duration ttl) {
        
        // Prefix cache key with school_id for multi-tenancy
        String prefixedKey = TenantContext.getCurrentSchool() + ":" + cacheKey;
        
        Cache cache = cacheManager.getCache("default");
        
        Cache.ValueWrapper value = cache.get(prefixedKey);
        if (value != null) {
            return (T) value.get();
        }
        
        try {
            T computed = loader.call();
            cache.put(prefixedKey, computed);
            return computed;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

// Cache invalidation on updates
@Service
@RequiredArgsConstructor
public class StudentServiceWithCaching {
    private final StudentRepository studentRepository;
    private final CacheManager cacheManager;
    
    @Transactional
    @CacheEvict(value = "students", key = "#schoolId + ':' + #studentId")
    public Student updateStudent(UUID schoolId, UUID studentId, StudentUpdateRequest req) {
        Student student = studentRepository.findById(studentId)
            .filter(s -> s.getSchoolId().equals(schoolId))
            .orElseThrow();
        
        student.setFirstName(req.getFirstName());
        student.setLastName(req.getLastName());
        
        return studentRepository.save(student);
    }
    
    @Cacheable(
        value = "students",
        key = "#schoolId + ':' + #studentId"
    )
    public Student getStudent(UUID schoolId, UUID studentId) {
        return studentRepository.findById(studentId)
            .filter(s -> s.getSchoolId().equals(schoolId))
            .orElseThrow();
    }
}
```

---

## 8. Error Handling & Validation

### 8.1 Global Exception Handler

```java
// error/GlobalExceptionHandler.java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(StudentNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleStudentNotFound(StudentNotFoundException e) {
        log.warn("Student not found: {}", e.getMessage());
        
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.builder()
                .code("STUDENT_NOT_FOUND")
                .message(e.getMessage())
                .timestamp(LocalDateTime.now())
                .build());
    }
    
    @ExceptionHandler(TenantNotSetException.class)
    public ResponseEntity<ErrorResponse> handleTenantNotSet(TenantNotSetException e) {
        log.error("Tenant context not set!");
        
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.builder()
                .code("TENANT_CONTEXT_ERROR")
                .message("Request processing failed - invalid tenant context")
                .timestamp(LocalDateTime.now())
                .build());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
        MethodArgumentNotValidException e) {
        
        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );
        
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse.builder()
                .code("VALIDATION_ERROR")
                .message("Input validation failed")
                .details(errors)
                .timestamp(LocalDateTime.now())
                .build());
    }
}

// Custom exceptions
public class SchoolSaasException extends RuntimeException {
    protected String code;
    protected Map<String, Object> details;
    
    public SchoolSaasException(String code, String message) {
        super(message);
        this.code = code;
    }
}

public class StudentNotFoundException extends SchoolSaasException {
    public StudentNotFoundException(UUID studentId) {
        super("STUDENT_NOT_FOUND", "Student with ID " + studentId + " not found");
    }
}

public class TenantNotSetException extends SchoolSaasException {
    public TenantNotSetException(String message) {
        super("TENANT_CONTEXT_ERROR", message);
    }
}
```

### 8.2 Input Validation

```java
// studentparent/dto/StudentCreateRequest.java
@Data
@Valid
public class StudentCreateRequest {
    
    @NotBlank(message = "Enrollment number is required")
    @Size(min = 3, max = 20, message = "Enrollment number must be 3-20 characters")
    private String enrollmentNumber;
    
    @NotBlank(message = "First name is required")
    @Size(min = 2, max = 100)
    private String firstName;
    
    @NotBlank(message = "Last name is required")
    @Size(min = 2, max = 100)
    private String lastName;
    
    @NotNull(message = "Date of birth is required")
    @Past(message = "Date of birth must be in the past")
    private LocalDate dateOfBirth;
    
    @Email(message = "Email must be valid")
    private String email;
    
    @NotNull(message = "Admission date is required")
    @PastOrPresent(message = "Admission date cannot be in the future")
    private LocalDate admissionDate;
    
    @NotNull(message = "Gender is required")
    private Gender gender;
    
    // Custom validator
    @AssertTrue(message = "Student must be at least 3 years old")
    private boolean isValidAge() {
        if (dateOfBirth == null) return true;
        return Period.between(dateOfBirth, LocalDate.now()).getYears() >= 3;
    }
}
```

---

## 9. Testing Strategy

### 9.1 Unit Tests with Testcontainers

```java
// studentparent/service/DashboardServiceTest.java
@DataJpaTest
@Import({DashboardService.class, CachingConfiguration.class})
@Testcontainers
class DashboardServiceTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private StudentRepository studentRepository;
    
    @Autowired
    private GradeRepository gradeRepository;
    
    @Autowired
    private DashboardService dashboardService;
    
    private UUID schoolId;
    private UUID studentId;
    
    @BeforeEach
    void setup() {
        schoolId = UUID.randomUUID();
        studentId = UUID.randomUUID();
        
        // Set tenant context
        TenantContext.setCurrentSchool(schoolId);
    }
    
    @AfterEach
    void cleanup() {
        TenantContext.clear();
    }
    
    @Test
    void testGetDashboard_Success() {
        // Arrange
        Student student = Student.builder()
            .id(studentId)
            .schoolId(schoolId)
            .enrollmentNumber("EMP001")
            .firstName("John")
            .lastName("Doe")
            .dateOfBirth(LocalDate.of(2010, 5, 15))
            .admissionDate(LocalDate.of(2023, 1, 1))
            .status(StudentStatus.ACTIVE)
            .build();
        
        studentRepository.save(student);
        
        // Act
        StudentDashboardDTO dashboard = dashboardService.getDashboard(schoolId, studentId);
        
        // Assert
        assertThat(dashboard).isNotNull();
        assertThat(dashboard.getStudent().getFirstName()).isEqualTo("John");
    }
    
    @Test
    void testGetDashboard_StudentNotFound() {
        // Act & Assert
        assertThrows(StudentNotFoundException.class,
            () -> dashboardService.getDashboard(schoolId, UUID.randomUUID()));
    }
}
```

### 9.2 Integration Tests

```java
// integration/StudentParentIntegrationTest.java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
@AutoConfigureTestDatabase(replace = Replace.NONE)
class StudentParentIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    private String jwtToken;
    private UUID schoolId = UUID.randomUUID();
    private UUID userId = UUID.randomUUID();
    
    @BeforeEach
    void setup() {
        // Generate test JWT token
        jwtToken = jwtTokenProvider.generateToken(
            new TestAuthentication(userId, schoolId)
        );
    }
    
    @Test
    void testGetDashboard_Authenticated() {
        // Arrange
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + jwtToken);
        
        HttpEntity<Void> entity = new HttpEntity<>(headers);
        
        // Act
        ResponseEntity<StudentDashboardDTO> response = restTemplate.exchange(
            "http://localhost:" + port + "/api/v1/students/me/dashboard",
            HttpMethod.GET,
            entity,
            StudentDashboardDTO.class
        );
        
        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

---

## 10. Performance Tuning & Optimization

### 10.1 Database Indexing Strategy

```sql
-- Indices for common queries
CREATE INDEX idx_students_school_status ON students(school_id, status);
CREATE INDEX idx_students_enrollment ON students(school_id, enrollment_number);

CREATE INDEX idx_grades_exam_student ON grades(school_id, exam_id, student_id);
CREATE INDEX idx_grades_student ON grades(school_id, student_id);
CREATE INDEX idx_grades_created ON grades(school_id, created_at DESC);

CREATE INDEX idx_fees_student ON fees(school_id, student_id, payment_status);

CREATE INDEX idx_attendance_date ON attendance(school_id, attendance_date);

-- Composite index for analytics queries
CREATE INDEX idx_student_analytics ON students(school_id, status, created_at)
  INCLUDE (first_name, last_name);
```

### 10.2 Query Result Batching

```java
// Fetch large datasets efficiently
@Service
public class BulkDataFetchService {
    private static final int BATCH_SIZE = 1000;
    
    public void processAllStudents(UUID schoolId, Consumer<List<Student>> processor) {
        int offset = 0;
        
        while (true) {
            List<Student> batch = studentRepository.findBySchoolIdBatch(
                schoolId, offset, BATCH_SIZE
            );
            
            if (batch.isEmpty()) break;
            
            processor.accept(batch);
            offset += BATCH_SIZE;
            
            // Clear persistence context to prevent OutOfMemoryError
            entityManager.clear();
        }
    }
}
```

---

## Conclusion

This LLD document provides detailed implementation specifications for all modules of the School Management SaaS platform. Key implementation patterns include:

✅ **Module isolation** via Spring Modulith with clear API boundaries  
✅ **Tenant isolation** via TenantContext + PostgreSQL RLS  
✅ **Event-driven architecture** for async, decoupled processing  
✅ **High-performance queries** with pagination, eager loading strategies  
✅ **Security-first design** with JWT, encryption, MFA  
✅ **Caching layers** for sub-200ms response times  
✅ **Error handling** with custom exceptions and global handlers  
✅ **Testing strategies** from unit to integration tests  

Implementation teams should follow these patterns closely to ensure consistency, security, and performance across the platform.

---

**Document Version:** 1.0  
**Last Updated:** December 2, 2025  
**Approved By:** Chief Software Architect