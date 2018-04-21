1. 效果展示

2. 数据类型：
    * 新闻数据 hash
    * 新闻id String
    * 分页数据 List
    * 排序 Sorted set

3. 实现功能关键代码
    * 新闻的新增
    ```
    # 新增新闻数据
    def add_news(self, news_obj):
        # 获取新闻id，自增
        int_id = self.r.incr('news_id')
        # 拼接新闻数据hash key(news:2)
        news_id = self._news_id()
        # 存储新闻数据(hash)
        rest = self.r.hmset(news_id, news_obj)
        # 存储新闻的id(list)
        self.r.lpush(self._news_list_name(), int_id)
        # 存储新闻的类别-新闻id(set)
        news_type = self._news_type(news_obj)['news_type']
        self.r.sadd(news_type, int_id)
        return rest
    ```
    * 新增事务支持
    ```
       '''新增新闻数据(事务支持)'''
    def add_news_width_trans(self, news_obj):
        '''新增新闻+事务支持'''
        '''直接通过管道pipe获取，而不是通过连接获取'''
        try:
            # 事务
            pipe = self.r.pipeline()

            # 获取新闻id，自增，这里id只能
            从连接获取，而不能从缓存获取
            int_id = self.r.incr('news_id')
            # 拼接新闻数据hash key(news:2)
            news_id = 'news:%d' % int_id
            # 存储新闻数据(hash)
            rest = pipe.hmset(news_id, news_obj)
            # 存储新闻的id(list)
            pipe.lpush('news', int_id)
            # 存储新闻的类别-新闻id(set)
            news_type = 'news_type:%s' % news_obj['news_type']
            pipe.sadd(news_type, int_id)
            # 事务执行返回结果给rest
            rest = pipe.execute()
        except:
            pass
        return rest
    ```
    * 新闻前台首页
    ```
     def get_all_news(self):
        '''获取所有新闻数据'''
        # 取所有的新闻的id，从列表中去news
        id_list = self.r.lrange(self._news_list_name(), 0, -1)
        news_list = []
        #循环id列表，获取新闻数据hash
        for int_id in id_list:
            # 新闻的key
            news_id = self._news_id(int_id)
            data = self.r.hgetall(news_id)
            data['id'] = int_id
            news_list.append(data)
        return news_list
    ```
    * 分类页
    ```
        def get_news_from_cat(self, news_type):
        '''根据新闻的类别来获取新闻数据'''
        news_list = []
        # 获取新闻类别key
        news_type = self._news_type(news_type)
        # 查询该类别所有新闻id
        id_list = self.r.smembers(news_type)
        # 通过循环来查询所有新闻数据
        for int_id in id_list:
            news_id = self._news_id(int_id)
            data = self.r.hgetall(news_id)
            data['id'] = int_id
            news_list.append(data)
        return news_list
    ```
    * 详情页
    ```
        def get_news_from_id(self, int_id):
        '''根据新闻的id来获取新闻数据'''
        # 获取新闻key
        news_id = self._news_id(int_id)
        # 查询新闻数据hash
        data = self.r.hgetall(news_id)
        data['id'] = int_id
        return data

    ```
    * 后台首页及分页
    ```
        def paginage(self, page=1,per_page=10):
        '''分页数据'''
        if page is None:
            page = 1
        news_list = []
        # 计算开始和结束标记
        start = (page - 1) * per_page
        end = page * per_page - 1
        # 获取当页的id
        id_list = self.r.lrange(self._news_list_name(), start, end)
        # 通过循环来查询数据
        for int_id in id_list:
            news_id = self._news_id(int_id)
            data = self.r.hgetall(news_id)
            data['id'] = int_id
            news_list.append(data)
        return Paginate(news_list, page, per_page)

    ```
    * 新闻修改和删除
