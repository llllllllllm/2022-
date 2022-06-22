# 2022
一个图书馆信息系统中的聊天室
# 1 版本信息

```
Python --3.8.9
Django --4.0.5
Mysql --8.0.28
```

# 2 启动程序

### 1 在 MySQL 中新建一个名为 `LibraryChatRoom` 的数据库，字符集选择 `UTF-8` 。

### 2 安装 MySQL 驱动 `mysqlclient` 。

```
pip install mysqlclient
```

### 3 在 `LibraryChatRoom/settings.py` 中配置数据库的 `USER` 和 `PASSWORD`。

```
DATABASES = {
    'default':
    {
        'ENGINE': 'django.db.backends.mysql',    # 数据库引擎
        'NAME': 'LibraryChatRoom',  # 数据库名称
        'HOST': '127.0.0.1',  # 数据库地址，本机 ip 地址 127.0.0.1
        'PORT': 3306,  # 端口
        'USER': 'root',  # 数据库用户名
        'PASSWORD': 'xxxxx',  # 数据库密码
    }
}
```



### 4 进入项目目录下。

```
cd ./LibraryChatRoom
```

### 5 迁移数据库。

```
python manage.py makemigrations
python manage.py migrate
```

### 6 启动系统。

```
python manage.py runserver
```

### 7 访问 127.0.0.1:8000 。

