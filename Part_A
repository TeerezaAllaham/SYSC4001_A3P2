#include <stdio.h>      // for printf, fopen, fgets, fclose
#include <stdlib.h>     // for exit, atoi, rand, srand
#include <string.h>     // for memset, strncpy, strlen, strchr
#include <unistd.h>     // for fork, usleep, getpid
#include <sys/ipc.h>    // shared memory key creation
#include <sys/shm.h>    // shared memory functions
#include <sys/types.h>  // for pid_t, system types
#include <sys/wait.h>   // for wait
#include <time.h>       // for time
#include <errno.h>      // for error handling

// some constants
#define MAX_RUBRIC_LINES 5      // makes sure there are only 5 lines in rubric
#define MAX_LINE_LEN     128    // max length of each rubric line
#define MAX_EXAMS        64     // max number of exam files
#define STUDENT_LEN      16     // max length of student number string

// states for question marking
#define Q_UNTOUCHED     0       // question not yet picked
#define Q_PROGRESSING   1       // question being marked by TA
#define Q_CORRECTED     2       // question marking done

// some global variables
static char rubric_path[256];       // path to rubric file
static char exam_files[MAX_EXAMS][256];  // paths to exam files 
static int  num_exams = 0;      // number of exam files

// shared data structure
typedef struct {
    char rubric[MAX_RUBRIC_LINES][MAX_LINE_LEN];
    char current_student[STUDENT_LEN];  // ex : "1024"
    int  question_state[MAX_RUBRIC_LINES]; // question marking state
    int  current_exam_index;    // index of current exam being processed
    int  terminate;             // flag to signal TAs to exit when student ID 9999 is reached
} SharedData;

// deals with sleeping for a random time between min_ms and max_ms milliseconds 
static void sleep_ms(int min_ms, int max_ms)
{
    int range = max_ms - min_ms + 1;
    int ms = min_ms + (rand() % range);
    usleep(ms * 1000);  // convert to microseconds
}

// load rubric from file into shared memory
static int load_rubric(const char *path, SharedData *sh)
{
    FILE *f = fopen(path, "r");
    if (!f) {
        perror("fopen rubric");
        return -1;
    }

    // Read exactly 5 lines from rubric file
    for (int i = 0; i < MAX_RUBRIC_LINES; i++) {
        if (!fgets(sh->rubric[i], MAX_LINE_LEN, f)) {
            // If fewer lines, set remaining to empty
            sh->rubric[i][0] = '\0';
        } else {
            // strip newline
            size_t len = strlen(sh->rubric[i]);
            if (len > 0 && sh->rubric[i][len - 1] == '\n') {
                sh->rubric[i][len - 1] = '\0';
            }
        }
    }

    fclose(f);
    return 0;
}

static int save_rubric(const char *path, SharedData *sh)
{
    FILE *f = fopen(path, "w");
    if (!f) {
        perror("fopen rubric for write");
        return -1;
    }

    // Write all 5 lines back to rubric file
    for (int i = 0; i < MAX_RUBRIC_LINES; i++) {
        fprintf(f, "%s\n", sh->rubric[i]);
    }

    fclose(f);
    return 0;
}

// load exam file into shared memory
static int load_exam(SharedData *sh, int exam_index)
{
    // Write all 5 lines back
    if (exam_index < 0 || exam_index >= num_exams) {
        fprintf(stderr, "No more exams (index %d)\n", exam_index);
        sh->terminate = 1;
        return -1;
    }

    const char *path = exam_files[exam_index];
    FILE *f = fopen(path, "r");
    if (!f) {
        perror("fopen exam");
        sh->terminate = 1;
        return -1;
    }

    // Read first line → student number
    char buf[STUDENT_LEN];
    if (!fgets(buf, sizeof(buf), f)) {
        fprintf(stderr, "Exam file %s is empty\n", path);
        fclose(f);
        sh->terminate = 1;
        return -1;
    }

    // strip newline
    size_t len = strlen(buf);
    if (len > 0 && buf[len - 1] == '\n') {
        buf[len - 1] = '\0';
    }

    // Copy student number
    strncpy(sh->current_student, buf, STUDENT_LEN - 1);
    sh->current_student[STUDENT_LEN - 1] = '\0';

    //  reset question states
    for (int i = 0; i < MAX_RUBRIC_LINES; i++) {
        sh->question_state[i] = Q_UNTOUCHED;
    }

    printf("[PARENT] Loaded exam %d (%s) student %s into shared memory.\n", exam_index, path, sh->current_student);

    // Check for sentinel student ID 9999
    if (atoi(sh->current_student) == 9999) {
        printf("[PARENT] Sentinel student 9999 reached. TAs will exit.\n");
        sh->terminate = 1;
    }

    fclose(f);
    return 0;
}

// TA process function
static void ta(int id, SharedData *sh)
{
    // Ensure each TA has different random seed
    srand(getpid());

    printf("[TA %d, PID %d] Started.\n", id, getpid());

    // TA runs until terminate flag is raised
    while (1) {
        if (sh->terminate) {
            printf("[TA %d] Terminate flag set. Exiting.\n", id);
            break;
        }

        // check and maybe correct rubric
        printf("[TA %d] Checking rubric for student %s (exam %d).\n",
               id, sh->current_student, sh->current_exam_index);

        for (int q = 0; q < MAX_RUBRIC_LINES; q++) {
            char *line = sh->rubric[q];
            if (line[0] == '\0') continue;  // skip empty lines

            printf("[TA %d] Reviewing rubric line %d: '%s'\n", id, q + 1, line);

            // Simulate review time 0.5–1.0 seconds
            sleep_ms(500, 1000);

            // Randomly decide to correct (25% chance)
            if (rand() % 4 == 0) {
                char *comma = strchr(line, ',');
                if (comma && comma[1] != '\0') {
                    char *c = &comma[1];
                    // Increment the score by 1 to shift ascii value
                    (*c)++;
                    printf("[TA %d] Corrected rubric line %d -> '%s'\n", id, q + 1, line);
                }
            }
        }

        // Save rubric back to file
        printf("[TA %d] Writing rubric back to file: %s\n", id, rubric_path);
        save_rubric(rubric_path, sh);

        // mark questions
        int all_done = 0;
        while (!all_done && !sh->terminate) {
            all_done = 1;

            // pick a question to mark
            int picked_q = -1;
            for (int q = 0; q < MAX_RUBRIC_LINES; q++) {
                if (sh->question_state[q] == Q_UNTOUCHED) {
                    picked_q = q;
                    sh->question_state[q] = Q_PROGRESSING; 
                    all_done = 0;
                    break;
                }
                if (sh->question_state[q] != Q_CORRECTED) {
                    all_done = 0;
                }
            }

            if (picked_q == -1) {
                // no more questions to mark
                if (all_done) {
                    printf("[TA %d] All questions appear done for student %s.\n", id, sh->current_student);
                }
                break;
            }

            // simulate marking time 1.0–2.0 seconds
            printf("[TA %d] Marking student %s question %d...\n", id, sh->current_student, picked_q + 1);
            sleep_ms(1000, 2000);

            // question marked
            sh->question_state[picked_q] = Q_CORRECTED;
            printf("[TA %d] Finished marking student %s question %d.\n", id, sh->current_student, picked_q + 1);
        }

        if (sh->terminate) {
            printf("[TA %d] Terminate flag set after marking. Exiting.\n", id);
            break;
        }

        // try to load next exam if all questions done
        if (all_done && !sh->terminate) {
            int next_exam = sh->current_exam_index + 1;
            printf("[TA %d] Attempting to load next exam index %d.\n",
                   id, next_exam);

            if (next_exam >= num_exams) {
                printf("[TA %d] No more exams listed. Setting terminate.\n", id);
                sh->terminate = 1;
                break;
            }

            sh->current_exam_index = next_exam;
            load_exam(sh, next_exam);

            if (sh->terminate) {
                printf("[TA %d] Terminate flag set after loading exam. Exiting.\n",
                       id);
                break;
            }
        }
    }

    printf("[TA %d, PID %d] Finished.\n", id, getpid());
}

// main function 
int main(int argc, char *argv[])
{
    // Expect at least: program, numTAs, rubric, exam1
    if (argc < 4) {
        fprintf(stderr,
                "Usage: %s <num_TAs>=2 rubric.txt exam1.txt exam2.txt ...\n",
                argv[0]);
        return EXIT_FAILURE;
    }

    int num_TAs = atoi(argv[1]);
    if (num_TAs < 2) {
        fprintf(stderr, "num_TAs must be >= 2.\n");
        return EXIT_FAILURE;
    }

    // Save rubric path
    strncpy(rubric_path, argv[2], sizeof(rubric_path) - 1);
    rubric_path[sizeof(rubric_path) - 1] = '\0';

    num_exams = argc - 3;
    if (num_exams > MAX_EXAMS) {
        fprintf(stderr, "Too many exams; max is %d\n", MAX_EXAMS);
        return EXIT_FAILURE;
    }

    for (int i = 0; i < num_exams; i++) {
        strncpy(exam_files[i], argv[3 + i], sizeof(exam_files[i]) - 1);
        exam_files[i][sizeof(exam_files[i]) - 1] = '\0';
    }

    // create shared memory
    int shmid = shmget(IPC_PRIVATE, sizeof(SharedData), IPC_CREAT | 0666);
    if (shmid < 0) {
        perror("shmget");
        return EXIT_FAILURE;
    }

    SharedData *sh = (SharedData *)shmat(shmid, NULL, 0);
    if (sh == (void *)-1) {
        perror("shmat");
        shmctl(shmid, IPC_RMID, NULL);
        return EXIT_FAILURE;
    }

    // initialize shared memory
    memset(sh, 0, sizeof(SharedData));
    sh->current_exam_index = 0;
    sh->terminate = 0;

    if (load_rubric(rubric_path, sh) != 0) {
        fprintf(stderr, "Failed to load rubric.\n");
        shmdt(sh);
        shmctl(shmid, IPC_RMID, NULL);
        return EXIT_FAILURE;
    }

    if (load_exam(sh, 0) != 0) {
        fprintf(stderr, "Failed to load first exam.\n");
        shmdt(sh);
        shmctl(shmid, IPC_RMID, NULL);
        return EXIT_FAILURE;
    }

    // fork TA processes
    for (int i = 0; i < num_TAs; i++) {
        pid_t pid = fork();
        if (pid < 0) {
            perror("fork");
        } else if (pid == 0) {
            // child process
            SharedData *child_sh = (SharedData *)shmat(shmid, NULL, 0);
            if (child_sh == (void *)-1) {
                perror("shmat child");
                exit(EXIT_FAILURE);
            }
            ta(i, child_sh);
            shmdt(child_sh);
            exit(EXIT_SUCCESS);
        }
        // parent continues loop
    }

    //parent wait for all TAs to finish
    for (int i = 0; i < num_TAs; i++) {
        wait(NULL);
    }

    // cleanup shared memory
    shmdt(sh);
    shmctl(shmid, IPC_RMID, NULL);

    printf("[PARENT] All TAs finished. Cleanup done.\n");
    return EXIT_SUCCESS;
}
