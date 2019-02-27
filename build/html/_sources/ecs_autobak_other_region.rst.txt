.. contents::
   :depth: 3
..

一、背景
========

1.1 问题：
----------

同事反馈有可以鉴于目前几次大的公有云事故，腾讯云/阿里云两大公有云厂商尚且存在这样令人触目惊心的时刻，更何况其他厂商和我们的日常操作，有人的地方就有误操作，百分之一的风险但如果一旦发生就是100%的问题，虽SLA但或多或少存在影响，客户反馈如果阿里的A地域发生故障，例如地质灾害或不可控因素引发的ecs无法访问，用户数据都在云上无法操作情况下该如何，但考虑到不同地域灾备的高额度IT成本，想采用每天ecs的冷备。

1.2 思路：
----------

在最大程度的降低IT成本，又想在不可控大规模地域性灾难面前做些什么，每天凌晨业务低峰期对ECS制作镜像，同时复制到其他的不同地域，如北京的镜像复制到上海，当北京整个region异常情况下，可利用复制在目标地域的ECS创建出来，在此抛砖引玉，后续可以将ecs在目标地域开出来并关机，归档删除之前的镜像，等等。同样可以将RDS备份也同样备份到异地OSS内，目前阿里已经有EBS非常方便的灾难情况下恢复RDS。利用此思路同意的适用于其他场景下。

二、代码
========

2.1 结构
----------

|image0|
如果多个实例可同时写入配置文件，用,进行分割。

`github地址 <https://github.com/redhatxl/my-python-code/tree/master/imageoper>`__

2.2 核心代码
----------

> 配置文件

::

    # 阿里云ak配置，建议采用子账户只授权ecs镜像操作
    [common]
    # 阿里云acccesskeyid
    accessKeyId = LTAIhfXlcjyln6tW
    # 阿里云accesssecret
    accessSecret = GwfAMvR4K2ELmt76184oqLTVgRfAso
    # log目录名称
    logdir_name = logdir
    # log文件名称
    logfile_name = ecsoperlog.log

    # ecs源地域配置信息段
    #支持在华北 1、华北 2、华北 3、华北 5、华东 1、华东 2 和华南 1 地域之间复制镜像。涉及其他国家和地区地域时，可以 提交工单 申请
    [source]
    # 源地域实例regionid，可以参考：https://help.aliyun.com/document_detail/40654.html?spm=a2c1g.8271268.10000.5.5f98df25B98bhJ
    s_RegionId = cn-shanghai

    # 源实例id,可指定多个用，进行分隔
    s_InstanceId =  i-uf661wb708uvqc9jyhem,i-uf661wb708uvqc9jyhel

    # 源端制作镜像name
    s_ImageName = api-source-image

    # 源镜像描述信息
    s_Description = api-source-image源镜像描述信息

    # 镜像复制目的地域配置信息段
    [destination]
    # 目的地域实例regionid，
    d_DestinationRegionId = cn-qingdao

    # 复制过来的镜像名称
    d_DestinationImageName = api-destination-image

    # 复制过来的镜像描述信息
    d_DestinationDescription = api-destination-image目的镜像描述信息

    image操作(制作镜像->查看镜像制作状态->复制镜像)

::

        # 创建实例生成器
        def _get_Instance(self):
            for Instance in self.s_InstanceId_list.split(','):
                yield Instance
                            
        def _create_image(self):
            """
            创建镜像
            :return:返回镜像id
            """
            s_timer = time.strftime("%Y-%m-%d-%H:%M", time.localtime(time.time()))
            request = CreateImageRequest.CreateImageRequest()
            request.set_accept_format('json')
            request.add_query_param('RegionId', self.s_RegionId)
            request.add_query_param('InstanceId', self.s_InstanceId)
            request.add_query_param('ImageName', self.s_ImageName + s_timer)
            request.add_query_param('Description', self.s_Description + s_timer)
            response = self.ecshelper.do_action_with_exception(request)
            self.logoper.info('创建镜像任务已提交,镜像id:%s' % json.loads(response)["ImageId"])
            print('创建镜像任务已提交,镜像id:%s' % json.loads(response)["ImageId"])
            return json.loads(response)["ImageId"]
                    
     def _describe_image(self,imageid):
            """
            查询image状态
            :param imageid:
            :return:
            """
            request = DescribeImagesRequest.DescribeImagesRequest()
            request.set_accept_format('json')
            request.add_query_param('RegionId', self.s_RegionId)
            request.add_query_param('ImageId', imageid)
            response = self.ecshelper.do_action_with_exception(request)
            # 进度 json.loads(response)['Images']['Image'][0]['Progress']
            self.logoper.info('镜像创建进度:%s' %json.loads(response)['Images']['Image'][0]['Progress'])
            # 镜像状态
            return json.loads(response)['Images']['Image'][0]['Status']


        #镜像复制
        def _copy_image(self,imageid):
            """
            镜像复制
            :param imageid:源镜像id
            :return: 复制成功后的镜像id
            """
            flag = True
            while flag:
                try:
                    if self._describe_image(imageid) == 'Available':
                        flag = False
                    else:
                        time.sleep(300)
                except Exception as e:
                    pass
            print('镜像已经创建完成')
            d_timer = time.strftime("%Y-%m-%d-%H:%M", time.localtime(time.time()))
            request = CopyImageRequest.CopyImageRequest()
            request.set_accept_format('json')
            request.add_query_param('RegionId', self.s_RegionId)
            request.add_query_param('DestinationRegionId', self.d_DestinationRegionId)
            request.add_query_param('DestinationImageName', self.d_DestinationImageName + d_timer)
            request.add_query_param('DestinationDescription', self.d_DestinationDescription + d_timer)
            request.add_query_param('ImageId', imageid)
            response = self.ecshelper.do_action_with_exception(request)
            self.logoper.info('复制镜像任务已提交,镜像id:%s' % json.loads(response)['ImageId'])
            print('复制镜像任务已提交,镜像id:%s' % json.loads(response)['ImageId'])
            return json.loads(response)['ImageId']

三、测试
========

3.1 查看运行结果
----------------

|image1| 

3.2 查看web控制台
----------------

> 源镜像

|image2| > 添加了时间戳，方便查看

|image3| > 目的地域镜像

|image4| 

3.3 查看日志
----------------

|image5|

四、优化
========

-  可以后续增加对指定天数的镜像进行归档删除

.. |image0| image:: http://i2.51cto.com/images/blog/201807/26/5baa365bde3803c1dc069ffe253f9da0.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image1| image:: http://i2.51cto.com/images/blog/201807/26/2c47ad17822e00f55b4d44bad6f34170.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image2| image:: http://i2.51cto.com/images/blog/201807/25/151498a184d482c398c7466bd9fd1bee.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image3| image:: http://i2.51cto.com/images/blog/201807/25/d0794a752612c927d336470dec5097cf.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image4| image:: http://i2.51cto.com/images/blog/201807/25/6a585802e06a4c2d2c27d1034582cac2.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
.. |image5| image:: http://i2.51cto.com/images/blog/201807/26/28d08832853a260d5450fcd92e5d4b08.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
