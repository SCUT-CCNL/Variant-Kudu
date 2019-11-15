# Variant-Kudu

Variant-Kudu is a scalable toolkit to analyze massive genetic variation datasets based on cloudera manager. This file introduces the environment, build and usage of Variant-Kudu, which contains the following sections.

1. System environment requirements
2. Usage and Example

## System environment requirements

Variant-Kudu is based on Apache Kudu, Apache Spark, Apache Impala. To use Variant-Kudu, you need to install the following software in your own cluster. Moveover, your system must be Linux-based.

*  Java 1.8.0_60
*  Scala 2.10.5
*  Hadoop 2.6.0
*  Spark 1.6.0
*  Kudu 1.3.0
*  Impala 2.8.0

We suggest using cloudera manager, which is a toolkit integrating the above software. Using this toolkit can simplify the configuration of environment.

*  Cloudera manager 5.11.0

## Usage

Variant-Kudu consists of two stages: Data storage and data analyze. In data storage, it is need to import the VCF data into Kudu and generate distributed bitmap index on HDFS. In data analyze, two form of query is implemented, SQL syntax and parametric syntax.

1.Data Preprocessing and import into Kudu. We implement this function in SaveInKudu class. The parameters are as follows.

* **-spark_master** : the ip and port of Spark master.
* **-kudu_master** : the ip of Kudu cluster.
* **-driver-memory** : the memory allocated to spark driver.
* **-executor-memory** : the memory allocated to spark executor.
* **-vcf_path** : the path of the VCF file.
* **-ped_path** : the path of the ped file.
* **-table_name** : the name of the table stored on Kudu. Multiple genotype tables would be generated according to this table name.

For the following example, meta-information table chr1_META and multiple genotype tables chr1_GBR, chr1_CHS, chr1_FIN would be generated on the Kudu cluster.
```
java -cp Variant-Kudu.jar org.scut.util.SaveInKudu
-spark_master spark://master:7077
-driver-memory 50G
-executor-memory 50G
-kudu_master 192.168.0.10:7051
-vcf_path /path/chr1.vcf
-ped_path /path/chr1_sample.ped
-table_name chr1

2.generate distributed bitmap index on HDFS

For the following example, each genotype tables of chr1 stored on Kudu cluster would be transform into distributed bitmap index stored on HDFS
```
java -cp Variant-Kudu.jar org.scut.util.SaveInBitmap
-spark_master spark://master:7077
-driver-memory 50G
-executor-memory 50G
-kudu_master 192.168.0.10:7051
-hadoop_master 192.168.0.100:50070
-table_name chr1
-bitmap_path /root/chr1/

3.query by SQL syntax.
In this function, we can do variation data analyze using SQL. When a query "select count(*) from chr1-6 where hg00100=’0|0’ and hg00096=’1|0’ and NA12076=’0|0’, we would parse the hg00100=’0|0’ and hg00096=’1|0’ and NA12076=’0|0’" is input, the columns involved is hg00096, hg00100, and NA12076. This is a genotype-information-involved query and would be executed by distributed bitmap index.
If the query is not a genotype-information-involved query, it would be executed by Impala.

Use the following command and then input your query sentence. Note that in the SQL "select xxx from table1 where xxx", the name of the table1 should be corresponding to the table in Kudu.
```
java -cp Variant-Kudu.jar org.scut.util.Query
-spark_master spark://master:7077
-driver-memory 50G
-executor-memory 50G
-kudu_master 192.168.0.10:7051
-impala_master 192.168.0.10:21000
-hadoop_master 192.168.0.100:50070
-bitmap_path /root/chr1/

4.query by parametric syntax
The style of this implementation is similar to GQT. The parametric analyze of genetic variation data would be execute on distributed bitmap index. Genotype queries are composed of a set of predefined values and functions

* **-values**
    HOM_REF, HET, HOM_ALT
* **-functions**
    count(HET): counting the number of samples that are heterozygous
    pct(HET HOM_ALT):The percent of samples that are either heterozygous or homozygous alternate
* **-p: select the designated population**


For the following example, it would counting the number of samples that are heterozygous or homozygous alternate in population GBR, YRI and MSL.
```
java -cp Variant-Kudu.jar org.scut.util.QueryBitmap
-spark_master spark://master:7077
-driver-memory 50G
-executor-memory 50G
-hadoop_master 192.168.0.100:50070
-bitmap_path /root/chr1/
-functions count(HET HOM_ALT)
-p ('GBR', 'YRI', 'MSL')
