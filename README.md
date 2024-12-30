# MySQL Student Management System - Comprehensive Guide

## Table of Contents
1. Database Setup & Structure
2. Basic Operations
3. Intermediate Operations
4. Advanced Operations
5. Database Administration
6. Performance Optimization
7. Security Measures
8. Best Practices

## 1. Database Setup & Structure

### Create Database
```sql
-- Create database with specific character set
CREATE DATABASE student_management
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

USE student_management;
```

### Create Tables
```sql
-- Students table with detailed information
CREATE TABLE students (
    student_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    date_of_birth DATE,
    gender ENUM('M', 'F', 'Other'),
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    address TEXT,
    emergency_contact VARCHAR(100),
    registration_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('Active', 'Inactive', 'Suspended') DEFAULT 'Active',
    CONSTRAINT chk_email CHECK (email LIKE '%@%.%')
);

-- Courses table with comprehensive details
CREATE TABLE courses (
    course_id INT PRIMARY KEY AUTO_INCREMENT,
    course_name VARCHAR(100) NOT NULL,
    course_code VARCHAR(20) UNIQUE,
    credits INT,
    department VARCHAR(50),
    description TEXT,
    max_capacity INT,
    instructor_name VARCHAR(100),
    start_date DATE,
    end_date DATE,
    CONSTRAINT chk_credits CHECK (credits > 0)
);

-- Enrollments table with grades
CREATE TABLE enrollments (
    enrollment_id INT PRIMARY KEY AUTO_INCREMENT,
    student_id INT,
    course_id INT,
    enrollment_date DATE,
    grade DECIMAL(5,2),
    status ENUM('Enrolled', 'Completed', 'Withdrawn') DEFAULT 'Enrolled',
    FOREIGN KEY (student_id) REFERENCES students(student_id) 
        ON DELETE CASCADE ON UPDATE CASCADE,
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    CONSTRAINT chk_grade CHECK (grade >= 0 AND grade <= 100)
);

-- Attendance tracking table
CREATE TABLE attendance (
    attendance_id INT PRIMARY KEY AUTO_INCREMENT,
    student_id INT,
    course_id INT,
    attendance_date DATE,
    status ENUM('Present', 'Absent', 'Late') DEFAULT 'Present',
    remarks TEXT,
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

## 2. Basic Operations

### Insert Operations

```sql
-- Insert multiple students
INSERT INTO students 
(first_name, last_name, date_of_birth, gender, email, phone) 
VALUES 
('John', 'Doe', '2000-05-15', 'M', 'john.doe@email.com', '123-456-7890'),
('Jane', 'Smith', '2001-03-20', 'F', 'jane.smith@email.com', '123-456-7891'),
('Bob', 'Johnson', '1999-11-08', 'M', 'bob.j@email.com', '123-456-7892');

-- Insert course with full details
INSERT INTO courses 
(course_name, course_code, credits, department, description, max_capacity, instructor_name, start_date, end_date)
VALUES 
('Introduction to Programming', 'CS101', 3, 'Computer Science', 
'Basic programming concepts using Python', 30, 'Dr. Smith', '2024-01-15', '2024-05-15'),
('Database Management', 'CS202', 4, 'Computer Science',
'Introduction to SQL and database design', 25, 'Dr. Jones', '2024-01-15', '2024-05-15'),
('Web Development', 'CS303', 3, 'Computer Science',
'HTML, CSS, and JavaScript fundamentals', 28, 'Prof. Wilson', '2024-01-15', '2024-05-15');

-- Enroll students in courses
INSERT INTO enrollments 
(student_id, course_id, enrollment_date)
VALUES
(1, 1, CURRENT_DATE()),
(1, 2, CURRENT_DATE()),
(2, 1, CURRENT_DATE()),
(3, 3, CURRENT_DATE());

-- Record attendance
INSERT INTO attendance 
(student_id, course_id, attendance_date, status, remarks)
VALUES
(1, 1, CURRENT_DATE(), 'Present', 'Participated actively'),
(2, 1, CURRENT_DATE(), 'Late', 'Arrived 10 minutes late'),
(3, 3, CURRENT_DATE(), 'Absent', 'Informed in advance');
```

### Select Operations

```sql
-- Basic student information
SELECT 
    CONCAT(first_name, ' ', last_name) as full_name,
    email,
    TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) as age
FROM students
WHERE status = 'Active'
ORDER BY last_name, first_name;

-- Course enrollment statistics
SELECT 
    c.course_code,
    c.course_name,
    COUNT(e.student_id) as enrolled_students,
    c.max_capacity,
    (c.max_capacity - COUNT(e.student_id)) as available_seats,
    ROUND((COUNT(e.student_id) / c.max_capacity * 100), 2) as fill_percentage
FROM courses c
LEFT JOIN enrollments e ON c.course_id = e.course_id
GROUP BY c.course_id
ORDER BY fill_percentage DESC;

-- Student attendance report
SELECT 
    s.first_name,
    s.last_name,
    c.course_code,
    COUNT(CASE WHEN a.status = 'Present' THEN 1 END) as days_present,
    COUNT(CASE WHEN a.status = 'Absent' THEN 1 END) as days_absent,
    COUNT(CASE WHEN a.status = 'Late' THEN 1 END) as days_late,
    ROUND((COUNT(CASE WHEN a.status = 'Present' THEN 1 END) * 100.0 / COUNT(*)), 2) as attendance_percentage
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
LEFT JOIN attendance a ON s.student_id = a.student_id AND c.course_id = a.course_id
GROUP BY s.student_id, c.course_id
HAVING attendance_percentage < 75
ORDER BY attendance_percentage;
```

## 3. Intermediate Operations

### Update Operations

```sql
-- Update student status based on attendance
UPDATE students s
SET status = 'Suspended'
WHERE student_id IN (
    SELECT DISTINCT student_id
    FROM attendance
    GROUP BY student_id
    HAVING COUNT(CASE WHEN status = 'Absent' THEN 1 END) > 10
);

-- Update course capacity and adjust enrollments
BEGIN TRANSACTION;

UPDATE courses 
SET max_capacity = max_capacity - 5
WHERE course_code = 'CS101';

-- Remove excess enrollments based on last-enrolled-first-out
DELETE FROM enrollments 
WHERE course_id = (SELECT course_id FROM courses WHERE course_code = 'CS101')
AND enrollment_id IN (
    SELECT enrollment_id
    FROM (
        SELECT enrollment_id
        FROM enrollments
        WHERE course_id = (SELECT course_id FROM courses WHERE course_code = 'CS101')
        ORDER BY enrollment_date DESC
        LIMIT 5
    ) as e
);

COMMIT;

-- Update grades based on attendance
UPDATE enrollments e
SET grade = (
    SELECT 
        CASE 
            WHEN attendance_percent >= 90 THEN grade + 5
            WHEN attendance_percent < 75 THEN grade - 10
            ELSE grade
        END
    FROM (
        SELECT 
            COUNT(CASE WHEN status = 'Present' THEN 1 END) * 100.0 / COUNT(*) as attendance_percent
        FROM attendance a
        WHERE a.student_id = e.student_id 
        AND a.course_id = e.course_id
    ) as att
)
WHERE EXISTS (
    SELECT 1 
    FROM attendance a 
    WHERE a.student_id = e.student_id 
    AND a.course_id = e.course_id
);
```

### Complex Queries

```sql
-- Find students at risk (poor attendance and grades)
SELECT 
    s.student_id,
    CONCAT(s.first_name, ' ', s.last_name) as student_name,
    c.course_code,
    e.grade,
    ROUND(att.attendance_percentage, 2) as attendance_percentage,
    CASE 
        WHEN e.grade < 60 AND att.attendance_percentage < 75 THEN 'Critical'
        WHEN e.grade < 70 OR att.attendance_percentage < 80 THEN 'Warning'
        ELSE 'Good'
    END as risk_status
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
JOIN (
    SELECT 
        student_id,
        course_id,
        COUNT(CASE WHEN status = 'Present' THEN 1 END) * 100.0 / COUNT(*) as attendance_percentage
    FROM attendance
    GROUP BY student_id, course_id
) att ON s.student_id = att.student_id AND c.course_id = att.course_id
WHERE e.grade < 70 OR att.attendance_percentage < 80
ORDER BY risk_status, e.grade;

-- Course performance analysis
SELECT 
    c.course_code,
    c.course_name,
    COUNT(DISTINCT e.student_id) as total_students,
    ROUND(AVG(e.grade), 2) as average_grade,
    ROUND(MIN(e.grade), 2) as lowest_grade,
    ROUND(MAX(e.grade), 2) as highest_grade,
    ROUND(
        SUM(CASE WHEN e.grade >= 90 THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) as a_grade_percentage,
    ROUND(
        SUM(CASE WHEN e.grade < 60 THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) as failing_percentage
FROM courses c
LEFT JOIN enrollments e ON c.course_id = e.course_id
GROUP BY c.course_id
HAVING total_students > 0
ORDER BY average_grade DESC;
```

## 4. Advanced Operations

### Stored Procedures

```sql
-- Procedure to enroll student with validation
DELIMITER //

CREATE PROCEDURE enroll_student(
    IN p_student_id INT,
    IN p_course_code VARCHAR(20),
    OUT p_status VARCHAR(100)
)
BEGIN
    DECLARE v_course_id INT;
    DECLARE v_current_enrolled INT;
    DECLARE v_max_capacity INT;
    DECLARE v_student_status VARCHAR(20);
    
    -- Get student status
    SELECT status INTO v_student_status
    FROM students
    WHERE student_id = p_student_id;
    
    -- Check if student is active
    IF v_student_status != 'Active' THEN
        SET p_status = 'Error: Student is not active';
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Student must be active to enroll';
    END IF;
    
    -- Get course details
    SELECT 
        course_id, 
        max_capacity 
    INTO 
        v_course_id, 
        v_max_capacity
    FROM courses 
    WHERE course_code = p_course_code;
    
    -- Check current enrollment
    SELECT COUNT(*) INTO v_current_enrolled
    FROM enrollments
    WHERE course_id = v_course_id;
    
    -- Validate capacity
    IF v_current_enrolled >= v_max_capacity THEN
        SET p_status = 'Error: Course is full';
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Course has reached maximum capacity';
    END IF;
    
    -- Enroll student
    INSERT INTO enrollments (student_id, course_id, enrollment_date)
    VALUES (p_student_id, v_course_id, CURRENT_DATE());
    
    SET p_status = 'Success: Student enrolled';
    
END //

DELIMITER ;

-- Procedure to generate student report
DELIMITER //

CREATE PROCEDURE generate_student_report(
    IN p_student_id INT
)
BEGIN
    -- Student details
    SELECT 
        CONCAT(s.first_name, ' ', s.last_name) as student_name,
        s.email,
        s.status,
        COUNT(DISTINCT e.course_id) as enrolled_courses,
        ROUND(AVG(e.grade), 2) as average_grade
    FROM students s
    LEFT JOIN enrollments e ON s.student_id = e.student_id
    WHERE s.student_id = p_student_id
    GROUP BY s.student_id;
    
    -- Course details and grades
    SELECT 
        c.course_code,
        c.course_name,
        e.grade,
        ROUND(
            (SELECT COUNT(CASE WHEN status = 'Present' THEN 1 END) * 100.0 / COUNT(*)
             FROM attendance 
             WHERE student_id = p_student_id AND course_id = c.course_id)
        , 2) as attendance_percentage
    FROM enrollments e
    JOIN courses c ON e.course_id = c.course_id
    WHERE e.student_id = p_student_id;
    
    -- Attendance details
    SELECT 
        course_code,
        COUNT(CASE WHEN a.status = 'Present' THEN 1 END) as days_present,
        COUNT(CASE WHEN a.status = 'Absent' THEN 1 END) as days_absent,
        COUNT(CASE WHEN a.status = 'Late' THEN 1 END) as days_late
    FROM attendance a
    JOIN courses c ON a.course_id = c.course_id
    WHERE a.student_id = p_student_id
    GROUP BY a.course_id;
END //

DELIMITER ;
```

### Triggers

```sql
-- Trigger to update course status when max capacity is reached
DELIMITER //

CREATE TRIGGER check_course_capacity
AFTER INSERT ON enrollments
FOR EACH ROW
BEGIN
    DECLARE v_current_count INT;
    DECLARE v_max_capacity INT;
    
    -- Get current enrollment count
    SELECT COUNT(*), c.max_capacity 
    INTO v_current_count, v_max_capacity
    FROM enrollments e
    JOIN courses c ON e.course_id = c.course_id
    WHERE e.course_id = NEW.course_id
    GROUP BY e.course_id;
    
    -- Update course status if full
    IF v_current_count >= v_max_capacity THEN
        UPDATE courses 
        SET status = 'Full' 
        WHERE course_id = NEW.course_id;
    END IF;
END //

-- Trigger to maintain attendance records when student is deleted
CREATE TRIGGER archive_student_records
BEFORE DELETE ON students
FOR EACH ROW
BEGIN
    INSERT INTO archived_students
    SELECT *, CURRENT_TIMESTAMP, USER()
    FROM students
    WHERE student_id = OLD.student_id;
    
    INSERT INTO archived_attendance
    SELECT *, CURRENT_TIMESTAMP, USER()
    FROM attendance
    WHERE student_id = OLD.student_id;
END //

DELIMITER ;
```

## 5. Database Administration

### Backup and Restore
