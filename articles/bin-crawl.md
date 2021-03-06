##Nutch教程——Nutch流程控制源码详解（bin/crawl中文注释版） 


```bash
#!/bin/bash

# 此中文注释由社区"Nutch开发者" nutcher.org提供，作者是"逼格DATA"，未经允许，禁止转载
# 官方网站 http://nutcher.org
# 教程地址 http://nutcher.org/book/

# 爬取命令 crawl 种子文件夹路径 存放数据文件夹路径 solr的URL 爬取深度（层数）
# The Crawl command script : crawl <seedDir> <crawlDir> <solrURL> <numberOfRounds>
#
# 
# UNLIKE THE NUTCH ALL-IN-ONE-CRAWL COMMAND THIS SCRIPT DOES THE LINK INVERSION AND 
# INDEXING FOR EACH SEGMENT


#$1 $2 ... $n 表示命令后跟的第n个参数
#存放待注入种子的路径
SEEDDIR="$1"
#存放爬取数据（URL状态信息、爬取数据、解析数据）文件夹的路径
CRAWL_PATH="$2"
#如果要将爬取结果提交给solr做索引，第三个参数要设置为solr的url
#如果不需要索引到solr,可将下面这行注释，但需要对其他地方做相应修改
SOLRURL="$3"
#爬取的深度（广度遍历的层数）
#如果不需要提交到solr，将上行注释之后，注意将下行的$4改为$3
LIMIT="$4"


#如果缺少参数，结束整个程序
if [ "$SEEDDIR" = "" ]; then
echo "Missing seedDir : crawl <seedDir> <crawlDir> <solrURL> <numberOfRounds>"
exit -1;
fi

if [ "$CRAWL_PATH" = "" ]; then
echo "Missing crawlDir : crawl <seedDir> <crawlDir> <solrURL> <numberOfRounds>"
exit -1;
fi

#如果不想提交数据到solr，记得将此模块全部注释
if [ "$SOLRURL" = "" ]; then
echo "Missing SOLRURL : crawl <seedDir> <crawlDir> <solrURL> <numberOfRounds>"
exit -1;
fi

if [ "$LIMIT" = "" ]; then
echo "Missing numberOfRounds : crawl <seedDir> <crawlDir> <solrURL> <numberOfRounds>"
exit -1;
fi

#############################################
# MODIFY THE PARAMETERS BELOW TO YOUR NEEDS #
#############################################

# set the number of slaves nodes
#如果在集群运行，一定要将这里改成slave的数量，否则在爬取阶段，只有1台机器会执行爬取
numSlaves=1

# and the total number of available tasks
# sets Hadoop parameter "mapred.reduce.tasks"
#reduce任务的数量会根据上面设定的numSlaves来设定
numTasks=`expr $numSlaves \* 2`

# number of urls to fetch in one iteration
# 250K per task?
sizeFetchlist=`expr $numSlaves \* 50000`

# time limit for feching
timeLimitFetch=180

# num threads for fetching
#每台集群爬取的线程数
numThreads=50

#############################################

#获取执行命令的目录(runtime/local或runtime/deploy)中bin文件夹的绝对路径
bin="`dirname "$0"`"
bin="`cd "$bin"; pwd`"

# determines whether mode based on presence of job file
mode=local
#如果bin文件夹的上级目录（当前文件夹）中包含nutch*.job这样的文件，则认为是分布式运行
#runtime/local文件夹中不包含nutch*.job文件，而runtime/deploy文件夹中包含
if [ -f "${bin}"/../*nutch*.job ]; then
mode=distributed
fi

# note that some of the options listed here could be set in the 
# corresponding hadoop site xml param file 
#一些公用参数
commonOptions="-D mapred.reduce.tasks=$numTasks -D mapred.child.java.opts=-Xmx1000m -D mapred.reduce.tasks.speculative.execution=false -D mapred.map.tasks.speculative.execution=false -D mapred.compress.map.output=true"

# check that hadoop can be found on the path 
# 如果是分布式模式，用which命令检测环境变量中是否配置了hadoop相关参数(HADOOP_HOME,PATH)
# 如果没有在PATH中配置hadoop的bin文件夹，退出程序
# 本地模式不需要配置hadoop相关参数
if [ $mode = "distributed" ]; then
if [ $(which hadoop | wc -l ) -eq 0 ]; then
echo "Can't find Hadoop executable. Add HADOOP_HOME/bin to the path or run in local mode."
exit -1;
fi
fi

# initial injection
# 注入种子列表，$SEEDDIR是bin/crawl后的第一个参数，$CRAWL_PATH是bin/crawl后的第二个参数
# 种子被注入到
"$bin/nutch" inject "$CRAWL_PATH"/crawldb "$SEEDDIR"

#nutch里每个命令执行的都是某个java类的main函数，这些main函数中都会通过System.exit(结束代码)
#$?可以获取到结束代码的值，如果结束代码为0,表示程序运行正常，如果不为0,表示程序异常
#-ne表示不等于
if [ $? -ne 0 ] 
then exit $? 
fi


# main loop : rounds of generate - fetch - parse - update
# 执行循环，每次执行一层（一个深度）的爬取
# 每层爬取都会执行：
#    1.生成本次待爬取任务列表（generate)
#    2.抓取爬取列表中页面（fetch)
#    3.解析爬取页面（parse),抽取所有链接，以及搜索引擎相关数据（网页标题、网页文本、meta信息等)
#    4.将从本次爬取中解析出的链接，更新(update)到$CRAWL_PATH/crawldb中,同时将抓取到的链接的状态（抓取成功、失败、重试次数），更新到$CRAWL_PATH/crawldb中
# LIMIT是bin/crawl命令后参数中给定的
for ((a=1; a <= LIMIT ; a++))
do
#nutch爬取是一个长期的任务，如果想结束nutch的爬取，可以在当前目录（执行命令的目录）下，新建一个.STOP文件，每层爬取一开始都会检测目录是否有.STOP文件（-e表示文件是否存在)
if [ -e ".STOP" ]
then
echo "STOP file found - escaping loop"
break
fi

echo `date` ": Iteration $a of $LIMIT"

echo "Generating a new segment"
#从$CRAWL_PATH/crawldb中，生成待爬取任务列表，并且在segments文件夹下，根据当前时间生成一个文件夹segments/时间,将待爬取任务存放到segments/时间/crawl_generate文件夹中，以SequenceFile<Text,CrawlDatum>的形式存储
#topN表示最多生成多少待爬取任务（URL）
"$bin/nutch" generate $commonOptions "$CRAWL_PATH"/crawldb "$CRAWL_PATH"/segments -topN $sizeFetchlist -numFetchers $numSlaves -noFilter

if [ $? -ne 0 ] 
then exit $? 
fi

# capture the name of the segment
# call hadoop in distributed mode
# or use ls

# 爬取(fetch)任务需要获取上面生成(generate)的任务列表，generate任务会根据当前时间在segments文件夹中生成响应的segment文件夹(segments/时间),时间是用System.currentTimeMillis()生成的long类型的数值，数值越小时间越早，获取刚生成(generate)的segment的方法是：
#    1.用ls命令获取segments文件夹下的文件列表(long类型列表)
#    2.用sort命令将文件名(long类型的时间)排序(从小到大)
#    3.用tail -n 1获取最后一行(最大的时间),也就是最新生成的segment文件夹
if [ $mode = "local" ]; then
SEGMENT=`ls "$CRAWL_PATH"/segments/ | sort -n | tail -n 1`
else
#如果在分布式模式下，需要对hdfs中文件进行查询，用ls无效，需要用hadoop fs -ls
SEGMENT=`hadoop fs -ls "$CRAWL_PATH"/segments/ | grep segments |  sed -e "s/\//\\n/g" | egrep 20[0-9]+ | sort -n | tail -n 1`
fi

echo "Operating on segment : $SEGMENT"

# fetching the segment
# 开始抓取,$numThreads是上面给定的线程数量
# 爬取有2种模式：
#   1.爬取时同时解析
#   2.爬取时不解析
# 默认是爬取时不解析，将解析任务放到后面的parse任务中
# shell中的 -noParsing已经无效
# 如果要修改模式，需要修改conf/nutch-default.xml中的fetcher.parse的值，true表示爬取时同时解析
# 如果将fetcher.parse设为true，设为爬取时同时解析，需要将下面的parse模块全部注释掉
echo "Fetching : $SEGMENT"
"$bin/nutch" fetch $commonOptions -D fetcher.timelimit.mins=$timeLimitFetch "$CRAWL_PATH"/segments/$SEGMENT -noParsing -threads $numThreads

if [ $? -ne 0 ] 
then exit $? 
fi

# parsing the segment
# 解析网页，解析模块包含两个任务
#    1.抽取网页中的链接，使得爬虫可以找到新的链接，在下一层（或者以后）进行爬取
#    2.抽取网页的标题、文本、meta等信息，这些数据会在索引阶段，提交给solr供搜索使用
# 如果将fetcher.parse设为true，设为爬取时同时解析，需要将本parse模块全部注释掉
echo "Parsing : $SEGMENT"
# enable the skipping of records for the parsing so that a dodgy document 
# so that it does not fail the full task
skipRecordsOptions="-D mapred.skip.attempts.to.start.skipping=2 -D mapred.skip.map.max.skip.records=1"
"$bin/nutch" parse $commonOptions $skipRecordsOptions "$CRAWL_PATH"/segments/$SEGMENT

if [ $? -ne 0 ] 
then exit $? 
fi

# 更新(update)模块，nutch的爬取任务维护最核心的模块
# nutch将所有已知的URL信息全部存在$CRAWL_PATH/crawldb中，包含已经爬取的URL，和还未爬取的URL
# updatedb任务完成2个功能：
#    1.将fetch任务中爬取的URL的状态（爬取失败、成功、retry次数等），更新到$CRAWL_PATH/crawldb中
#    2.将parse任务中解析出的链接信息，加到$CRAWL_PATH/crawldb中
# 上面两个功能都要保证$CRAWL_PATH/crawldb中URL的唯一性(一个URL只出现一次），唯一性是通过map recude框架的机制来保证的，reduce获取到的value列表都是对应一个相同的URL的，在这些value中选取一个最合适的留下
# updatedb with this segment
echo "CrawlDB update"
"$bin/nutch" updatedb $commonOptions "$CRAWL_PATH"/crawldb  "$CRAWL_PATH"/segments/$SEGMENT

if [ $? -ne 0 ] 
then exit $? 
fi

# note that the link inversion - indexing routine can be done within the main loop 
# on a per segment basis
# 链接反转，如果不需要做索引(不用solr)可以将这个模块注释
echo "Link inversion"
"$bin/nutch" invertlinks "$CRAWL_PATH"/linkdb "$CRAWL_PATH"/segments/$SEGMENT

if [ $? -ne 0 ] 
then exit $? 
fi

# 根据网页MD5等进行基于内容的去重，如果用nutch做数据采集而不是搜索引擎，可将此模块注释
echo "Dedup on crawldb"
$bin/nutch dedup $CRAWL_PATH/crawldb

if [ $? -ne 0 ] 
then exit $? 
fi

# 将爬取的数据索引并提交到solr，如果不需要索引，可以将下面两个模块注释
echo "Indexing $SEGMENT on SOLR index -> $SOLRURL"
"$bin/nutch" index -D solr.server.url=$SOLRURL "$CRAWL_PATH"/crawldb -linkdb "$CRAWL_PATH"/linkdb "$CRAWL_PATH"/segments/$SEGMENT

if [ $? -ne 0 ] 
then exit $? 
fi

#清除solr中一些索引（因为状态更新的缘故），如果不需要索引，可以将这个模块注释
echo "Cleanup on SOLR index -> $SOLRURL"
"$bin/nutch" clean -D solr.server.url=$SOLRURL "$CRAWL_PATH"/crawldb

if [ $? -ne 0 ] 
then exit $? 
fi

done

exit 0
```

