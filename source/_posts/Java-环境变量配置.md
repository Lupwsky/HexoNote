---
title: Java-环境变量配置
date: 2018-10-09 23:25:51
categories:  Java
---

# Java 环境变量

export JAVA_HOME=/home/lupw/Java/jdk1.8.0_101
export JRE_HOME=/home/lupw/Java/jdk1.8.0_101/jre
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin

经常忘记, 记录下来

<!-- more -->