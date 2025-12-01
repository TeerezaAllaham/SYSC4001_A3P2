# SYSC4001_A3P2
Teereza Allaham (101289630)  + Sahil Todeti (101259541) 

Part 2 of this assignment implements a multi process TA marking system using system V shared memory (part A) and system V semaphores for synchronization (part B).

Multiple TA processes concurrently:
- Load and “correct” a shared rubric.
- Mark questions for a student exam.
- Advance to the next exam.
- Terminate when the student (9999) is reached.

**How to run Part A:**
gcc -o Part_A Part_A.c
./Part_A 2 rubric.txt exams/exam1 exams/exam2 exams/exam20
or
./Part_A 2 rubric.txt exams/exam*

**How to run Part B:**
gcc -o Part_B Part_B.c
./Part_B 2 Part_A_rubric.txt exams/exam1 exams/exam2 exams/exam20
or
./Part_B 2 rubric.txt exams/exam*

*2 = number of TAs... can be any number you want the amount of TAs to be. 
For this assignment it was told to start a program with 2 TAs.

*exam20 holds student 9999

note: I apologize that the exam files are not labeled as txt. I made the error and it's time consuming to fix. Although it still works perfectly so it's okay.
