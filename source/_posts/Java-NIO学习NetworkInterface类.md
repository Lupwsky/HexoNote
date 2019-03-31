---
title: Java-NIO学习NetworkInterface类
date: 2019-03-11 23:01:26
categories: Java
---

NetworkInterface 类是 1.4 加入的类, 此类表示一个由名称和分配给此接口的 IP 地址列表组成的网络接口, 包含了网络接口名称和IP地址等信息

# NetworkInterface 类提供的方法

NetworkInterface 类提供的方法可以参考文档: [NetworkInterface 类](http://doc.codingdict.com/java_api/java/net/NetworkInterface.html)

# 获取 NetworkInterface 类实例

获取 NetworkInterface 类实例只能通过 NetworkInterface 的静态方法获取, 一共有四个, 如下:

```java
// 通过网络设备在当前操作系统中的 IP 获取 NetworkInterface 类实例
NetworkInterface.getByInetAddress(InetAddress inetAddress);

// 通过网络设备在当前操作系统中的名称获取 NetworkInterface 类实例
NetworkInterface.getByName(String name);

// 通过网络设备在当前操作系统中的网络索引值获取 NetworkInterface 类实例
NetworkInterface.getByIndex(int index);

// 获取当前操作系统的所有网络设备的网络接口
Enumeration<NetworkInterface> networkInterface = NetworkInterface.getNetworkInterfaces();
```

<!-- more -->

# 获取网络接口的信息示例

```java
NetworkInterface firstNetworkInterface = null;
Enumeration<NetworkInterface> networkInterfaces = NetworkInterface.getNetworkInterfaces();
while (networkInterfaces.hasMoreElements()) {
    NetworkInterface networkInterface = networkInterfaces.nextElement();
    if (firstNetworkInterface == null) {
        firstNetworkInterface = networkInterface;
    }

    log.info("网络接口名称 = {}", networkInterface.getName());
    log.info("网络接口显示名称 = {}", networkInterface.getDisplayName());
    log.info("网络接口的索引值 = {}", networkInterface.getIndex());
    log.info("网络接口 MAC 地址 = {}", networkInterface.getHardwareAddress());
    log.info("网络接口子的父 NetworkInterface 实例 (如果是物理接口, 为 null) = {}", networkInterface.getParent());
    log.info("网络接口是否是虚拟接口 (子接口) = {}", networkInterface.isVirtual());
    log.info("网络接口是否已经开启并运行 = {}", networkInterface.isVirtual());
    log.info("网络接口是回环网络接口 (通常我们访问的 127.0.0.1, 此 IP 一台机器上只有一个) = {}", networkInterface.isVirtual());
}
```

输出结果如下:

```text
网络接口名称 = lo0
网络接口显示名称 = lo0
网络接口的索引值 = 1
网络接口 MAC 地址 = null
网络接口子的父 NetworkInterface 实例 (如果是物理接口, 为 null) = null
网络接口是否是虚拟接口 (子接口) = false
网络接口是否已经开启并运行 = false
网络接口是回环网络接口 (通常我们访问的 127.0.0.1, 此 IP 一台机器上只有一个) = false

网络接口名称 = en0
网络接口显示名称 = en0
网络接口的索引值 = 5
网络接口 MAC 地址 = [76, 50, 117, -117, -24, 5]
网络接口子的父 NetworkInterface 实例 (如果是物理接口, 为 null) = null
网络接口是否是虚拟接口 (子接口) = false
网络接口是否已经开启并运行 = false
网络接口是回环网络接口 (通常我们访问的 127.0.0.1, 此 IP 一台机器上只有一个) = false
```

使用示例二, 如下:

```java
Enumeration<NetworkInterface> networkInterfaceEnumeration = NetworkInterface.getNetworkInterfaces();
    // 通常情况下, 即使主机没有任何其他网络连接, 回环接口也总是存在的, IPv4 = 127.0.0.1, IPv6 = 0:0:0:0:0:0:0:1
    if (networkInterfaceEnumeration == null) {
        log.info("[获取IP信息] 没有获取到IP信息");
    } else {
        while (networkInterfaceEnumeration.hasMoreElements()) {
            NetworkInterface networkInterface = networkInterfaceEnumeration.nextElement();
            // 网络名称
            String networkName = networkInterface.getName();

            // IP 版本
            String ipVersion = "NONE";
            List<String> hostNameList = new ArrayList<>();
            List<String> addressList  = new ArrayList<>();
            Enumeration<InetAddress> inetAddressEnumeration = networkInterface.getInetAddresses();
            while(inetAddressEnumeration.hasMoreElements()) {
                InetAddress inetAddress = inetAddressEnumeration.nextElement();
                if (inetAddress instanceof Inet4Address) {
                    ipVersion = "v4";
                } else if(inetAddress instanceof Inet6Address) {
                    ipVersion = "v6";
                } else {
                    ipVersion = "NONE";
                }
                hostNameList.add(inetAddress.getHostName());

                // IPv4 是点分形式, IPv6 是冒号分隔的 16 进制形式
                addressList.add(inetAddress.getHostAddress());
            }
            log.info("[获取IP信息] ipVersion = {}, networkName = {}, hostName = {}, address = {}",
                    ipVersion, networkName, hostNameList.toString(), addressList.toString());
        }
    }
} catch (SocketException e) {
    e.printStackTrace();
}
```

输出结果如下:

```text
[获取IP信息] ipVersion = v6, networkName = utun0, hostName = [fe80:0:0:0:f54a:8459:f37a:1fd%utun0], address = [fe80:0:0:0:f54a:8459:f37a:1fd%utun0]
[获取IP信息] ipVersion = v6, networkName = awdl0, hostName = [fe80:0:0:0:a462:a3ff:fe53:401d%awdl0], address = [fe80:0:0:0:a462:a3ff:fe53:401d%awdl0]
[获取IP信息] ipVersion = v4, networkName = en0, hostName = [fe80:0:0:0:18a6:6895:8725:a61e%en0, 192.168.0.107], address = [fe80:0:0:0:18a6:6895:8725:a61e%en0, 192.168.0.107]
[获取IP信息] ipVersion = v4, networkName = lo0, hostName = [fe80:0:0:0:0:0:0:1%lo0, localhost, localhost], address = [fe80:0:0:0:0:0:0:1%lo0, 0:0:0:0:0:0:0:1, 127.0.0.1]
```

# 获取网络接口 IP 信息

InetAddress 类表示了一个 IP 地址信息, 获取 InetAddress 的示例, 可以通过通过 InetAddress 的静态方法来构造, 如下:

```java
// 以下方法的 host 可以是计算机名, IP 地址或者域名, 获取的时候如果没有对应 IP 信息, 则会抛出异常
InetAddress.getAllByName(String host);
InetAddress.getByAddress(byte[] addr);
InetAddress.getByAddress(String host, byte[] addr);
InetAddress.getByName(String host);
```

使用示例:

```java
// 通过 hostName 获取地址信息
String hostName = "www.baidu.com";
try {
    List<String> addressList  = new ArrayList<>();
    InetAddress[] inetAddressList = InetAddress.getAllByName(hostName);
    for (InetAddress inetAddress : inetAddressList) {
        addressList.add(inetAddress.getHostAddress());
    }
    log.info("[获取IP信息] hostName = {}, address = {}", hostName, addressList.toString());
} catch (UnknownHostException e) {
    e.printStackTrace();
}
```

输出结果如下:

```text
[获取IP信息] hostName = www.baidu.com, address = [14.215.177.38, 14.215.177.39]
```

InetAddress 的常用的方法如下:

```java
// 返回 IP 地址, 以字符串的形式, 如 127.0.0.1
InetAddress.getHostAddress();

// 返回 IP 地址, 返回的是 byte 数组
InetAddress.getAddress();

// 获取 IP 地址所对应的的主机别名
InetAddress.getHostName();

// 获取 IP 地址所对应的主机名
InetAddress.getCanonicalHostName();
```

InetAddress 还有有两个子类, Inet4Address, Inet6Address 分别表示 IPv4 和 IPv6 地址信息, 开发的时候需要获取网络接口的 IP, 可以通过 NetworkInterface.getInetAddresses() 获取绑定在这个网络接口上的 IP 信息, 使用示例如下:

```java
Enumeration<InetAddress> addressEnumeration = networkInterface.getInetAddresses();
while (addressEnumeration.hasMoreElements()) {
    InetAddress inetAddress = addressEnumeration.nextElement();
    log.info(inetAddress.getHostAddress());
    log.info(inetAddress.getHostName());
    log.info(inetAddress.getCanonicalHostName());
}
```

# 获取本机 IP

获取本机 IP 可以使用如下代码实现:

```java
InetAddress.getLocalHost().getHostAddress()
```

上面的方法在大多数机器上都可以取到本机 IP, 但是在多网卡时获取就会出现问题, 曾经在项目中使用这种方法时就出现过问题, 可以使用下面的方法获取, 下面的方法在多网卡时可以获取本机 IP, 但是如果对虚拟网卡可能会获取到虚拟网卡, 获取到不是正确的 IP, 代码如下:

```java
private static InetAddress getLocalHostLANAddress() throws UnknownHostException {
    try {
        InetAddress candidateAddress = null;
        // 遍历所有的网络接口
        for (Enumeration ifaces = NetworkInterface.getNetworkInterfaces(); ifaces.hasMoreElements();) {
            NetworkInterface iface = (NetworkInterface) ifaces.nextElement();
            // 在所有的接口下再遍历 IP
            for (Enumeration inetAddrs = iface.getInetAddresses(); inetAddrs.hasMoreElements();) {
                InetAddress inetAddr = (InetAddress) inetAddrs.nextElement();
                // 排除 loopback 类型地址
                if (!inetAddr.isLoopbackAddress()) {
                    if (inetAddr.isSiteLocalAddress()) {
                        // 如果是 site-local 地址，就是它了
                        return inetAddr;
                    } else if (candidateAddress == null) {
                        // site-local 类型的地址未被发现，先记录候选地址
                        candidateAddress = inetAddr;
                    }
                }
            }
        }
        if (candidateAddress != null) {
            return candidateAddress;
        }

        // 如果没有发现 non-loopback 地址.只能用最次选的方案
        InetAddress jdkSuppliedAddress = InetAddress.getLocalHost();
        if (jdkSuppliedAddress == null) {
            throw new UnknownHostException("The JDK InetAddress.getLocalHost() method unexpectedly returned null.");
        }
        return jdkSuppliedAddress;
    } catch (Exception e) {
        UnknownHostException unknownHostException = new UnknownHostException("Failed to determine LAN address: " + e);
        unknownHostException.initCause(e);
        throw unknownHostException;
    }
}
```

上面的方法源自文章: [详谈再论JAVA获取本机IP地址](https://www.cnblogs.com/starcrm/p/7071227.html)