# demo-spring-batch

## Quick Start
1. [build.gradle](build.gradle)
```
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    testImplementation 'org.springframework.batch:spring-batch-test'
}
```
2. [Application.java](src/main/java/com/example/demospringbatch/DemoSpringBatchApplication.java)
```
@EnableBatchProcessing
@SpringBootApplication
public class DemoSpringBatchApplication {...}
```
3. [JobConfiguration.java](src/main/java/com/example/demospringbatch/job/SimpleJobConfiguration.java)
```
@RequiredArgsConstructor
@Configuration
public class SimpleJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
	private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job simpleJob() {
        return jobBuilderFactory.get("simpleJob")
                .start(simpleStep1())
                .build();
    }

    @Bean
    public Step simpleStep1() {
        return stepBuilderFactory.get("simpleStep1")
                .tasklet((contribution, chunkContext) -> {
                   
                    return RepeatStatus.FINISHED;
                })
                .build();
    }
}
```

## Meta-Data Schema
- [docs](https://docs.spring.io/spring-batch/docs/current/reference/html/schema-appendix.html#metaDataSchema)

- 스프링 배치를 사용하기 위해선 배치 메타데이터를 담을 이미 정해진 구조의 데이터베이스 테이블들이 존재해야 한다.  
이 테이블의 스키마를 제공해주고 있기 때문에 `schema-mysql.sql` 파일을 찾아 데이터베이스에서 실행시켜 테이블을 생성한다.

- Meta-Data 종류
    - BATCH_JOB_INSTANCE
        - 작업 정보
        - Job Parameter에 따라 새로운 row가 추가된다. (Job Parameter에 따라 새로운 작업 인스턴스다.)
        
    - BATCH_JOB_EXECUTION
        - 작업 실행 기록
        - `STATUS` 컬럼 : 실행 결과. [BatchStatus](https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/core/BatchStatus.html) 값.
        - `EXIT_CODE` 컬럼 : 실행 종료 코드. [ExitStatus](https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/core/ExitStatus.html) 값.
        - COMPLETED인 경우 Job Instance 작업은 재수행되지 않는다.
        
    - BATCH_JOB_EXECUTION_PARAMS
        - 작업 실행시 입력받은 Job Parameter 기록


## Step 순서 제어
- Step : 실제 작업을 수행하는 역할
- [docs](https://docs.spring.io/spring-batch/docs/current/reference/html/step.html#configureStep)

### 순차적 실행
    - 참고: [StepNextJobConfiguration.java](src/main/java/com/example/demospringbatch/job/StepNextJobConfiguration.java)
        ```
        @Bean
        public Job stepNextJob() {
            return jobBuilderFactory.get("stepNextJob")
                    .start(step1())
                    .next(step2())
                    .next(step3())
                    .build();
        }
        ```
    - 앞 step에서 에러가 발생하면 뒤 step들은 실행되지 않는다.
    
### 조건별 실행 
#### ExitStatus에 따른 순서 제어 
- 참고: [FlowBuilder](https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/core/job/builder/FlowBuilder.html)
- 참고: [StepNextConditionalJobConfiguration.java](src/main/java/com/example/demospringbatch/job/StepNextConditionalJobConfiguration.java)

- 나만의 exitCode  
    BatchStatus는 Job 또는 Step의 실행 결과이고, ExitStatus는 Step의 실행 후 상태이다. 
    BatchStatus는 enum이지만 ExitStatus는 객체로 커스텀 할 수가 있다.
    ```
    public class CustomStepExecutionListener extends StepExecutionListenerSupport {
        public ExitStatus afterStep(StepExecution stepExecution) {
            // ...
            return new ExitStatus("CUSTOM EXIT CODE");
        }
    }
    ```
    ```
    @Bean
    public Job stepNextConditionalJob() {
        return jobBuilderFactory.get("stepNextConditionalJob")
              .start(conditionalJobStep1())
                  .on("FAILED")
                  .end()
              .from(conditionalJobStep1())
                  .on("CUSTOM EXIT CODE")
                  .to(conditionalJobStep2())
                  .end()
              .build();
    }
    ```
- Step이 실제 작업 로직 외에 분기 처리까지 담당해야 한다는 단점이 있다.

#### Decider에 따른 순서 제어
- 참고: [FlowExecutionStatus](https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/core/job/flow/FlowExecutionStatus.html)
- 참고: [DeciderJobConfiguration.java](src/main/java/com/example/demospringbatch/job/DeciderJobConfiguration.java)

- 나만의 exitCode  
    ```
    @Bean
    public JobExecutionDecider decider() {
        return new OddDecider();
    }

    public static class OddDecider implements JobExecutionDecider {

        @Override
        public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
            Random rand = new Random();

            int randomNumber = rand.nextInt(50) + 1;
            log.info("랜덤숫자: {}", randomNumber);

            if(randomNumber % 2 == 0) {
                return new FlowExecutionStatus("EVEN");
            } else {
                return new FlowExecutionStatus("ODD");
            }
        }
    }
    ```
    ```
    @Bean
    public Job deciderJob() {
        return jobBuilderFactory.get("deciderJob")
                .start(startStep())
                .next(decider())
                .from(decider())
                    .on("ODD")
                    .to(oddStep())
                .end()
                .build();
    }
    ```
- 분기 처리는 Decider에게 맡기고 Step은 배치 로직에 집중할 수 있다.
 
  
## 실행 설정
- 지정한 배치 작업만 실행되도록 하기  
    1. [application.yml](src/main/resources/application.yml)  
        ```
        spring.batch.job.names: ${job.name:NONE}
        ```
    2. 프로그램 실행 변수 설정  
        ```
        java -jar demo-spring-batch.jar --job.name=SomeJobName
        ```
       
## 출처 및 참고
- https://docs.spring.io/spring-batch/docs/current/reference/html/index.html
- https://jojoldu.tistory.com/324?category=902551