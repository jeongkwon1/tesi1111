#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/wait.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <time.h>
#define MATRIX_DIMENSION_XY 1000
//SEARCH FOR TODO
//

// sets one element of the matrix
void set_matrix_elem(float *M,int x,int y,float f)
{
    M[x + y*MATRIX_DIMENSION_XY] = f;
    return;
}
//

// lets see it both are the same
int quadratic_matrix_compare(float *A,float *B)
{
    for(int a = 0;a<MATRIX_DIMENSION_XY;a++)
        for(int b = 0;b<MATRIX_DIMENSION_XY;b++)
            if(A[a +b * MATRIX_DIMENSION_XY]!=B[a +b * MATRIX_DIMENSION_XY]) 
                return 0;
   
    return 1;
}   
//

//print a matrix
void quadratic_matrix_print(float *C)
{
    printf("\n");
    for(int a = 0;a<MATRIX_DIMENSION_XY;a++)
        {
            printf("\n");
            for(int b = 0;b<MATRIX_DIMENSION_XY;b++)
                printf("%.2f,",C[a + b* MATRIX_DIMENSION_XY]);
        }
    printf("\n");
}
//

// multiply two matrices
void quadratic_matrix_multiplication(float *A,float *B,float *C)
{
    //nullify the result matrix first
    for(int a = 0;a<MATRIX_DIMENSION_XY;a++)
        for(int b = 0;b<MATRIX_DIMENSION_XY;b++)
            C[a + b*MATRIX_DIMENSION_XY] = 0.0;
    //multiply
    for(int a = 0;a<MATRIX_DIMENSION_XY;a++) // over all cols a
        for(int b = 0;b<MATRIX_DIMENSION_XY;b++) // over all rows b
            for(int c = 0;c<MATRIX_DIMENSION_XY;c++) // over all rows/cols left
                {
                    C[a + b*MATRIX_DIMENSION_XY] += A[c + b*MATRIX_DIMENSION_XY] * B[a 
                    + c*MATRIX_DIMENSION_XY]; 
                }
}

void synch(int par_id,int par_count,int *ready)
{
    //TODO: synch algorithm. make sure, ALL processes get stuck here until all ARE here
    int newVal = ++ready[par_id];
    
    for(int i = 0; i < par_count; i++)
    {
        while(ready[i] < newVal);
    }
}

void quadratic_matrix_multiplication_parallel(int par_id, int par_count,float* A,float * B,float* C, int* ready)
{
    if (par_id == 0)
    {
        //nullify the result matrix first
        for(int a = 0;a<MATRIX_DIMENSION_XY;a++)
            for(int b = 0;b<MATRIX_DIMENSION_XY;b++)
                C[a + b*MATRIX_DIMENSION_XY] = 0.0;
    }
    synch(par_id, par_count, ready);
    int rows;
    if(MATRIX_DIMENSION_XY % par_count == 0)
    {
        rows = MATRIX_DIMENSION_XY/par_count;
        //multiply
        for(int a = 0; a<MATRIX_DIMENSION_XY; a++) // over all cols a
            for(int b = (par_id * rows); b<(par_id * rows) + rows; b++) // over all rows b
                for(int c = 0;c<MATRIX_DIMENSION_XY;c++) // over all rows/cols left
                    {
                        C[a + b*MATRIX_DIMENSION_XY] += A[c + b*MATRIX_DIMENSION_XY] * B[a 
                        + c*MATRIX_DIMENSION_XY]; 
                    }
    }
    else
    {
        rows = MATRIX_DIMENSION_XY/par_count;
        if(par_id == 0)
        {
            rows += MATRIX_DIMENSION_XY % par_count;
            //multiply
            for(int a = 0; a<MATRIX_DIMENSION_XY; a++) // over all cols a
                for(int b = 0; b < MATRIX_DIMENSION_XY/par_count + (MATRIX_DIMENSION_XY % par_count); b++) // over all rows b
                    for(int c = 0;c<MATRIX_DIMENSION_XY;c++) // over all rows/cols left
                        {
                            C[a + b*MATRIX_DIMENSION_XY] += A[c + b*MATRIX_DIMENSION_XY] * B[a 
                            + c*MATRIX_DIMENSION_XY]; 
                        }
        }
        else
        {
            //multiply
            for(int a = 0; a<MATRIX_DIMENSION_XY; a++) // over all cols a
                for(int b = (par_id * rows) + (MATRIX_DIMENSION_XY % par_count); b<((par_id * rows) + rows + (MATRIX_DIMENSION_XY % par_count)); b++) // over all rows b
                    for(int c = 0;c<MATRIX_DIMENSION_XY;c++) // over all rows/cols left
                        {
                            C[a + b*MATRIX_DIMENSION_XY] += A[c + b*MATRIX_DIMENSION_XY] * B[a 
                            + c*MATRIX_DIMENSION_XY]; 
                        }
        }
    }

    
}
//


//

int main(int argc, char *argv[])
{
    int par_id = 0; // the parallel ID of this process
    int par_count = 1; // the amount of processes
    float *A,*B,*C; //matrices A,B and C
    int *ready; //needed for synch
    float *time;
    time_t start;
    time_t end;
    if(argc!=3){printf("no shared\n");}
    else
        {
            par_id= atoi(argv[1]);
            par_count= atoi(argv[2]);
        // strcpy(shared_mem_matrix,argv[3]);
        }
    if(par_count==1){printf("only one process\n");}
    if(par_count > 10)
    {
        if(par_id == 0)
        {
            perror("Too many programs try a value between 1-10");
            return 0;
        }
        exit(EXIT_FAILURE);
    }
    
    int fd[5];
    if(par_id==0)
        {
            //TODO: init the shared memory for A,B,C, ready. shm_open with O_CREAT here! then ftruncate! then mmap
            fd[0] = shm_open("matrixA", O_RDWR | O_CREAT, 0777);
            fd[1] = shm_open("matrixB", O_RDWR | O_CREAT, 0777);
            fd[2] = shm_open("matrixC", O_RDWR | O_CREAT, 0777);
            fd[3] = shm_open("Ready", O_RDWR | O_CREAT, 0777);
            fd[4] = shm_open ("Timer", O_RDWR | O_CREAT, 0777);
            ftruncate(fd[0], MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float));
            ftruncate(fd[1], MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float));
            ftruncate(fd[2], MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float));
            ftruncate(fd[3], sizeof(int) * par_count);
            ftruncate(fd[4], sizeof(float));
            A = (float*)mmap(NULL, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[0], 0);
            B = (float*)mmap(NULL, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[1], 0);
            C = (float*)mmap(NULL, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[2], 0);
            ready = (int*)mmap(NULL, sizeof(int) * par_count, PROT_READ | PROT_WRITE, MAP_SHARED, fd[3], 0);
            time = (float*)mmap(NULL, sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[4], 0);
            *time = 0;
            for(int i = 0; i < par_count; i++)
            {
                ready[i] = 0;
            }
        }
    else
        {
            //TODO: init the shared memory for A,B,C, ready. shm_open withOUT O_CREAT here! NO ftruncate! but yes to mmap
            sleep(2); //needed for initalizing synch
            fd[0] = shm_open("matrixA", O_RDWR, 0777);
            fd[1] = shm_open("matrixB", O_RDWR, 0777);
            fd[2] = shm_open("matrixC", O_RDWR, 0777);
            fd[3] = shm_open("Ready", O_RDWR, 0777);
            fd[4] = shm_open ("Timer", O_RDWR | O_CREAT, 0777);
            A = (float*)mmap(NULL, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[0], 0);
            B = (float*)mmap(NULL, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[1], 0);
            C = (float*)mmap(NULL, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[2], 0);
            ready = (int*)mmap(NULL, sizeof(int) * par_count, PROT_READ | PROT_WRITE, MAP_SHARED, fd[3], 0);
            time = (float*)mmap(NULL, sizeof(float), PROT_READ | PROT_WRITE, MAP_SHARED, fd[4], 0);
        }
    synch(par_id,par_count,ready);
    if(par_id == 0)
        {
            for(int i = 0; i < (MATRIX_DIMENSION_XY*MATRIX_DIMENSION_XY); i++)
            {
                A[i] = 1;
                B[i] = 1;
            }
        }
    synch(par_id,par_count,ready);
    start = clock();
    quadratic_matrix_multiplication_parallel(par_id, par_count,A,B,C, ready);
    end = clock(); //added
    *time += (end - start); //added
    synch(par_id,par_count,ready);
    if (par_id == (par_count - 1))
    {
        printf("Matrix Multiplication %d*%d= %f seconds\n",MATRIX_DIMENSION_XY,MATRIX_DIMENSION_XY, (*time)/CLOCKS_PER_SEC);
    }
    *time=0;

    start = clock();
    quadratic_matrix_multiplication_parallel(par_id, par_count,A,B,C, ready);
    end = clock(); //added
    *time += (end - start); //added
    synch(par_id,par_count,ready);
    if (par_id == (par_count - 1))
    {
        printf("Matrix Multiplication %d*%d= %f seconds\n",MATRIX_DIMENSION_XY,MATRIX_DIMENSION_XY, (*time)/CLOCKS_PER_SEC);
    }
    *time=0;
    start = clock();
    quadratic_matrix_multiplication_parallel(par_id, par_count,A,B,C, ready);
    end = clock(); //added
    *time += (end - start); //added
    synch(par_id,par_count,ready);
    if (par_id == (par_count - 1))
    {
        printf("Matrix Multiplication %d*%d= %f seconds\n",MATRIX_DIMENSION_XY,MATRIX_DIMENSION_XY, (*time)/CLOCKS_PER_SEC);
    }
    *time=0;
    start = clock();
    quadratic_matrix_multiplication_parallel(par_id, par_count,A,B,C, ready);
    end = clock(); //added
    *time += (end - start); //added
    synch(par_id,par_count,ready);
    if (par_id == (par_count - 1))
    {
        printf("Matrix Multiplication %d*%d= %f seconds\n",MATRIX_DIMENSION_XY,MATRIX_DIMENSION_XY, (*time)/CLOCKS_PER_SEC);
    }
    *time=0;
    start = clock();
    quadratic_matrix_multiplication_parallel(par_id, par_count,A,B,C, ready);
    end = clock(); //added
    *time += (end - start); //added
    synch(par_id,par_count,ready);
    if (par_id == (par_count - 1))
    {
        printf("Matrix Multiplication %d*%d= %f seconds\n",MATRIX_DIMENSION_XY,MATRIX_DIMENSION_XY, (*time)/CLOCKS_PER_SEC);
    }
   
    synch(par_id, par_count, ready);

    //lets test the result:
    float M[MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY];
    quadratic_matrix_multiplication(A, B, M);
    synch(par_id, par_count, ready); // added
    if (quadratic_matrix_compare(C, M))
        printf("full points!\n");
    else
        printf("buuug!\n");
    close(fd[0]);
    close(fd[1]);
    close(fd[2]);
    close(fd[3]);
    close(fd[4]);
    shm_unlink("matrixA");
    shm_unlink("matrixB");
    shm_unlink("matrixC");
    shm_unlink("Ready"); // changed
    shm_unlink("Timer"); // added
    munmap(A, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float)); // added
    munmap(B, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float)); // added
    munmap(C, MATRIX_DIMENSION_XY * MATRIX_DIMENSION_XY * sizeof(float)); // added
    munmap(ready, sizeof(int) * par_count); // added
    munmap(time, sizeof(float)); //added
    return 0;    
}
