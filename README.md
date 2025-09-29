Query to create tables
 Table 1: Students
CREATE TABLE students (
    student_id NUMBER PRIMARY KEY,
    first_name VARCHAR2(50),
    last_name VARCHAR2(50),
    department VARCHAR2(100),
    enrollment_year NUMBER
);

Table 2: Books
CREATE TABLE books (
    book_id NUMBER PRIMARY KEY,
    title VARCHAR2(200),
    author VARCHAR2(100),
    subject_area VARCHAR2(100),
    publication_year NUMBER,
    total_copies NUMBER
);

Table 3: Borrowing Transactions
CREATE TABLE borrowing_transactions (
    transaction_id NUMBER PRIMARY KEY,
    student_id NUMBER REFERENCES students(student_id),
    book_id NUMBER REFERENCES books(book_id),
    borrow_date DATE,
    return_date DATE,
    semester VARCHAR2(20)
);


Query to Insert items in tables

Insert in students
INSERT INTO students VALUES (1, 'Alice', 'Mukamana', 'Computer Science', 2023);
INSERT INTO students VALUES (2, 'Bob', 'Hakizimana', 'Business', 2023);
INSERT INTO students VALUES (3, 'Claire', 'Uwase', 'Engineering', 2024);
INSERT INTO students VALUES (4, 'David', 'Ndizeye', 'Computer Science', 2024);
INSERT INTO students VALUES (5, 'Emma', 'Iradukunda', 'Business', 2023);

Insert in  books
INSERT INTO books VALUES (101, 'Database Systems', 'Dr. Smith', 'Computer Science', 2022, 5);
INSERT INTO books VALUES (102, 'Business Analytics', 'Prof. Johnson', 'Business', 2021, 3);
INSERT INTO books VALUES (103, 'Advanced Programming', 'Dr. Brown', 'Computer Science', 2023, 4);
INSERT INTO books VALUES (104, 'Financial Management', 'Dr. Wilson', 'Business', 2020, 2);
INSERT INTO books VALUES (105, 'Electrical Circuits', 'Prof. Davis', 'Engineering', 2022, 3);

Insert in borrowing transactions
INSERT INTO borrowing_transactions VALUES (1001, 1, 101, DATE '2024-01-15', DATE '2024-02-01', 'Spring2024');
INSERT INTO borrowing_transactions VALUES (1002, 2, 102, DATE '2024-01-20', DATE '2024-02-10', 'Spring2024');
INSERT INTO borrowing_transactions VALUES (1003, 1, 103, DATE '2024-02-01', DATE '2024-02-20', 'Spring2024');
INSERT INTO borrowing_transactions VALUES (1004, 3, 105, DATE '2024-01-25', DATE '2024-02-15', 'Spring2024');
INSERT INTO borrowing_transactions VALUES (1005, 4, 101, DATE '2024-02-10', DATE '2024-03-01', 'Spring2024');
INSERT INTO borrowing_transactions VALUES (1006, 2, 104, DATE '2024-02-15', DATE '2024-03-05', 'Spring2024');
INSERT INTO borrowing_transactions VALUES (1007, 5, 102, DATE '2024-03-01', DATE '2024-03-20', 'Spring2024');
INSERT INTO borrowing_transactions VALUES (1008, 1, 101, DATE '2024-03-05', NULL, 'Spring2024');
COMMIT;

Query for ranking function

Top 5 most borrowed books by department with different ranking methods
SELECT 
    b.subject_area,
    b.title,
    COUNT(bt.transaction_id) as borrow_count,
    ROW_NUMBER() OVER (PARTITION BY b.subject_area ORDER BY COUNT(bt.transaction_id) DESC) as row_num,
    RANK() OVER (PARTITION BY b.subject_area ORDER BY COUNT(bt.transaction_id) DESC) as rank_pos,
    DENSE_RANK() OVER (PARTITION BY b.subject_area ORDER BY COUNT(bt.transaction_id) DESC) as dense_rank_pos,
    PERCENT_RANK() OVER (PARTITION BY b.subject_area ORDER BY COUNT(bt.transaction_id) DESC) as percent_rank
FROM books b
JOIN borrowing_transactions bt ON b.book_id = bt.book_id
GROUP BY b.subject_area, b.title
QUALIFY ROW_NUMBER() OVER (PARTITION BY b.subject_area ORDER BY COUNT(bt.transaction_id) DESC) <= 5;

Query for aggregate function

Running monthly borrowing totals and 3-month moving average
SELECT 
    TO_CHAR(borrow_date, 'YYYY-MM') as borrow_month,
    COUNT(transaction_id) as monthly_borrows,
    SUM(COUNT(transaction_id)) OVER (
        ORDER BY TO_CHAR(borrow_date, 'YYYY-MM') 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as running_total,
    AVG(COUNT(transaction_id)) OVER (
        ORDER BY TO_CHAR(borrow_date, 'YYYY-MM')
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg_3months
FROM borrowing_transactions
GROUP BY TO_CHAR(borrow_date, 'YYYY-MM')
ORDER BY borrow_month;

Query for Navigation function

Month-over-month borrowing growth analysis
WITH monthly_stats AS (
    SELECT 
        TO_CHAR(borrow_date, 'YYYY-MM') as month,
        COUNT(transaction_id) as borrow_count
    FROM borrowing_transactions
    GROUP BY TO_CHAR(borrow_date, 'YYYY-MM')
)
SELECT 
    month,
    borrow_count,
    LAG(borrow_count, 1) OVER (ORDER BY month) as previous_month,
    borrow_count - LAG(borrow_count, 1) OVER (ORDER BY month) as monthly_change,
    ROUND(
        ((borrow_count - LAG(borrow_count, 1) OVER (ORDER BY month)) / 
        LAG(borrow_count, 1) OVER (ORDER BY month)) * 100, 2
    ) as growth_percentage,
    LEAD(borrow_count, 1) OVER (ORDER BY month) as next_month
FROM monthly_stats
ORDER BY month;

Query for distribution function

Student segmentation by borrowing frequency using NTILE and CUME_DIST
WITH student_borrow_stats AS (
    SELECT 
        s.student_id,
        s.first_name || ' ' || s.last_name as student_name,
        s.department,
        COUNT(bt.transaction_id) as total_borrows,
        NTILE(4) OVER (ORDER BY COUNT(bt.transaction_id) DESC) as borrowing_quartile,
        CUME_DIST() OVER (ORDER BY COUNT(bt.transaction_id)) as cumulative_distribution
    FROM students s
    LEFT JOIN borrowing_transactions bt ON s.student_id = bt.student_id
    GROUP BY s.student_id, s.first_name, s.last_name, s.department
)
SELECT 
    student_id,
    student_name,
    department,
    total_borrows,
    borrowing_quartile,
    CASE borrowing_quartile
        WHEN 1 THEN 'Heavy Reader'
        WHEN 2 THEN 'Regular Reader' 
        WHEN 3 THEN 'Occasional Reader'
        WHEN 4 THEN 'Light Reader'
    END as reader_segment,
    ROUND(cumulative_distribution * 100, 2) as percentile
FROM student_borrow_stats
ORDER BY total_borrows DESC;
