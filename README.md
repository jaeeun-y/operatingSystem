# operatingSystem


## 🚀 실험환경
Intel Core 프로세서 (6-Core / 6-Thread, 하이퍼스레딩 미지원) 기반 macOS 시스템
￼
  
터미널을 이용해 물리코어와 논리 스레드 개수 확인  
% sysctl -n hw.physicalcpu    
6  
% sysctl -n hw.logicalcpu  
6  
  
=> 하이퍼스레딩 기술이 탑재되지 않고 순수하게 물리 코어 개수 만큼만 스레드를 지원  

<img width="1368" height="666" alt="image" src="https://github.com/user-attachments/assets/e7609fbd-8222-46a0-9764-36e714fad33f" />


## 🚀용어 정리
Race Condition (경쟁 상태)
- 여러 개의 스레드(작업 단위)가 하나의 공유 자원(데이터)에 동시에 접근해서 값을 바꾸려고 경쟁하는 상황

임계구역 (Critical Section)
- Race Condition이 발생할 위험이 있는 '위험 구역'. 이 구역은 반드시 한 번에 하나의 스레드만 들어가도록 보호

동기화 기법 (Synchronization)
- 임계구역에 줄을 세우는 방법

OpenMP (Open Multi-Processing)
- C/C++ 코드에서 아주 쉽게 병렬 처리를 할 수 있게 도와주는 API(도구)

정확성-성능 간 Trade-off 
- 동기화를 빡세게,, 성능 하락
- 속도를 높이면,, 정확성 하락




## 🚀 Race Condition
실행할 때 마다 결과가 달라짐 확인

첫번째 실행  
1	No_Sync 	0.043917	20000000	SUCCESS  
2	No_Sync 	0.061566	10701723	FAIL  
4	No_Sync 	0.078409	5826032		FAIL  
8	No_Sync 	0.087039	3654938		FAIL  

두번째 실행  
1	No_Sync 	0.043201	20000000	SUCCESS  
2	No_Sync 	0.075200	10180609	FAIL  
4	No_Sync 	0.091262	5416106		FAIL  
8	No_Sync 	0.104197	4098143		FAIL  

세번째 실행  
1	No_Sync 	0.043005	20000000	SUCCESS  
2	No_Sync 	0.073870	10371070	FAIL  
4	No_Sync 	0.098256	5897646		FAIL  
8	No_Sync 	0.110861	4065693		FAIL  



## 🚀 동기화 기법

### 0. No_sync(동기화 없음)
스레드 1개일 때는 SUCCESS, 
스레드가 2개, 4개, 8개로 늘어날수록 Final_Sum 값이 줄어들며 FAIL이 뜸.

<원인> 스레드가 여러개가 되면 하나의 공유변수(sum)에 동시에 접근하는 Race Condition이 발생함.

### 1. Critical 
임계구역에 단 하나의 스레드만 들어가도록 통제.
스레드가 많아질 수록 속도가 엄청나게 느려짐.

<원인> 대기하는 시간(병목 현상)과 OS가 스레드를 멈췄다 깨우는 오버헤드가 커지기 때문

스레드가   
	  1개일 때 0.54초   
	  2개일 때 1.70초  
	  4개일 때 2.80초  
	  8개일 때 34.9초  

### 2. Atomic
Critical 처럼 스레드를 멈추지 않고 하드웨어(CPU) 수준에서 sum += 1 연산을 처리. 

<원인> 여러 스레드가 하나의 메모리 주소에 계속 신호를 보내기 때문에 스레드가 늘어나도 성능이 더 빨라지진 않음.

스레드가   
	  1개일 때 0.12초  
	  2개일 때 0.31초  
	  4개일 때 0.32초  
	  8개일 때 0.36초  

### 3. Reduction 
스레드마다 각자의 독립된 지역 변수를 채워주고 서로 간섭 없이 독립적으로 연산을 수행하게 한다.
연산이 완전히 끝난 시점에 딱 한번만 결과를 합치기 때문에 락(lock)으로 인한 대기시간이 전혀 없다.
멀티코어를 쓰면 쓸수록 속도가 정비례해서 빨라진다.

스레드가   
	  1개일 때 0.039초  
	  2개일 때 0.019초  
	  4개일 때 0.009초  
	  8개일 때 0.010초   

<원인1> Reduction은 각 스레드가 일을 다 끝낸 뒤, 마지막에 결과를 하나로 합치는 과정이 필요.  
스레드가 많아질수록 오버헤드가 늘어나 시간이 미세하게 늘어날 수 있다. (=스레드 스케줄링 경합)  

<원인2> 물리코어는 6개인데, 스레드를 8개로 지정하면 오버서브스크립션(과부하)상태가 됨.  
6개의 코어가 8개의 스레드를 번갈아가며 처리해야 하므로 문맥교환 오버헤드를 발생시켜서 시간이 늘어남.   

<결론> 문맥교환 오버헤드와 스레드 스케줄링 경합이 발생하여 병렬 처리 효율이 저하된 것으로 분석됨.  


```
## 🚀 결과

Threads	Mode	Avg_Time(s)	Final_Sum	Status
-----------------------------------------------------------------------
1	No_Sync 	0.043005	20000000	SUCCESS
1	Critical	0.547087	20000000	SUCCESS
1	Atomic  	0.119706	20000000	SUCCESS
1	Reduction	0.039359	20000000	SUCCESS
-----------------------------------------------------------------------
2	No_Sync 	0.073870	10371070	FAIL
2	Critical	1.784733	20000000	SUCCESS
2	Atomic  	0.402561	20000000	SUCCESS
2	Reduction	0.020522	20000000	SUCCESS
-----------------------------------------------------------------------
4	No_Sync 	0.098256	5897646		FAIL
4	Critical	2.963704	20000000	SUCCESS
4	Atomic  	0.384558	20000000	SUCCESS
4	Reduction	0.010248	20000000	SUCCESS
-----------------------------------------------------------------------
6	No_Sync 	0.101874	4215381		FAIL
6	Critical	4.150069	20000000	SUCCESS
6	Atomic  	0.385600	20000000	SUCCESS
6	Reduction	0.007821	20000000	SUCCESS
-----------------------------------------------------------------------
8	No_Sync 	0.110861	4065693		FAIL
8	Critical	35.051122	20000000	SUCCESS
8	Atomic  	0.390314	20000000	SUCCESS
8	Reduction	0.010080	20000000	SUCCESS
-----------------------------------------------------------------------
16	No_Sync 	0.104206	3373520		FAIL
16  Critical	93.607688	20000000	SUCCESS
16	Atomic  	0.386277	20000000	SUCCESS
16	Reduction	0.008012	20000000	SUCCESS
-----------------------------------------------------------------------



[실험 결과 요약 표]
스레드 수		No_Sync (동기화 없음)	Critical (크리티컬 섹션)	Atomic (아토믹)		Reduction (리덕션)
1			0.043005초 (SUCCESS)	0.547087초 (SUCCESS)		0.119706초 (SUCCESS)	0.039359초 (SUCCESS)
2			0.073870초 (FAIL)	1.784733초 (SUCCESS)		0.402561초 (SUCCESS)	0.020522초 (SUCCESS)
4			0.098256초 (FAIL)	2.963704초 (SUCCESS)		0.384558초 (SUCCESS)	0.010248초 (SUCCESS)
6 			0.101874초 (FAIL)	4.150069초 (SUCCESS)		0.385600초 (SUCCESS)	0.007821초 (최고 속도)
8 (과부하)	0.110861초 (FAIL)	35.051122초 (SUCCESS)	0.390314초 (SUCCESS)	0.010080초 (속도 저하)
16(과부하)	0.104206초 (FAIL)	93.607688초 (SUCCESS)	0.386277초 (SUCCESS)	0.008012초 (속도 정체)
```



```
## 🚀소스코드

#include <omp.h>
#include <stdio.h>
#include <stdlib.h>

// 연산 횟수 (2천만 번)
#define N 20000000 

// 1. Race Condition (동기화 없음)
long long run_no_sync() {
    long long sum = 0;
    #pragma omp parallel for
    for (int i = 1; i <= N; i++) {
        sum += 1; 
    }
    return sum;
}

// 2. OpenMP critical 기법
long long run_critical() {
    long long sum = 0;
    #pragma omp parallel for
    for (int i = 1; i <= N; i++) {
        #pragma omp critical
        sum += 1;
    }
    return sum;
}

// 3. OpenMP atomic 기법
long long run_atomic() {
    long long sum = 0;
    #pragma omp parallel for
    for (int i = 1; i <= N; i++) {
        #pragma omp atomic
        sum += 1;
    }
    return sum;
}

// 4. OpenMP reduction 기법
long long run_reduction() {
    long long sum = 0;
    #pragma omp parallel for reduction(+:sum)
    for (int i = 1; i <= N; i++) {
        sum += 1;
    }
    return sum;
}

int main() {
    int thread_counts[] = {1, 2, 4, 6, 8, 16}; 
    int num_configs = sizeof(thread_counts) / sizeof(int);

    // 엑셀 복사용 탭(\t) 구분 출력
    printf("Threads\tMode\t\tAvg_Time(s)\tFinal_Sum\tStatus\n");
    printf("-----------------------------------------------------------------------\n");

    for (int t = 0; t < num_configs; t++) {
        int current_threads = thread_counts[t];
        
        // 코어에서 실행될 스레드 수 명시적 지정
        omp_set_num_threads(current_threads); 

        for (int mode = 0; mode < 4; mode++) {
            double total_time = 0.0;
            long long final_sum = 0;

            // 5회 반복 실행 평균값 계산
            for (int rep = 0; rep < 5; rep++) {
                double start = omp_get_wtime();

                if (mode == 0) final_sum = run_no_sync();
                else if (mode == 1) final_sum = run_critical();
                else if (mode == 2) final_sum = run_atomic();
                else if (mode == 3) final_sum = run_reduction();

                double end = omp_get_wtime();
                total_time += (end - start);
            }

            double avg_time = total_time / 5.0;
            
            // 결과 검증 정답 판단 (N = 20000000)
            const char* status = (final_sum == N) ? "SUCCESS" : "FAIL";
            
            const char* mode_name = "";
            if (mode == 0) mode_name = "No_Sync ";
            if (mode == 1) mode_name = "Critical";
            if (mode == 2) mode_name = "Atomic  ";
            if (mode == 3) mode_name = "Reduction";

            printf("%d\t%s\t%.6f\t%lld\t%s\n", current_threads, mode_name, avg_time, final_sum, status);
        }
        printf("-----------------------------------------------------------------------\n");
    }
    return 0;
}
```
