# Starting Spark Master & Workers

**starting master:**
> spark-class org.apache.spark.deploy.master.Master

**starting workers:**
> spark-class org.apache.spark.deploy.worker.Worker spark://{host}:7077

**Properties Loading Sequence**
Properties set directly on the SparkConf take highest precedence, then flags passed to spark-submit or spark-shell, then options in the spark-defaults.conf file.
