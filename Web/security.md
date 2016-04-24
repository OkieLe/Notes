####跨站脚本

Web应用服务端在处理或存储来自客户端、第三方系统、外部接口等外部输入数据前未进行正确校验（包括未校验和校验不当）、其后在输出到客户端前也未正确进行相应的编码或转义，导致反射型或存储型XSS。

1.来自Web客户端的不可信数据包括但不限于：text、password、textareas、file表单域、hidden fields、selection boxes、check boxes、radio buttons、cookies、HTTP headers、热点链接包含的URL参数；
2.不可信数据做为页面的一部分输出到客户端时，需根据数据使用方式采用不同的编码方式，如用作为HTML标签内容时需要进行HTML编码、用作页面中超链接的URL值时需要进行URL编码等。常用的Web应用中编码方式有HTML编码、URL编码、JavaScript编码、CSS编码等，Web安全框架中已提供相应的编码方式可以直接使用，来将&、<、>、"、'、(、)等特殊字符转换成安全的形式，使其不会被当作语法符号或指令。

```Java
Authcentre中，9处功能的userID字段未经校验，导致反射型XSS攻击：   
初始化（initPswd.do）、   
修改用户密码（modifyPswd.do）   
增、删、改用户（insertAccount.do、deleteAccounts.do、modifyUser.do）   
增、删、改用户角色（insertRole.do、deleteRole.do、modifyRole.do）   
解锁用户（unlockUser.do）   
以初始化用户密码功能为例：initPswd.do的userID字段未经校验导致反射型XSS攻击。   
发送报文，可以看到userID的值未经任何校验就返回  
添加触发事件，就可以造成反射型XSS
```

####SQL注入

WEB应用程序、C/C++应用、Andriod应用(自研代码)中，使用来自客户端、第三方系统、外部接口等外部输入数据构造SQL语句访问数据库时，导致SQL注入问题。

常见情况如下：
1.对来自外部可控的数据未做有效的校验；
2.使用动态拼装SQL语句，而没有使用参数化查询；
应该优先考虑使用参数化查询来防止SQL注入。

```Java
public String appendStrInSqlAndRtnTmpTableName(String[] strConds, IDbOperator dbOper, Connection conn, StringBuilder selectSql)
{
    if (strConds.length <= EamBaseConsts.MAX_SQL_CONDITION_COUNT)
    {
        selectSql.append(EamBaseCommonUtil.joinSqlInCondition("(", strConds, ",", ")"));
        return null;
    }
    StringBuilder createTmpTableSql = new StringBuilder("");
    String tmpTableName = getTmpTableAndCreateSql(createTmpTableSql, TmpTableClass.varcharTable);

    dbOper.createTmpTable(conn, createTmpTableSql.toString());

    String insertTmpTableSql = getBatchInsertTmpTableSql(tmpTableName);

    dbOper.batchInsertStringTmpTable(conn, insertTmpTableSql, strConds);

    selectSql.append("(");
    selectSql.append(getSelectTmpTableSql(tmpTableName));
    SelectSql.append(")");

    return tmpTableName;
}
```

```Andriod
content://mediacenter/external/audio/widgetinfo存在SQL注入漏洞，将应用本不打算提供的数据全部暴露无疑。
if("content://mediacenter/external/audio/widgetinfo".equals(localUri.toString()))
 str = "audio_info";
while(TextUtils.isEmpty(str)){
 DBLogUtils.printLog("MediaCenterProvider","tableName is null in query function!",null);
 localSQLiteDatabase1.close();
 return null;
 if("content://mediacenter/external/audio/external".equals(localUri.toString())){
  paramQueryInnerBean.setDb(localSQLiteDatabase1);
  return new AudioExternalProvider().query(paramQueryInnerBean);
 }
}
```

####缓冲区溢出

C/C++代码使用用户输入、通信协议、用户态输入（对于内核程序而言）等外部输入数据，直接进行缓冲区操作、或作为数组下标、或作为地址偏移等，导致缓冲区写/读越界、或写任意内存地址。

常见情况如下：
1.设备处理接口协议报文时、未检查协议字段长度、就直接使用危险函数拷贝到固定大小的目的缓冲区
2.设备接收用户输入的命令，未检查命令长度，造成命令写越界
3.内核代码在直接或间接处理由ioctl、write、mmap等系统调用传递过来的串口、上层应用等外部输入数据时，直接进行缓冲区操作
4.使用用户输入的数据作为读写内存中固定地址的偏移值，并进行读写
5.双机备份心跳包中传递的主机状态值在数组合法值域外，系统直接使用传入的状态值来索引状态的字符串数组，并打印日志

【C/C++】The function loops over the specified taskno values and copies a single value at a time into a stack buffer, acTask.
The ews_strncpy function is used to copy the taskno values to the array. The third parameter of the ews_strncpy function is the number bytes of the source string to be copied to the character array. If this number is greater than the size of the array then a buffer overflow will occur.
```C++
UINT8 acTask[LMT_ARRAY_LEN_SMALL];
...
ews_memset(acTask, 0, LMT_ARRAY_LEN_SMALL);
xxx_strncpy(acTask,(LmtChar *)pucTaskNo, pucTemp - pucTaskNo); //pucTemp和pucTaskNo都是来自外部输入的特定字符位置，差值可外部控制
```

```C++
A write-what-where vulnerability exists in the mcss_write file operation for the MCSS character device.
//代码片段
int mcss_pou_function(UINT32 ulQueue, UINT32 ulIndex)
{
MC_POU_CMD_QUEUE_STATE_S *pstrQstate;

/* [HCSEC: Chinese comment unreproducable] */
pstrQState = (MC_POU_CMD_QUEUE_STATE_S *)(mcss_pou_queue_state + ulQueue);
pstrQState->index = ulIndex;
MC_PouAddQLen(ulQueue,1);
mcss_ko_statistic->ullPouSentCnt[ulQueue]++;

return MC_OK;
}

if (_peer_stat != _rcv_peer_stat)//_peer_stat来自外部报文输入，未对值域进行充分校验
{
    if ((csUnknown == _peer_stat) || (csInitial == _rcv_peer_stat))
    {
        ....
    }

    _peer_stat = _rcv_peer_stat;
    log_message("peer Status change to %s!", sClusterStatus[_peer_stat]);
}
```

####未正确清理特定内容

系统使用用户输入、通信协议、用户态输入、第三方系统、外部接口等外部输入数据时,未正确进行合法性校验，导致系统宕机、复位、CPU/内存资源过度消耗、服务或进程不可用、权限绕过等。

常见情况如下：
1.外部数据影响代码逻辑、引起系统系统或拒绝服务
2.使用外部协议报文输入作为循环条件，或者作为循环条件的判断依据，引起死循环
3.外部作为内存分配大小参数、导致可申请超大内存
4.外部数据参与数值运算、导致数值溢出、反转，最终导致系统异常
5.Android应用对不可信Intent传入的数据造成不合法操作，导致拒绝服务攻击

【C/C++】pucTo指向报文的选项字段，如果故意构造报文，使其满足以下条件：
1.选项第一个字节非0，非1
2.选项第二个字节为0
则会导致lOptLen为0，且选项指针因为pucFr+=lOptLen，永远不会向后偏移，通过lOptLen = pucFr[IPOPT_OLEN];和lOpt = pucFr[0];却出的数据永远不变，始终满足循环条件，造成死循环

```C++
for (; lCnt > 0; lCnt -= lOptLen, pucFr += lOptLen)
    {
        lOpt = pucFr[0];
        if (lOpt == IPOPT_EOL)
        {
            break;
        }
        if (lOpt == IPOPT_NOP)
        {
            /* Preserve for IP mcast tunnel's LSRR alignment. */
            *pucTo++ = IPOPT_NOP;
            lOptLen = 1;
            continue;
        }
        else
        {
            lOptLen = pucFr[IPOPT_OLEN];
        }
        /* bogus lengths should have been caught by ip_dooptions */
        if (lOptLen > lCnt)
        {
            lOptLen = lCnt;
        }
        if (IPOPT_COPIED(lOpt))
        {
            (VOID)VOS_Mem_Copy( (char *)pucTo, (char *)pucFr,(ULONG)lOptLen );
            pucTo += lOptLen;
        }
    }
```
【C/C++】驱动/dev/jpeg中限制了内存映射中mmap偏移量固定为0xf8c40000，但当mmap时传入Length超长（f8c40000+Length > 0xffffffff）时将会映射出内核空间的地址。通过操作内核空间指令可获取ROOT权限。

####引用空指针

使用指针前未进行非空校验，出现空指针引用，且能被外部触发导致系统宕机/复位、应用拒绝服务等问题。

常见情况如下：
1.C/C++代码对内存分配函数的返回值没有检查，直接访问该指针会导致系统异常
2.Android代码中跨信任边界传入的Intent未判空，出现空指针导致应用异常

【C/C++】对申请内存的函数返回值没有判空检查，直接引用指针指向的内容，导致进程异常退出
```C++
…
line_addr = (unsigned **)malloc(sizeof(__u32) * var.yres_virtual);
…
line_addr[y] = fbuffer + addr;//使用malloc返回值作为指针前，没有判断是否为空
```

```Java
【Android/Java】直接发送action，没有对action做非空处理导致SystemUI进程挂掉。
if(action.equals(TelephonyIntents.ACTION_SERVICE_STATE_CHANGED))
```

####命令注入

程序执行由用户输入、接口数据报文、操作系统环境变量、普通用户可修改的配置文件等外部数据所构建的全部或部分命令语句，导致实际执行结果与预期不一致，产生命令注入。

常见情况如下：
1.C/C++代码中使用未经校验的用户输入数据作为system()/popen()函数的参数，而system()/popen()是通过调用命令解析器（如UNIX/Linux的bash、sh，Windows的CMD.exe）来执行入参指定的程序/命令，导致命令注入
2.Java代码中使用Runtime.exec()函数来调用系统命令解析器来执行用户输入的命令或参数

【C/C++】使用外部传入的文件名拼接命令行，造成命令注入，可使设备重启
```C++
    if(0 == VOS_StrNCmp(pstDkfile->acFileModName, "dpukernel", DM_MODMLE_NAME_LEN))
    {        
        lResult = VOS_nsprintf(acCmdBuf, IPIMU_CMD_BUF_LEN, "insmod %s",
                               pstDkfile->acFileFullName);//使用文件名进行命令行拼接
        if (lResult < 0)
        {
            IPD_LOG_ERR(IPD_DBG_UPDATE, lResult, "Construct cmd failed");
            return VOS_ERR;
        }  
        ucFileUpgGraceFlag = IPIMU_UPG_UNSUPPORTED;
    }
```
