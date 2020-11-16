# COVID-19-Spreading-Model-Parallel-Computing
## 1.Introduction
### 1.1 Purpose
The project is required to design, deploy, develop, and then experiment with a massive parallel system. It applies the CPU with the MPI model, Hybrid CPU/GPU with MPI and CUDA model to compare the performance and make the analysis. The end goal is to make progress towards a parallel computing performance design and analysis that is of high enough quality to be published.
### 1.2 Scope
COVID-19, a world-level coronavirus disease, is now spreading at a staggering rate. Billions of data of COVID-19 is available to predict or measure the coronavirus disease. Using high-performance parallel computing to deal with the substantial volume data in a supercomputer is the topic of the project. The project plans to create a new infection model as the theme, applying the CPU, GPU, and MPI technologies to test the performance of paralleling computing in different circumstances, including various MPI ranking and GPU nodes. Set benchmark to make comparisons of performance based on running time and cycle count. Analyze the algorithms and reasons behind the different performances.
### 1.3 Model
It create several 2-D matrices as different states. Every matric represents a single state. The cells inside the state are the residents. For a person at a cell, it has 4 conditions which are represented by an int: value 0 means infected with the COVID-19 infection; value 1 means healthy one; value 2 represents recovered from the COVID-19infection and values 3 implies died due to COVID-19 infection. For every person in every iteration, scaning the eight nearby person conditions and applying the spreading rule to determine the value of the central person. For every state shares the same boundary with another, using ghost lines to mimic this situation so that every state can get the information in the boundary of a nearby one. When passing the states to GPU, transferring 2-D to 1-D and layer them as one long row and using the mathematic rules to make offset the index problem.<br>
For e.g.
Initially, 6*6 world for NY State can be:<br>
100101 <br>
001101 <br>
100000 <br>
001111 <br>
100101 <br>
100101 
### 1.4 Spreading Rule
After iterations, there will be four conditions for every cell： <br>
value 0: the guy is infected <br>
value 1: the guy is still healthy, good luck <br>
value 2: the guy recovers from the infection <br>
value 3: unfortunately, the guy died <br>
Generally, a person has 8 neighbors. if the person is healthy, every infected neighbor increased 12.5% infection possibility for the man, which means if all his neighbors are infected, the person will definitely be infected, even mask can't help.  If the person is already infected, due to the high medical technology, he'll have 95% to recovery, otherwise, unfortunately, he'll die.
<br>
## 2.Parallel Algorithm
Apply CUDA, MPI, and MPI I/O. <br> 
For CUDA, apply memories at GPU by cudaMallocManaged function. A long row then store the data in every state one by one in order so that there should be a 1-D array in GPU memory. So it can use parallel computation by using multiple blocks and nodes at the same time. <br>
For MPI, every state is a rank. To share the same boundary, use ghost columns at the left and right of the state. So different rank passes the ghost columns to the next rank; people at the next state can scan surrounding people and determine the next stage. The thing is in our assumption, the first (the most left) and the last (the most right) states don’t share the boundary, which means the states are not a loop. I think this assumption is realistic: people who live in the east won’t infect the people who live in the west if they don’t move at the social distance circumstance. <br>
For MPI I/O, first, use MPI_File_open to open the files contains state infection information. Then create a vast shared space to store the states' information and assign space for every rank; in other words, every state. Next, use MPI_File_read to read information from the files. Then create a similar huge shared space to store the result and assign space for every rank as we read. To output the result to file we get after iterations, call MPI_File_write to write them into the external file and finish the whole MPI I/O process. In other words, read from several files including the state matrix and write to one shared file for various ranks as one huge matrix which combines results for all states.
<br>
## 3.Hybrid CPU/GPU via MPI
The files are c and cu files, and they'll  be compiled by mpicc and nvcc together. The MPI ranks and I/O ideas are the same as the above group. But after extracting the data from reading vast space, data are passed into GPU memory while converting it to the 1-D array. Then the CUDA part will parallel compute the result via iterations. Run this group in AiMOS supercomputer system to make sure there are enough GPUs. It's able to test this by changing the amounts of nodes and GPUs.
<br>
## 4. Comparison
### 4.1 Strong Scaling Performance
For both two groups(CPU via MPI and Hybrid CPU/GPU via MPI), test the strong scaling performance. In this case, the problem size, which is the size of states remains the same and CPU/GPU node increases. Use three world sizes: 32*32, 128*128 and 1024*1024. In the testing environment, every node can contain maximum 6 GPUs and it's able to apply at most 2 nodes every time, which means there will be 1 to 12 nodes when we run the program. In strong scaling performance experiment, the program is considered to scale linearly if the speedup, here use running time and cycle count to represent is equal to the number of processing nodes used. Keep the 256*256 as world size and change the node amounts as 1, 2, 3, 4.
### 4.2 Weak Scaling Performance
Similar with strong scaling test, for both two groups, test the weak scaling performance. In this case, the problem size, which is size of states increases the same and CPU/GPU node increases. In weak scaling performance experiments, the program speedup remains the same while problem size and the number of processing nodes used are linearly increased at the same time.
<br>
## 5.Files
### 5.1 gol-main.c
Basic MPI rank code, controls rank numbers. <br>
MPI I/O functions, including MPI parallelly reading from and writing to different files. Transpose the matrix read in and set the initial value for the 2-D matrix world.<br>
A timer that calculates the project total running time as the clue for comparison. <br>
Code call the kernel functions, which are defined in gol-with-cuda.cu file. 
### 5.2 gol-with-cuda.cu
Basic GPU memory allocation and deallocation. Initialize the 1-D world(converted from 2-D world). <br>
Kernel function: caculate number of each cell's neighbors, judge the condition for next iteration according to the conditions of itself and its neighbors(Specific rules are in 1.4 Spreading Rule).  <br>
Function sets the number of blocks and threads to use for every test. <br>
Function controls the two worlds, one is for current iteration, one is the draft and it'll be the next iteration. Swap them between the period of every successful iteration. <br>
### 5.3 sov-without-cuda.c
Technically, this is the version of gol-main.c without calling GPU kernel function. This is to test and compare as CPU via MPI group for the efficiency. It contains only MPI rank I/O, model build and execute calculation in CPU rather than GPU.
### 5.4 slurmSpecturm.sh
Configuration set file. It clears 5 parameters to pass in the model.<br>
First: MPI rank size, which is also the amount of states <br>
Second: World size for every state, like 6, so the world will be a 6*6 2-D matrix <br>
Third: The amount of threads will be used for the test <br>
Fourth: The amount of iterations will be used for the test <br> 
Fifth: this is a flag indicates wether write the the last iteration of the world(which is also the result) to the outside files or not. 1 indicates write to outside files and others means not write <br>
### 5.5 makefile
Includes all the commands needed to compile all the files and execute the program in the bash environment. Type in make in the bash all these commands would been executed.
### 5.6 parallel project test case1.xlsx:
Data of all the test cases.
### 5.7 g1.xlsx:
Data for making graphs.
### 5.8 generate_number.c
This is to generate the random world. The only argument is the world size.




