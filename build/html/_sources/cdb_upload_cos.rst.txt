.. contents::
   :depth: 3
..

一、背景
========

-  需求：目前遇到的客户需求为将腾讯云CDB备份文件自动上传到腾讯云COS内，在此抛砖引玉，还有很多类似的需求均可以采用此类方法解决，线下IDC数据文件备份至云端COS内，或根据文件下载地址url将文件上传至COS内。
-  思路：首先获取到CDB的备份下载url，通过COS的API上传文件，大佬如有更好的方法欢迎一块讨论。

二、技术细节
============

-  COS：COS有API同时有SDK，这就很方便我们来通过Python对COS进行各类操作，\ `COS
   SDK for
   Python <https://cloud.tencent.com/document/product/436/12269>`__
-  CDB：CDB有API但是CDB的查询备份下载没有对应的SDK，此时只能通过API来进行获取，腾讯云API的签名很复杂，要进行：构造参数字典->对dict排序->拼接sign->对sign编码->拼接完成最终url->完成调用，\ `签名方法 <https://cloud.tencent.com/document/product/236/1738>`__\ ，\ `查询备份API <https://cloud.tencent.com/document/api/236/4691>`__

-  requirements:

   ::

       cos-python-sdk-v5==1.5.2
       requests==2.19.1
       tencentcloud-sdk-python==3.0.15
       urllib3==1.23

-  文件目录结构 |image0|

三、代码
========

`github地址 <https://github.com/redhatxl/cdbbak_to_cos>`__ 

3.1 配置文件
-----------------------

::

    # auth:RBS
    # func:将腾讯云cdb备份文件上传至cos制定的bucket内
    # python version:python3+
    # cos version:v5
    # https://console.cloud.tencent.com/cos5/bucket

    # 腾讯云公共信息配置段
    [common]
    # 腾讯云 secretid
    secret_id = AKIDMdjegcmoGxxxxxxxxxxxxxxxxxxxx
    # 腾讯云 secretkey
    secret_key = d5MRL4VoxyvlQvxxxxxxxxxxxxxx

    # 腾讯云cos信息配置段
    [cosinfo]
    # cos所在地域
    cos_region = ap-chengdu

    # 腾讯云bucket名字(cos v5 bucket名称组成:bucket+appid)
    bucket_name = xuel-test-bucket-125396xxxx

    # 腾讯云cdb信息配置段
    [cdbinfo]
    # cdb实例id
    cdb_instanceid = cdb-rqaxxxxx

    # cdb所在地域
    cdb_region = ap-shanghai

    # cdb 日志备份类型，coldbackup（冷备），binlog（二进制日志）和slowlog_day（慢查询日志）
    cdb_bak_type = coldbackup

    # 日志文件信息配置段
    [loginfo]
    #日志文件目录名称
    logdir_name = rds_to_cos
    #日志文件名称
    logfile_name = rdsbak_to_cos.log

3.2 CDB API核心操作代码
-----------------------

::

    #构建字典
    keydict = {
            'Action': self.cdb_action,
            'Timestamp': str(int(time.time())),
            'Nonce': str(int(random.random() * 1000)),
            'Region': self.cdb_region,
            'SecretId': self.secret_id,
            # 'SignatureMethod': SignatureMethod,
            'cdbInstanceId': self.cdb_instanceid,
            'type': self.cdb_bak_type
    }
    #字典排序
    sorted(zip(keydict.keys(), keydict.values()))
    #字符串拼接
    sign_str_init = ''
    for value in sortlist:
            sign_str_init += value[0] + '=' + value[1] + '&'
    sign_str = 'GET' + self.cdb_api_url + sign_str_init[:-1]
    return sign_str, sign_str_init
    #获取签名串并编码
    secretkey = self.secret_key
    signature = bytes(sign_str, encoding='utf-8')
    secretkey = bytes(secretkey, encoding='utf-8')
    my_sign = hmac.new(secretkey, signature, hashlib.sha1).digest()
    my_sign = base64.b64encode(my_sign)
    parse.quote(my_sign)
    #获取最终url
    result_url = 'https://' + self.cdb_api_url + sign_str + '&Signature=' + result_sign

单独运行此模块可以得到以下信息： |image1| 

3.3 COS SDK核心操作代码
-----------------------

::

    #根据文件大小自动选择简单上传或分块上传，分块上传具备断点续传功能
    with open(filename, 'wb') as localfile:
            localfile.write(requests.request('get', url).content)
    # 进行上传
    response = cos_client.upload_file(
            Bucket=self.bucket_name,
            LocalFilePath=filename,
            Key=filename,
            PartSize=partsize,
            MAXThread=maxthread
    )
    # 删除本地文件
    if os.path.exists(filename):
            os.remove(filename)

3.4 日志记录核心代码
--------------------

::

    #创建目录
    def create_dir(self):
            _LOGDIR = os.path.join(os.path.dirname(__file__), self.logdir_name)
            _TIME = time.strftime('%Y-%m-%d', time.gmtime()) + '-'
            _LOGNAME = _TIME + self.logfile_name
            LOGFILENAME = os.path.join(_LOGDIR, _LOGNAME)
            if not os.path.exists(_LOGDIR):
                    os.mkdir(_LOGDIR)
            return LOGFILENAME
    #定义日志文件
    def create_logger(self, logfilename):
            logger = logging.getLogger()
            logger.setLevel(logging.INFO)
            handler = logging.FileHandler(logfilename)
            handler.setLevel(logging.INFO)
            formater = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
            handler.setFormatter(formater)
            logger.addHandler(handler)
            return logger

四、测试结果
============

获取CDB下载链接 |image2| 完成上传查看COS文件 |image3|

五、总结
========

-  优化:可以后期通过配合定时任务完成自动化任务
-  扩展:\ **源端**\ ：不仅仅局限于CDB备份文件，对于随便下载url，均可以上传到COS内。\ **终端**\ ：终端也不仅局限于腾讯云COS，此思路方法也可用于其他云平台如阿里OSS,亚马逊Amazon
   S3,百度云BOS 等。

.. |image0| image:: http://i2.51cto.com/images/blog/201807/16/7b900e0bc1557861d63f8ab54aff507c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image1| image:: http://i2.51cto.com/images/blog/201807/16/d026283eb19a50748f4dbfef1c4f1480.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image2| image:: http://i2.51cto.com/images/blog/201807/16/2f85f2561e012d54e03a5ff15815f29a.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image3| image:: http://i2.51cto.com/images/blog/201807/16/58ffab7bc61f224b535f4ceb1b6491d1.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
