# Hadoop Project on Ubuntu

## Introduction
This project demonstrates how to set up and use Hadoop on Ubuntu for distributed data processing. It includes configuration of HDFS, YARN, MapReduce, and Hive.

## Prerequisites
Ensure you have:
- **Ubuntu 20.04/22.04**
- **Java 11 (OpenJDK 11)**
- **SSH configured for passwordless login**

## Installation Steps

### 1. Update System
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Java
```bash
sudo apt install openjdk-11-jdk -y
java -version
```

### 3. Create Hadoop User
```bash
sudo adduser hadoop
sudo usermod -aG sudo hadoop
su - hadoop
```

### 4. Configure SSH
```bash
ssh-keygen -t rsa -P ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh localhost
```

### 5. Download and Install Hadoop
```bash
wget https://downloads.apache.org/hadoop/common/hadoop-3.3.5/hadoop-3.3.5.tar.gz
tar -xvzf hadoop-3.3.5.tar.gz
mv hadoop-3.3.5 ~/hadoop
```

### 6. Configure Environment Variables
Add to `~/.bashrc`:
```bash
export HADOOP_HOME=$HOME/hadoop
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:/bin/java::")
```
Apply changes:
```bash
source ~/.bashrc
```

### 7. Configure Hadoop
#### `core-site.xml`
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

#### `hdfs-site.xml`
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

#### `mapred-site.xml`
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

#### `yarn-site.xml`
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

### 8. Start Hadoop
```bash
hdfs namenode -format
start-dfs.sh
start-yarn.sh
jps
```

## Install and Use Hive
### 1. Download and Install Hive
```bash
wget https://downloads.apache.org/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz
tar -xvzf apache-hive-3.1.3-bin.tar.gz
mv apache-hive-3.1.3-bin ~/hive
```
Add to `~/.bashrc`:
```bash
export HIVE_HOME=$HOME/hive
export PATH=$HIVE_HOME/bin:$PATH
```
Apply changes:
```bash
source ~/.bashrc
```

### 2. Initialize Hive
```bash
schematool -initSchema -dbType derby
hive
```

### 3. Load Data into Hive
```sql
CREATE TABLE ventes (
    id INT,
    produit STRING,
    prix DOUBLE
) 
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;

LOAD DATA INPATH '/user/hadoop/input/data.csv' INTO TABLE ventes;

SELECT produit, SUM(prix) AS total_ventes FROM ventes GROUP BY produit;
```

## Running a MapReduce Job
### Example: WordCount
Create `WordCount.java`:
```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.util.StringTokenizer;

public class WordCount {
    public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            StringTokenizer itr = new StringTokenizer(value.toString());
            while (itr.hasMoreTokens()) {
                word.set(itr.nextToken());
                context.write(word, one);
            }
        }
    }

    public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
            int sum = 0;
            for (IntWritable val : values) {
                sum += val.get();
            }
            context.write(key, new IntWritable(sum));
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        job.setJarByClass(WordCount.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setCombinerClass(IntSumReducer.class);
        job.setReducerClass(IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```
Compile and run with Hadoop:
```bash
hadoop com.sun.tools.javac.Main WordCount.java
jar cf wc.jar WordCount*.class
hadoop jar wc.jar WordCount /user/hadoop/input /user/hadoop/output
```

## Conclusion
This guide covers the installation and configuration of Hadoop and Hive, as well as an example of a MapReduce job. If you have any questions or need further assistance, feel free to ask!
