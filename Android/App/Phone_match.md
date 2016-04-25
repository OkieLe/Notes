新建/编辑联系人时，如果号码有变化，更新data表时，会进行号码匹配相关数据更新。

具体的更新操作由DataRowHandlerForPhoneNumber来完成：
- 更新后的号码number分别进行两种格式化，得到numberE164，normalizedNumber；
- 如果numberE164不为null，会被插入号码所在行data.data4列中；否则data.data4列为空；
- phone_lookup表中对应normalized_number列插入normalizedNumber，min_match列插入由normalizedNumber产生的最小匹配号码；
- 如果numberE164不为null，phone_lookup表中再插入一行，对应normalized_number列插入numberE164，min_match列插入由numberE164产生的最小匹配号码。

来电/呼出/Mms中的号码经过一些流程(CallerInfoAsyncQuery, etc.)，Uri，例如content://com.android.contacts/phone_lookup/1234567传入联系人数据库进行查询。
- ContactsProvider2.queryLocal()中匹配PHONE_LOOKUP，从Uri中取出号码number；
- number分别进行两种格式化，得到numberE164，normalizedNumber；
- 这两个值被传入ContactsDatabaseHelper.buildPhoneLookupAndContactQuery()创建查询语句；
- 由normalizedNumber产生minMatch，只有min_match列等于minMatch的记录才会进入用于查询的集合；

匹配成功的条件：
1. numberE164不为空，normalized_number列的值等于numberE164；
2. normalized_number列的值长度len小于等于normalizedNumber的长度，且normalizedNumber后len位等于normalized_number列的值相同；
3. numberE164为空，normalized_number列的值长度n大于normalizedNumber的长度，且normalized_number列的值后n位等于normalizedNumber。

以上匹配过程中用到几个格式化，下面简单说明。
- numberE164：PhoneNumberUtils.formatNumberToE164(number, countryCode)，返回号码的E.164表示法，如果传入的号码不符合标准，返回null；
>- 主要调用外部库libphonenumber处理，先调用PhoneNumberUtil.parse(number, countryCode)对号码进行解析；
>- 如果返回结果符合标准PhoneNumberUtil.isValidNumber()，再调用PhoneNumberUtil.format()对解析结果进行格式化，否则返回null。

- normalizedNumber：PhoneNumberUtils.normalizeNumber(number)
>- 不包含大小写字母的号码，返回"+1234567890"，即除了首位的+，任何非数字字符都会被丢弃。
>- 任何位置包含大小写字母的号码，会把字母转换为对应数字后，再次按条件1格式化后返回。

- minMatch：PhoneNumberUtils.toCallerIDMinMatch(number)
>- 先处理号码，保留`0-9、*、#、+，WILD(‘N’)`, `+`只保留第一个，遇到Pause/Wait立即返回当前已处理的子串；
>- 倒序取出处理过的号码的后MIN_MATCH位，MIN_MATCH默认为7。
