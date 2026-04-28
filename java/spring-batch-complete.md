# 배치 처리 완전 정복 — Spring Batch 개념부터 성능 최적화까지

## 1. 배치(Batch)란?

배치 처리는 **대량의 데이터를 일괄적으로 처리**하는 방식. 실시간 요청/응답 방식(온라인 처리)과 달리, 정해진 시간에 묶어서 처리함.

### 온라인 처리 vs 배치 처리

| 구분 | 온라인 처리 | 배치 처리 |
| --- | --- | --- |
| 처리 시점 | 요청 즉시 | 예약된 시간(야간, 월말 등) |
| 데이터 규모 | 소량 (건당) | 대량 (수만~수억 건) |
| 응답 속도 | 빠름 (ms~s) | 상관없음 (분~시간) |
| 사용자 상호작용 | 있음 | 없음 (무인 처리) |
| 실패 처리 | 즉각 에러 반환 | 재시도, 스킵, 롤백 전략 필요 |

### 배치가 필요한 실무 시나리오

- **정산 처리**: 매일 자정 전날 주문/결제 데이터를 집계해 정산 테이블 생성
- **알림 발송**: 자정에 만료 예정 쿠폰 보유 회원 추출 → 푸시/이메일 발송
- **데이터 마이그레이션**: 레거시 DB → 신규 DB 대량 이관
- **통계 집계**: 시간별/일별/월별 통계 테이블 갱신
- **파일 처리**: 외부 시스템에서 수신한 CSV/TXT 파일 파싱 후 DB 적재
- **데이터 정제**: 휴면 회원 전환, 만료 데이터 삭제, 상태 업데이트

---

## 2. Spring Batch 핵심 아키텍처

Spring Batch는 엔터프라이즈 수준의 배치 처리를 위한 경량 프레임워크.

```
┌─────────────────────────────────────────────────────────┐
│                    JobLauncher                           │
│                        │                                 │
│                      Job ◀── JobParameters               │
│                        │                                 │
│         ┌──────────────┼──────────────┐                 │
│       Step 1         Step 2         Step 3               │
│         │                                                │
│   ┌─────┴──────┐                                        │
│   │  ItemReader │  → 읽기                               │
│   │ItemProcessor│  → 가공                               │
│   │  ItemWriter │  → 쓰기                               │
│   └────────────┘                                        │
│         │                                                │
│    chunk-oriented processing                             │
│    (commit-interval 단위로 트랜잭션)                     │
└─────────────────────────────────────────────────────────┘
```

### 핵심 구성 요소

**Job**: 배치 작업의 최상위 단위. 여러 Step으로 구성.

**Step**: 실제 처리 단위. Chunk 방식 또는 Tasklet 방식으로 구현.

**Chunk 방식**: `ItemReader → ItemProcessor → ItemWriter` 를 chunk 단위로 반복.

**Tasklet 방식**: 단순한 단일 작업(파일 삭제, 알림 발송 등)에 사용.

**JobRepository**: 배치 실행 메타데이터(실행 이력, 상태, 파라미터)를 DB에 저장.

**JobLauncher**: Job을 실행시키는 인터페이스.

---

## 3. 기본 구조 — 의존성 및 설정

### build.gradle

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    runtimeOnly 'com.mysql:mysql-connector-j'
    testImplementation 'org.springframework.batch:spring-batch-test'
}
```

### application.yml

```yaml
spring:
  batch:
    job:
      enabled: false          # 앱 시작 시 자동 실행 비활성화
    jdbc:
      initialize-schema: always   # 메타 테이블 자동 생성 (운영은 never)
  datasource:
    url: jdbc:mysql://localhost:3306/batchdb
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
```

### BatchConfig — 기본 뼈대

```java
@Configuration
@EnableBatchProcessing
@RequiredArgsConstructor
public class BatchConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final DataSource dataSource;

    // Job, Step 빈 등록은 이 클래스 또는 별도 JobConfig 클래스에서 관리
}
```

---

## 4. Chunk 기반 처리 — 실전 구현

### 시나리오: 주문 데이터 일별 정산 처리

하루치 주문을 읽어 → 정산 금액 계산 → 정산 테이블에 저장.

### 엔티티 정의

```java
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long userId;
    private Long productId;
    private Integer amount;         // 주문 금액
    private String status;          // COMPLETED, CANCELLED
    private LocalDate orderDate;
}

@Entity
@Table(name = "settlement")
@Getter @Setter
@NoArgsConstructor
public class Settlement {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long userId;
    private Integer totalAmount;    // 정산 금액
    private LocalDate settlementDate;
    private LocalDateTime createdAt;
}
```

### OrderSettlementJobConfig — Reader / Processor / Writer

```java
@Configuration
@RequiredArgsConstructor
public class OrderSettlementJobConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final EntityManagerFactory entityManagerFactory;
    private final SettlementRepository settlementRepository;

    private static final int CHUNK_SIZE = 1000;

    @Bean
    public Job orderSettlementJob() {
        return new JobBuilder("orderSettlementJob", jobRepository)
                .start(orderSettlementStep())
                .build();
    }

    @Bean
    public Step orderSettlementStep() {
        return new StepBuilder("orderSettlementStep", jobRepository)
                .<Order, Settlement>chunk(CHUNK_SIZE, transactionManager)
                .reader(orderItemReader(null))
                .processor(orderItemProcessor())
                .writer(settlementItemWriter())
                .faultTolerant()
                .skip(Exception.class)
                .skipLimit(10)
                .retry(DataAccessException.class)
                .retryLimit(3)
                .build();
    }

    @Bean
    @StepScope
    public JpaPagingItemReader<Order> orderItemReader(
            @Value("#{jobParameters['targetDate']}") String targetDate) {

        LocalDate date = LocalDate.parse(targetDate);

        Map<String, Object> params = new HashMap<>();
        params.put("orderDate", date);
        params.put("status", "COMPLETED");

        return new JpaPagingItemReaderBuilder<Order>()
                .name("orderItemReader")
                .entityManagerFactory(entityManagerFactory)
                .pageSize(CHUNK_SIZE)
                .queryString(
                    "SELECT o FROM Order o "
                    + "WHERE o.orderDate = :orderDate "
                    + "AND o.status = :status "
                    + "ORDER BY o.id ASC"
                )
                .parameterValues(params)
                .build();
    }

    @Bean
    public ItemProcessor<Order, Settlement> orderItemProcessor() {
        return order -> {
            if (order.getAmount() <= 0) return null;  // null 반환 = 필터링

            Settlement settlement = new Settlement();
            settlement.setUserId(order.getUserId());
            settlement.setTotalAmount(order.getAmount());
            settlement.setSettlementDate(order.getOrderDate());
            settlement.setCreatedAt(LocalDateTime.now());
            return settlement;
        };
    }

    @Bean
    public ItemWriter<Settlement> settlementItemWriter() {
        return items -> settlementRepository.saveAll(items);
    }
}
```

### 스케줄러로 실행

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SettlementScheduler {

    private final JobLauncher jobLauncher;
    private final Job orderSettlementJob;

    @Scheduled(cron = "0 0 1 * * *")   // 매일 새벽 1시
    public void runSettlementJob() {
        String targetDate = LocalDate.now().minusDays(1).toString();

        try {
            JobParameters params = new JobParametersBuilder()
                    .addString("targetDate", targetDate)
                    .addLong("timestamp", System.currentTimeMillis())  // 재실행 보장
                    .toJobParameters();

            JobExecution execution = jobLauncher.run(orderSettlementJob, params);
            log.info("[Settlement] Job 완료 | status={} | date={}",
                    execution.getStatus(), targetDate);

        } catch (Exception e) {
            log.error("[Settlement] Job 실행 실패 | date={}", targetDate, e);
        }
    }
}
```

---

## 5. Tasklet 방식 — 단순 작업 처리

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CleanupTasklet implements Tasklet {

    private final OrderRepository orderRepository;

    @Override
    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {

        LocalDate cutoff = LocalDate.now().minusDays(90);
        int deleted = orderRepository.deleteByStatusAndOrderDateBefore("CANCELLED", cutoff);

        log.info("[Cleanup] 취소 주문 {}건 삭제 완료", deleted);
        return RepeatStatus.FINISHED;
    }
}

@Bean
public Step cleanupStep() {
    return new StepBuilder("cleanupStep", jobRepository)
            .tasklet(cleanupTasklet, transactionManager)
            .build();
}
```

---

## 6. 멀티 Step Job — 흐름 제어

```java
@Bean
public Job monthlyReportJob() {
    return new JobBuilder("monthlyReportJob", jobRepository)
            .start(aggregateDataStep())
            .next(generateReportStep())
            .next(sendReportStep())
            .on("COMPLETED").end()
            .from(sendReportStep()).on("FAILED").to(notifyFailureStep())
            .end()
            .build();
}
```

### 조건부 흐름 — ExitStatus 활용

```java
@Bean
public Step aggregateDataStep() {
    return new StepBuilder("aggregateDataStep", jobRepository)
            .tasklet((contribution, chunkContext) -> {
                long count = orderRepository.countByOrderDate(LocalDate.now().minusDays(1));

                if (count == 0) {
                    contribution.setExitStatus(new ExitStatus("NO_DATA"));
                } else {
                    contribution.setExitStatus(ExitStatus.COMPLETED);
                }
                return RepeatStatus.FINISHED;
            }, transactionManager)
            .build();
}

@Bean
public Job conditionalJob() {
    return new JobBuilder("conditionalJob", jobRepository)
            .start(aggregateDataStep())
                .on("NO_DATA").end()
                .on("COMPLETED").to(generateReportStep())
            .end()
            .build();
}
```

---

## 7. 성능 최적화 전략

### 7-1. Chunk Size 튜닝

```java
// 일반적인 기준점
// - 단순 읽기/쓰기: 500 ~ 2000
// - 복잡한 변환 로직 포함: 100 ~ 500
// - 대용량 필드(CLOB/BLOB 포함): 50 ~ 200

// StepExecutionListener로 처리 시간 측정
@Bean
public StepExecutionListener stepTimingListener() {
    return new StepExecutionListenerSupport() {
        @Override
        public void beforeStep(StepExecution stepExecution) {
            log.info("[Step 시작] {}", stepExecution.getStepName());
        }

        @Override
        public ExitStatus afterStep(StepExecution stepExecution) {
            log.info("[Step 완료] {} | 읽기={} | 쓰기={} | 스킵={} | 소요={}ms",
                    stepExecution.getStepName(),
                    stepExecution.getReadCount(),
                    stepExecution.getWriteCount(),
                    stepExecution.getSkipCount(),
                    stepExecution.getEndTime().getTime() - stepExecution.getStartTime().getTime());
            return stepExecution.getExitStatus();
        }
    };
}
```

### 7-2. Parallel Step — 독립 Step 병렬 실행

```java
@Bean
public Job parallelStepsJob() {
    Flow userSettlementFlow = new FlowBuilder<Flow>("userSettlementFlow")
            .start(userSettlementStep()).build();

    Flow productStatsFlow = new FlowBuilder<Flow>("productStatsFlow")
            .start(productStatsStep()).build();

    Flow parallelFlow = new FlowBuilder<Flow>("parallelFlow")
            .split(new SimpleAsyncTaskExecutor())
            .add(userSettlementFlow, productStatsFlow)
            .build();

    return new JobBuilder("parallelStepsJob", jobRepository)
            .start(parallelFlow)
            .next(finalizeStep())
            .end()
            .build();
}
```

### 7-3. Multi-threaded Step — 청크 병렬 처리

```java
@Bean
public Step multiThreadedStep() {
    return new StepBuilder("multiThreadedStep", jobRepository)
            .<Order, Settlement>chunk(CHUNK_SIZE, transactionManager)
            .reader(orderItemReader(null))
            .processor(orderItemProcessor())
            .writer(settlementItemWriter())
            .taskExecutor(taskExecutor())
            .throttleLimit(4)
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(25);
    executor.setThreadNamePrefix("batch-thread-");
    executor.initialize();
    return executor;
}
```

> **주의**: JpaPagingItemReader는 thread-safe하지 않음.
> 멀티스레드 환경에서는 SynchronizedItemStreamReader로 감싸거나 JdbcPagingItemReader 사용.

```java
@Bean
@StepScope
public SynchronizedItemStreamReader<Order> synchronizedOrderReader(
        @Value("#{jobParameters['targetDate']}") String targetDate) {
    return new SynchronizedItemStreamReaderBuilder<Order>()
            .delegate(orderItemReader(targetDate))
            .build();
}
```

### 7-4. Partitioning — 데이터 분할 병렬 처리

```java
@Component
public class OrderPartitioner implements Partitioner {

    private final OrderRepository orderRepository;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Long minId = orderRepository.findMinId();
        Long maxId = orderRepository.findMaxId();

        long range = (maxId - minId) / gridSize + 1;
        Map<String, ExecutionContext> result = new HashMap<>();

        for (int i = 0; i < gridSize; i++) {
            long startId = minId + (range * i);
            long endId = startId + range - 1;

            ExecutionContext context = new ExecutionContext();
            context.putLong("startId", startId);
            context.putLong("endId", Math.min(endId, maxId));
            result.put("partition-" + i, context);
        }
        return result;
    }
}

@Bean
public Step partitionManagerStep() {
    return new StepBuilder("partitionManagerStep", jobRepository)
            .partitioner("workerStep", orderPartitioner)
            .step(workerStep())
            .gridSize(8)
            .taskExecutor(taskExecutor())
            .build();
}

@Bean
@StepScope
public JdbcPagingItemReader<Order> partitionedOrderReader(
        @Value("#{stepExecutionContext['startId']}") Long startId,
        @Value("#{stepExecutionContext['endId']}") Long endId) {

    Map<String, Object> params = new HashMap<>();
    params.put("startId", startId);
    params.put("endId", endId);

    SqlPagingQueryProviderFactoryBean queryProvider = new SqlPagingQueryProviderFactoryBean();
    queryProvider.setSelectClause("SELECT id, user_id, product_id, amount, order_date");
    queryProvider.setFromClause("FROM orders");
    queryProvider.setWhereClause("WHERE id BETWEEN :startId AND :endId AND status = 'COMPLETED'");
    queryProvider.setSortKey("id");

    try {
        return new JdbcPagingItemReaderBuilder<Order>()
                .name("partitionedOrderReader")
                .dataSource(dataSource)
                .queryProvider(queryProvider.getObject())
                .parameterValues(params)
                .pageSize(CHUNK_SIZE)
                .rowMapper(new BeanPropertyRowMapper<>(Order.class))
                .build();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

### 7-5. JdbcBatchItemWriter — 대량 Insert 최적화

```java
@Bean
public JdbcBatchItemWriter<Settlement> jdbcBatchSettlementWriter() {
    return new JdbcBatchItemWriterBuilder<Settlement>()
            .dataSource(dataSource)
            .sql(
                "INSERT INTO settlement (user_id, total_amount, settlement_date, created_at) "
                + "VALUES (:userId, :totalAmount, :settlementDate, :createdAt)"
            )
            .beanMapped()
            .build();
}

// MySQL: rewriteBatchedStatements=true 설정 시 실제 배치 Insert 활성화
// url: jdbc:mysql://host/db?rewriteBatchedStatements=true
```

**성능 비교 (100만 건 기준)**

| 방식 | 소요 시간 |
| --- | --- |
| JPA `save()` 건별 | 약 35분 |
| JPA `saveAll()` | 약 12분 |
| `JdbcBatchItemWriter` | 약 3분 |
| `JdbcBatchItemWriter` + `rewriteBatchedStatements` | 약 1분 |

---

## 8. 오류 처리 전략 — Skip & Retry

```java
@Bean
public Step faultTolerantStep() {
    return new StepBuilder("faultTolerantStep", jobRepository)
            .<Order, Settlement>chunk(CHUNK_SIZE, transactionManager)
            .reader(orderItemReader(null))
            .processor(orderItemProcessor())
            .writer(settlementItemWriter())
            .faultTolerant()
            .skip(InvalidDataException.class)
            .skip(NullPointerException.class)
            .skipLimit(100)
            .retry(DataAccessException.class)
            .retryLimit(3)
            .noSkip(OutOfMemoryError.class)
            .listener(skipListener())
            .build();
}

@Bean
public SkipListener<Order, Settlement> skipListener() {
    return new SkipListenerSupport<>() {
        @Override
        public void onSkipInProcess(Order item, Throwable t) {
            log.warn("[Skip] 처리 중 스킵 | orderId={} | error={}",
                    item.getId(), t.getMessage());
        }

        @Override
        public void onSkipInWrite(Settlement item, Throwable t) {
            log.warn("[Skip] 저장 중 스킵 | userId={} | error={}",
                    item.getUserId(), t.getMessage());
        }
    };
}
```

---

## 9. JobExecutionListener — 전처리 / 후처리

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SettlementJobListener implements JobExecutionListener {

    private final SlackNotifier slackNotifier;

    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("[Job 시작] {} | params={}",
                jobExecution.getJobInstance().getJobName(),
                jobExecution.getJobParameters());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        BatchStatus status = jobExecution.getStatus();
        long readCount  = jobExecution.getStepExecutions().stream()
                .mapToLong(StepExecution::getReadCount).sum();
        long writeCount = jobExecution.getStepExecutions().stream()
                .mapToLong(StepExecution::getWriteCount).sum();

        if (status == BatchStatus.COMPLETED) {
            log.info("[Job 완료] 읽기={} 쓰기={}", readCount, writeCount);
        } else if (status == BatchStatus.FAILED) {
            log.error("[Job 실패] status={}", status);
            slackNotifier.sendAlert("정산 배치 실패!");
        }
    }
}
```

---

## 10. ItemReader 선택 가이드

| Reader | 적합한 케이스 | thread-safe | 특징 |
| --- | --- | --- | --- |
| `JpaPagingItemReader` | JPA 엔티티 기반 처리 | ❌ | JPQL 사용, 커서 없음 |
| `JdbcPagingItemReader` | 대용량, 멀티스레드 | ✅ | 순수 JDBC, 빠름 |
| `JdbcCursorItemReader` | 단일스레드 대용량 | ❌ | 커서 방식, 메모리 효율적 |
| `FlatFileItemReader` | CSV/TXT 파일 읽기 | ❌ | 구분자 기반 파싱 |
| `StaxEventItemReader` | XML 파일 읽기 | ❌ | SAX 방식 스트리밍 |
| `MongoItemReader` | MongoDB 컬렉션 읽기 | ✅ | Spring Data MongoDB |

### FlatFileItemReader — CSV 처리 예시

```java
@Bean
@StepScope
public FlatFileItemReader<OrderCsv> csvOrderReader(
        @Value("#{jobParameters['filePath']}") String filePath) {

    return new FlatFileItemReaderBuilder<OrderCsv>()
            .name("csvOrderReader")
            .resource(new FileSystemResource(filePath))
            .linesToSkip(1)
            .delimited()
            .delimiter(",")
            .names("userId", "productId", "amount", "orderDate")
            .targetType(OrderCsv.class)
            .build();
}
```

---

## 11. 실무 운영 패턴

### 중복 실행 방지

```java
// 동일 JobParameters로 COMPLETED된 Job은 재실행 불가
// FAILED 상태는 재시작 가능 (재해 복구)
JobParameters params = new JobParametersBuilder()
        .addString("targetDate", "2026-04-28")
        .toJobParameters();
```

### 배치 메타 테이블 관리

```sql
-- 실행 이력 조회
SELECT ji.JOB_NAME,
       je.START_TIME,
       je.END_TIME,
       je.STATUS,
       je.EXIT_CODE,
       TIMESTAMPDIFF(SECOND, je.START_TIME, je.END_TIME) AS duration_sec
FROM BATCH_JOB_EXECUTION je
JOIN BATCH_JOB_INSTANCE  ji ON je.JOB_INSTANCE_ID = ji.JOB_INSTANCE_ID
ORDER BY je.START_TIME DESC
LIMIT 20;

-- 오래된 이력 정리 (90일 이상)
DELETE FROM BATCH_JOB_EXECUTION
WHERE START_TIME < DATE_SUB(NOW(), INTERVAL 90 DAY)
  AND STATUS IN ('COMPLETED', 'FAILED');
```

---

## 12. 성능 체크리스트

**Reader 최적화**
- `@StepScope`로 지연 초기화 → JobParameter 주입 가능
- Paging Reader 정렬 기준 컬럼에 인덱스 필수
- 멀티스레드 환경: `JdbcPagingItemReader` 또는 `SynchronizedItemStreamReader` 사용

**Writer 최적화**
- JPA 대신 `JdbcBatchItemWriter` 사용
- MySQL: `rewriteBatchedStatements=true` JDBC URL 옵션 추가
- 대량 Insert 전 인덱스 비활성화 고려 (운영 환경 주의)

**트랜잭션 최적화**
- Chunk Size를 크게 잡아 트랜잭션 오버헤드 감소
- 읽기 전용 처리: `@Transactional(readOnly = true)` 적용
- 불필요한 영속성 컨텍스트 초기화: `EntityManager.clear()` 주기적 호출

**병렬 처리 선택 기준**

| 데이터 규모 | 전략 |
| --- | --- |
| 수만 건 | 단일 스레드 Chunk |
| 수십만 건 | Multi-threaded Step |
| 수백만 건 | Partitioning |
| 수억 건 | Partitioning + 원격 분산 처리 |

---

## 정리

배치는 "한 번에 많이 처리"가 핵심이지만, 실무에서는 **언제 실패할지 모른다는 전제**로 설계해야 함.

- 청크 단위 트랜잭션 → **장애 범위 최소화**
- Skip / Retry → **유연한 오류 허용**
- Partitioning → **수평적 확장**
- JobExecutionListener → **운영 가시성 확보**
- 메타 테이블 → **실행 이력 추적 및 재시작 보장**

Spring Batch는 이 모든 걸 프레임워크 레벨에서 지원하기 때문에, 비즈니스 로직에만 집중하면서도 엔터프라이즈급 안정성을 확보할 수 있음.
