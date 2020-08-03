# Dummy Example - WhiteLabel Digital Twin Framework - Core Library

This project shows how to implement a simple WLDT Worker with a demo configuration and behavior.

### WLDT Worker Configuration

The interface WldtWorkerConfiguration allows to create your personal data structure containing all the needed information for 
your new Worker to clone its physical counterpart and start operating. In the following example a new DummyConfiguration is created with some 
parameters, constructors and get/set methods.

```java
public class DummyWorkerConfiguration implements WldtWorkerConfiguration {

    private String username = null;
    private String password = null;
    private String twinIp = null;
    private int twinPort = 8080;

    public DummyWorkerConfiguration() {
    }

    public DummyWorkerConfiguration(String username, String password, String twinIp, int twinPort) {
        this.username = username;
        this.password = password;
        this.twinIp = twinIp;
        this.twinPort = twinPort;
    }

    public String getUsername() { return username;}

    public void setUsername(String username) { this.username = username; }

    public String getPassword() { return password; }

    public void setPassword(String password) { this.password = password;}

    public String getTwinIp() { return twinIp; }

    public void setTwinIp(String twinIp) { this.twinIp = twinIp; }

    public int getTwinPort() { return twinPort; }

    public void setTwinPort(int twinPort) { this.twinPort = twinPort; }
}
```

### Worker Implementation

WLDT allows developers to create new custom workers to handle and define specific behaviors and properly manage 
the automatic synchronization with physical counterparts. A new worker class extends the  class **WldtWorker** specifying the 
class types related to: the configuration through (by extending of the base class *WldtWorkerConfiguration* and the class 
types for the key and value that can be used for the worker's cache. 
The main method that the developer must override in order to implement the behavior of its module is *startWorkerJob()*. 
This function is called by the engine a the startup instant when it is ready to execute the worker(s) on the thread pool. 
Internally the worker can implement its own logic with additional multi-threading solutions and the import of any required libraries. 

In the following example we are creating a Dummy worker emulating a set of GET request to an external object. 
The complete example can be found in the repository [WLDT Dummy Worker](https://github.com/wldt/wldt-dummy-example).
The WldtDummyWorker class extends WldtWorker specifying DummyWorkerConfiguration as configuration reference and String and Integer
as Key and Value of the associated Worker caching system. 

```java
public class WldtDummyWorker extends WldtWorker<DummyWorkerConfiguration, String, Integer> {

    public WldtDummyWorker(String wldtId, DummyWorkerConfiguration dummyWorkerConfiguration) {
        super(dummyWorkerConfiguration);
        this.random = new Random();
        this.wldtId = wldtId;
    }

    public WldtDummyWorker(String wldtId, DummyWorkerConfiguration dummyWorkerConfiguration, IWldtCache<String, Integer> wldtCache) {
        super(dummyWorkerConfiguration, wldtCache);
        this.random = new Random();
        this.wldtId = wldtId;
    }

    @Override
    public void startWorkerJob() throws WldtConfigurationException, WldtRuntimeException {

        try{
            for(int i = 0; i < RUN_COUNT_LIMIT; i++)
                emulateExternalGetRequest(i);
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    private void emulateExternalGetRequest(int roundIndex) {
        ...
    }

}
```

### Internal Data Caching System

The framework provides an internal shared caching system that can be adopted by each worker specifying the typology 
of its cache in terms of key and value class types. The interface \code{IWldtCache<K,V>} 
defines the methods for a \NAME cache and a default implementation is provided by the 
framework through the class \code{WldtCache<K, V>}. 
Main exposed methods are: 

* void initCache()
* void deleteCache()
* void putData(K key, V value)
* Optional<V> getData(K key)
* void removeData(K key) 

Each cache instance is characterized by a string identifier and optionally 
by an expiration time and a size limit. 
An instance can be directly passed as construction parameter of 
a worker or it can be internally created for example inside a processing pipeline to handle cached data during data synchronization.

```java
if(this.workerCache != null && this.workerCache.getData(CACHE_VALUE_KEY).isPresent()) {
    physicalObjectValue = this.workerCache.getData(CACHE_VALUE_KEY).get();
}
else{
    physicalObjectValue = retrieveValueFromPhysicalObject();
    if(this.workerCache != null)
        this.workerCache.putData(CACHE_VALUE_KEY, physicalObjectValue);
}
```

The cache is configured when the single worker is created in the setup phase. See the following code and the next sections.

```java
WldtDummyWorker wldtDummyWorker = new WldtDummyWorker(
        wldtEngine.getWldtId(),
        new DummyWorkerConfiguration(),
        new WldtCache<>(5, TimeUnit.SECONDS));
```

### Processing Pipelines

As previously mentioned, the Processing Pipeline layer has been introduced to allow developer to easily and dynamically 
customize the behavior of WLDT workers through the use of configurable processing chains. 
The pipeline is defined through the interface ***IProcessingPipeline*** and the step definition 
class ***ProcessingStep***. Main methods to work and configure the pipeline are: 
* addStep(ProcessingStep step)
* removeStep(ProcessingStep step) 
* start(PipelineData initialData, ProcessingPipelineListener listener)  
 
***ProcessingStep*** and ***PipelineData*** classes are used to describe and implement each single step and to model the data passed trough the chain. 
Each step takes as input an initial PipelineData value and produces as output a new one. 
Two listeners classes have been also defined (***ProcessingPipelineListener*** and ***ProcessingStepListener***) 
to notify interested actors about the status of each step and/or the final results of the processing pipeline 
through the use of methods ***onPipelineDone(Optional<PipelineData> result)*** and ***onPipelineError()***.

An example of PipelineData Implementation is the following:

```java
public class DummyPipelineData implements PipelineData {

    private int value = 0;

    public DummyPipelineData(int value) { this.value = value; }

    public int getValue() { return value; }

    public void setValue(int value) { this.value = value; }

}
```

A dummy processing step using the DummyPipelineData class created is presented in the following code snippet.
The developer can override the ***execute*** method in order to define the single Step implementation receiving as parameters 
the incoming *Pipeline* data and the *ProcessingStepListener*. 
The listener is used to notify when the step has been correctly (*onStepDone*) completed and the new PipelineData output value has been generated. 
In case of an error the method *onStepError* can be used to notify the event.

```java
public class DummyProcessingStep implements ProcessingStep {

    @Override
    public void execute(PipelineCache pipelineCache, PipelineData data, ProcessingStepListener listener) {
        if(data instanceof DummyPipelineData && listener != null) {

            DummyPipelineData pipelineData = (DummyPipelineData)data;

            //Updating pipeline data
            pipelineData.setValue(pipelineData.getValue()*2);

            listener.onStepDone(this, Optional.of(pipelineData));
        }
        else {
            if(listener != null) {
                String errorMessage = "PipelineData Error !";
                listener.onStepError(this, data, errorMessage);
            }
            else
                logger.error("Processing Step Listener = Null ! Skipping processing step");
        }
    }
}
```

### Monitor Metrics and Performance

The framework allows the developer to easily define, measure, 
track and send to a local collector all the application's metrics and logs. 
This information can be also useful to dynamically balance the load on active 
twins operating on distributed clusters or to detect unexpected behaviors or performance degradation. 
The framework implements a singleton class called ***WldtMetricsManager*** exposing the methods:
 * getTimer(String metricId, String timerKey) 
 * measureValue(String metricId, String key, int value) 
 
These methods can be used to track elapsed time of a specific processing code block or with the second option to measure 
a value of interest associated to a key identifier. 

```java
private void emulateExternalGetRequest(int roundIndex) {

    Timer.Context metricsContext = WldtMetricsManager.getInstance().getTimer(String.format("%s.%s", METRIC_BASE_IDENTIFIER, this.wldtId), WORKER_EXECUTION_TIME_METRICS_FIELD);

    try{

        //YOUR CODE

    }catch (Exception e){
        e.printStackTrace();
    }
    finally {
        if(metricsContext != null)
            metricsContext.stop();
    }
}
```

<p align="center">
  <img class="center" src="images/example_metrics.png" width="80%">
</p>

The WLDT metric system provides by default two reporting option allowing the developer 
to periodically save the metrics on a CSV file or to send them directly to a Graphite collector.
An example of WldtConfiguration enabling both CSV and Graphite monitoring is the following: 

```java
WldtConfiguration wldtConfiguration = new WldtConfiguration();
wldtConfiguration.setDeviceNameSpace("it.unimore.dipi.things");
wldtConfiguration.setWldtBaseIdentifier("wldt");
wldtConfiguration.setWldtStartupTimeSeconds(10);
wldtConfiguration.setApplicationMetricsEnabled(true);
wldtConfiguration.setApplicationMetricsReportingPeriodSeconds(10);
wldtConfiguration.setMetricsReporterList(Arrays.asList("csv", "graphite"));
wldtConfiguration.setGraphitePrefix("wldt");
wldtConfiguration.setGraphiteReporterAddress("127.0.0.1");
wldtConfiguration.setGraphiteReporterPort(2003);
```

### Execute the new Worker

A new worker can be executed through the use of the main class WldtEngine. 
The class WldtConfiguration is used to configure the behavior of the WLDT framework as illustrated in the followinf simple example. 

```java
public static void main(String[] args)  {

    try{

        //Manual creation of the WldtConfiguration
        WldtConfiguration wldtConfiguration = new WldtConfiguration();
        wldtConfiguration.setDeviceNameSpace("it.unimore.dipi.things");
        wldtConfiguration.setWldtBaseIdentifier("it.unimore.dipi.iot.wldt.example.dummy");
        wldtConfiguration.setWldtStartupTimeSeconds(10);
        wldtConfiguration.setApplicationMetricsEnabled(true);
        wldtConfiguration.setApplicationMetricsReportingPeriodSeconds(10);
        wldtConfiguration.setMetricsReporterList(Collections.singletonList("csv"));

        //Init the Engine
        WldtEngine wldtEngine = new WldtEngine(wldtConfiguration);

        //Init Dummy Worker with Cache
        WldtDummyWorker wldtDummyWorker = new WldtDummyWorker(
                wldtEngine.getWldtId(),
                new DummyWorkerConfiguration(),
                new WldtCache<>(5, TimeUnit.SECONDS));

        //Set a Processing Pipeline
        wldtDummyWorker.addProcessingPipeline(WldtDummyWorker.DEFAULT_PROCESSING_PIPELINE, new ProcessingPipeline(new DummyProcessingStep()));
        
        //Add the New Worker
        wldtEngine.addNewWorker(wldtDummyWorker);
        
        //Start workers
        wldtEngine.startWorkers();

    }catch (Exception | WldtConfigurationException e){
        e.printStackTrace();
    }
}
```
