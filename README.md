# Spring Batch Notes
### Terminology and Important Concepts
Spring batch process is encapsulated by a Job which consists of multiple steps. 
Each step will have ItemReader, ItemProcessor and ItemWriter.
A job is executed by JobLauncher.
Metadata about configured and executed jobs is stored in JobRepository.

Each job maybe associated with multiple job instances, each of which is defined uniquely by its particular JobParameters that are used to start a batch job.
Each run of a job instance is referred to as a JobExecution. Each job execution typically tracks what happened during a run, such as current and exit status, start and end times, etc.

A step is an independent, specific phase of a batch Job. Every Job is composed of one or more steps. 
Similar to job, a step has individual step execution that represents single attempt to execute a step. Step execution stores the information about current and exit statuses, start and end times, and so on.
Step execution will also hold references to step and jobExecution instances.

ExecutionContext is key-value store that holds information scoped to step execution or job execution. Spring batch persists the execution context, which helps in cases where you want to restart the job. 
All that is needed is to put any object to be shared between steps into the context and the framework will take care of the rest. After restart, the values from the prior ExecutionContext are restored from the 
database and applied.

JobRepository is the mechanism in Spring Batch that makes all this persistence possible. It provides CRUD operations tfor JobLauncher, Job and Step instantiations. Once a JOb is launched, job execution is obtained from the repository and, during the course of execution, StepExecution and JObExecution instances are persisted to the repository.
Two main approaches to build a step. 1. Tasklet-based and 2.chunk-oriented processing.

Tasklet-based approach has simple interface with single method called "execute()" that will be called repeatedly util it returns RepeatStatus.FINISHED or throws an exception to signal a failure.
Each call to the tasklet is wrapped in a transaction.

In Chunk-oriented processing approach, refers to reading the data sequentially and creating "chunks" that will be written out within a transaction boundary. Each individual item is read from an ItemReader, handed to a ItemProcessor, and aggregated. Once the number of item read equals the commit interval, the entire chunk is written out via the ItemWriter, and then the transaction is committed. 

### Tasklet-based vs Chunk-oriented Processing
Different contexts will show the need for one approach or the other. While Tasklets feel more natural for "one task after the other" scenarios, chunks provide a simple solution to deal with paginated reads or situations where we don't want to keep a significant amount of data in memory.

### ItemReader

```
public interface ItemReader<T> {
    T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException;
}
```

An ItemReader provides the data and is expected to be stateful. It is typically called multiple times for each batch, with each call to read() returning the next value and finally returning null when all input data has been exhausted.
Spring Batch provides some out-of-the-box implementations of ItemReader, which can be used for a variety of purposes such as reading collections, files, integrating JMS and JDBC as well as multiple sources, and so on.

A Spring bean for this implementation is created with the @Component and @StepScope annotations, letting Spring know that this class is a step-scoped Spring component and will be created once per step execution as follows:

```
@StepScope
@Bean
public ItemReader<Customer> reader() {
    return new CustomerItemReader(XML_FILE);
}
```
        
### ItemProcessor

ItemProcessors transform input items and introduce business logic in an item-oriented processing scenario. They must provide an implementation of the interface org.springframework.batch.item.ItemProcessor:

```
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
}
```
        
The method process() accepts one instance of the I class and may or may not return an instance of the same type. Returning null indicates that the item should not continue to be processed. 
As usual, Spring provides few standard processors, such as CompositeItemProcessor that passes the item through a sequence of injected ItemProcessors and a ValidatingItemProcessor that validates input.

### ItemWriter
public interface ItemWriter<T> {
    void write(List<? extends T> items) throws Exception;
}

For outputting the data, Spring Batch provides the interface org.springframework.batch.item.ItemWriter for serializing objects as necessary.
The write() method is responsible for making sure that any internal buffers are flushed. If a transaction is active, it will also usually be necessary to discard the output on a subsequent rollback. 
The resource to which the writer is sending data should normally be able to handle this itself. There are standard implementations such as CompositeItemWriter, JdbcBatchItemWriter, JmsItemWriter, JpaItemWriter, SimpleMailMessageItemWriter, and others.

### Scheduling Spring Batch Jobs
By default, Spring Batch executes all jobs it can find (i.e., that are configured as in CustomerReportJobConfig) at startup. To change this behavior, disable job execution at startup by adding the following property to application.properties:

```
spring.batch.job.enabled=false
```

The actual scheduling is then achieved by adding the @EnableScheduling annotation to a configuration class and the @Scheduled annotation to the method that executes the job itself. Scheduling can be configured with delay, rates, or cron expressions:

// run every 5000 msec (i.e., every 5 secs)
@Scheduled(fixedRate = 5000)
public void run() throws Exception {
    JobExecution execution = jobLauncher.run(
        customerReportJob(), // method in config that returns Job instance.
        new JobParametersBuilder().toJobParameters()
    );
}

### Problem with above job launch
There is a problem with the above example though. At run time, the job will succeed the first time only. When it launches the second time (i.e. after five seconds), it will generate the error messages in the logs
(note that in previous versions of Spring Batch a JobInstanceAlreadyCompleteException would have been thrown).

### Reason
This happens because only unique JobInstances may be created and executed and Spring Batch has no way of distinguishing between the first and second JobInstance.

There are two ways of avoiding this problem when you schedule a batch job.

One is to be sure to introduce one or more unique parameters (e.g., actual start time in nanoseconds) to each job:

```
@Scheduled(fixedRate = 5000)
public void run() throws Exception {
    jobLauncher.run(
        customerReportJob(),
        new JobParametersBuilder().addLong("uniqueness", System.nanoTime()).toJobParameters()
    );
}
```

Alternatively, you can launch the next job in a sequence of JobInstances determined by the JobParametersIncrementer attached to the specified job with SimpleJobOperator.startNextInstance():

```
@Autowired
private JobOperator operator;
 
@Autowired
private JobExplorer jobs;
 
@Scheduled(fixedRate = 5000)
public void run() throws Exception {
    List<JobInstance> lastInstances = jobs.getJobInstances(JOB_NAME, 0, 1);
    if (lastInstances.isEmpty()) {
        jobLauncher.run(customerReportJob(), new JobParameters());
    } else {
        operator.startNextInstance(JOB_NAME);
    }
}

```

### Run Job with multiple steps

```
@Bean
public Step stepOne(){
return steps.get("stepOne")
        .tasklet(new MyTaskOne())
        .build();
}

@Bean
public Step stepTwo(){
return steps.get("stepTwo")
        .tasklet(new MyTaskTwo())
        .build();
} 

@Bean
public Job demoJob(){
return jobs.get("demoJob")
        .incrementer(new RunIdIncrementer())
        .start(stepOne())
        .next(stepTwo())
        .build();
}

```

Psuedo code that explains the processing inside the step of batch job.

```
List items = new Arraylist();
for(int i = 0; i < commitInterval; i++){
    Object item = itemReader.read();
    if (item != null) {
        items.add(item);
    }
}

List processedItems = new Arraylist();
for(Object item: items){
    Object processedItem = itemProcessor.process(item);
    if (processedItem != null) {
        processedItems.add(processedItem);
    }
}

itemWriter.write(processedItems);
```

### Transaction scope
Transaction scope in chunk-oriented processing includes reading chunk, processing chunk and writing the chunk.
Transaction scope in tasklet is  single call to execute() method.

### Sharing data between the steps
If the data is small then the data can be saved in execution context. Otherwise, use a temporary table or file to share the data with other steps.

### JobExplorer
Use JobExplorer to query the Job repository for job instances/job executions for specific job name.

```
@PreDestroy
public void destroy() throws Exception {
    jobExplorer.getJobNames().forEach(name -> log.info("job name: {}", name));
    jobExplorer.getJobInstances("linesJob", 0, jobExplorer.getJobInstanceCount("linesJob")).forEach(jobInstance -> {
        log.info("job instance id {}", jobInstance.getInstanceId());
    });
}

```
### JobOperator
JobOperator is needed to start a job with new instance or to resume previous execution of job instance.

```
@Scheduled(fixedRate = 5000000)
public void run() throws Exception {
    JobInstance lastJobInstance = jobExplorer.getLastJobInstance("linesJob");
    if (lastJobInstance == null) {
        jobLauncher.run(linesJob, new JobParameters());
    } else {

        JobExecution lastJobExecution = jobExplorer.getLastJobExecution(lastJobInstance);
        if (lastJobExecution != null && lastJobExecution.getStatus() == BatchStatus.FAILED) {
            operator.restart(lastJobExecution.getId());
        } else {
            operator.startNextInstance("linesJob");
        }
    }
}

```


### Note
* If we run the job with same job parameters then same job instance will be picked. If previous execution for that instance is successful then in current execution, no action will happen at each step. 
* To re-run the job with new instance, we need to execute job with new distinct parameters.
* Attaching RunIdIncrementer to job results in adding “run.id” parameter with incremented value to the job before launching a new instance.
Example:
job execution started for job with name=linesJob and parameters={run.id=6}
* Refer com.example.springbatchdemo.config.JobConfig class for customizing different spring batch job beans like JobExplorer, JobRepository, ExecutionContextSerializer (for saving entities to ExecutionContext), JobRegistryBeanPostProcessor (to register all jobs with jobRegistry so that we can use JobOperator to start/stop/restart jobs)
* It is always better to save objects in string or JSON format  to execution context using ObjectMapper and convert them to Objects again using ObjectMapper after reading from ExecutionContext.

### JobExecutionListener and StepExecutionListener
We can extend JobExecutionListenerSupport class to intercept after job execution calls. If we have any logic to execute after job execution we can implement there.
Example:
```
@Component
@Slf4j
public class JobCompletionNotificationListener extends JobExecutionListenerSupport {

	@Override
	public void afterJob(JobExecution jobExecution) {
		if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
			log.info("!!! JOB FINISHED! Time to verify the results");

		}
	}
}
```
Similary, we can implement StepExecutionListener interface to intercept before step and after step execution calls.
Example:
```
@Slf4j
@Component
public class LinesReader implements Tasklet, StepExecutionListener {

	@Autowired
	private ObjectMapper objectMapper;

	private List<Line> lines = new ArrayList<Line>();

	@Override
	public void beforeStep(StepExecution stepExecution) {
		log.info("Lines Reader initialized");
	}

	@Override
	public ExitStatus afterStep(StepExecution stepExecution) {
		try {
			stepExecution.getJobExecution().getExecutionContext().put("lines", objectMapper.writeValueAsString(lines));
		} catch (JsonProcessingException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		log.info("Lines Reader ended");
		return ExitStatus.COMPLETED;
	}

	@Override
	public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
		lines = DataUtils.getLines();
		log.info("Line Reader retrieved lines");
		return RepeatStatus.FINISHED;
	}

}
```

### ItemReader vs ItemWriter vs ItemStream
Both ItemReaders and ItemWriters serve their individual purposes well, but there is a common concern among both of them that necessitates another interface. In general, as part of the scope of a batch job, readers and writers need to be opened, closed, and require a mechanism for persisting state. The ItemStream interface serves that purpose, as shown in the following example:

```
public interface ItemStream {

    void open(ExecutionContext executionContext) throws ItemStreamException;

    void update(ExecutionContext executionContext) throws ItemStreamException;

    void close() throws ItemStreamException;
}
```

Clients of an ItemReader that also implement ItemStream should call open before any calls to read, in order to open any resources such as files or to obtain connections. A similar restriction applies to an ItemWriter that implements ItemStream. As mentioned in Chapter 2, if expected data is found in the ExecutionContext, it may be used to start the ItemReader or ItemWriter at a location other than its initial state. Conversely, close is called to ensure that any resources allocated during open are released safely. update is called primarily to ensure that any state currently being held is loaded into the provided ExecutionContext. This method is called before committing, to ensure that the current state is persisted in the database before commit.
In the special case where the client of an ItemStream is a Step (from the Spring Batch Core), an ExecutionContext is created for each StepExecution to allow users to store the state of a particular execution, with the expectation that it is returned if the same JobInstance is started again. 

So, if you want your custom reader or writer to be stateful, your reader/writer should implement ItemStream interface too.When reader/writer is stateful then it can pick up where it left in previous run.

The client of the writer needs to be aware of the ItemStream, so you may need to register it as a stream in the configuration. For example, in step configuration, we need to register streams in that step.

Example:
```
Bean
public Step step1() {
	return this.stepBuilderFactory.get("step1")
				.<String, String>chunk(2)
				.reader(fooReader())
				.processor(fooProcessor())
				.writer(compositeItemWriter())
				.stream(barWriter())
				.build();
}

@Bean
public CustomCompositeItemWriter compositeItemWriter() {

	CustomCompositeItemWriter writer = new CustomCompositeItemWriter();

	writer.setDelegate(barWriter());

	return writer;
}

@Bean
public BarWriter barWriter() {
	return new BarWriter();
}
```
### CompositeItemReader that delegates read to multiple item readers

```
class MyItemReader<T> implements ItemReader<T>, ItemStream {
  private ItemReader[] delegates;
  private int delegateIndex;
  private ItemReader<T> currentDelegate;
  private ExecutionContext stepExecutionContext;

  public void setDelegates(ItemReader[] delegates) {
    this.delegates = delegates;
  }

  @BeforeStep
  private void beforeStep(StepExecution stepExecution) {
    this.stepExecutionContext = stepExecution.getExecutionContext();
  }

  public T read() {
    T item = null;
    if(null != currentDelegate) {
      item = currentDelegate.read();
      if(null == item) {
        ((ItemStream)this.currentDelegate).close();
        this.currentDelegate = null;
      }
    }
    // Move to next delegate if previous was exhausted!
    if(null == item && this.delegateIndex< this.delegates.length) {
      this.currentDelegate = this.delegates[this.currentIndex++];
      ((ItemStream)this.currentDelegate).open(this.stepExecutionContext);
      update(this.stepExecutionContext);
      // Recurse to read() to simulate loop through delegates
      item = read();
    }
    return item;
  }

  public void open(ExecutionContext ctx) {
    // During open restore last active reader and restore its state
    if(ctx.containsKey("index")) {
      this.delegateIndex = ctx.getInt("index");
      this.currentDelegate = this.delegates[this.delegateIndex];
      ((ItemStream)this.currentDelegate ).open(ctx);
    }
  }

  public void update(ExecutionContext ctx) {
    // Update current delegate index and state
    ctx.putInt("index", this.delegateIndex);
    if(null != this.currentDelegate) {
      ((ItemStream)this.currentDelegate).update(ctx);
    }
  }

  public void close(ExecutionContext ctx) {
    if(null != this.currentDelegate) {
      ((ItemStream)this.currentDelegate).close();
  }
}
<bean id="myItemReader" class=path.to.MyItemReader>
  <property name="delegates">
    <array>
      <ref bean="itemReader1"/>
      <ref bean="itemReader2"/>
      <ref bean="itemReader3"/>
    </array>
  </property>
</bean>

<bean id="itemReader1" class="JdbcCursorItemReader">
  <property name="name" value="itemReader1" />
  <!-- Set other properties -->
</bean>
```

### Important topics
1. Parallel processing
2. Scaling

### Download Different JAVA Versions
Add below code to build.gradle and refresh gradle. Gradle will download that JAVA version in .gradle folder.

```
java {
	toolchain {
		languageVersion.set(JavaLanguageVersion.of(17))
	}
}
```

### References
* https://www.toptal.com/spring/spring-batch-tutorial
* https://self-learning-java-tutorial.blogspot.com/2020/11/spring-batch-job-with-multiple-steps.html
* https://www.baeldung.com/spring-batch-tasklet-chunk - discusses data sharing between steps
* https://www.tutorialsbuddy.com/spring-batch-with-mysql-example - JPA Item reader and item writer example
* https://www.yawintutor.com/spring-boot-batch-read-from-database-and-write-to-database-example/ - read and write to database using JDBC queries.
* https://www.petrikainulainen.net/programming/spring-framework/spring-batch-tutorial-writing-information-to-a-database-with-jdbc/ - writing data to database.
* https://docs.spring.io/spring-batch/docs/current/reference/html/step.html - provides psuedo code for better understanding of step in batch job.
* https://dzone.com/articles/a-composite-reader-for-batch-processing - custom reader that provides capability to process page data before being sent to ItemReader.read(). I cloned this git repository.
* https://coderedirect.com/questions/136895/spring-batch-job-read-from-multiple-sources - CompositeItemReader that delegates reads to multiple item readers.
* https://www.desynit.com/dev-zone/java/list-merging-in-spring-batch/ - To read data from two data sources. Data from first source will be enhanced by data fom second source.
