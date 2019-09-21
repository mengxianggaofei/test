### 模型关系

- 一对多的关联

  ```python
  from flask import Flask
  from flask_script import Manager
  from flask_sqlalchemy import SQLAlchemy
  from flask_migrate import Migrate, MigrateCommand
  import os
  
  app = Flask(__name__)
  manager = Manager(app)
  base_dir = os.path.abspath(os.path.dirname(__file__))
  database_uri = "sqlite:///"+ os.path.join(base_dir, "data.sqlite")
  #app.config["SQLALCHEMY_COMMIT_ON_TEARDOWN"] = True
  app.config["SQLALCHEMY_DATABASE_URI"] = database_uri
  db = SQLAlchemy(app)
  migrate = Migrate(app, db)
  manager.add_command("db", MigrateCommand)
  
  class Student(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      name = db.Column(db.String(20), unique=True)
      #dynamic（不加载记录，但提供加载记录的查询）
      articles = db.relationship("Article", backref="stu",lazy="dynamic")
  
  class Article(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      content = db.Column(db.Text)
      sid = db.Column(db.Integer, db.ForeignKey("student.id"))
  
  #开始进行一对多的查询。
  @app.route("/onemany/<id>/")
  def one_to_many(id):
      # #直接查所有的文章
      # articles = Article.query.filter(Article.sid == id).all()
      # return ",".join(a.content for a in articles)
      #模型关联查询
      #通过学生查找文章
      # student = Student.query.get(id)
      # articles = student.articles.all()
      # return ",".join(a.content for a in articles)
      #通过文章查学生
      # article = Article.query.get(id)
      # student = Student.query.get(article.sid)
      # return student.name
      article = Article.query.get(id)
      return article.stu.name
  
  if __name__ == '__main__':
      manager.run()
  ```

  

- 多对多的关联

  ```python
  from flask import Flask
  from flask_script import Manager
  from flask_migrate import Migrate,MigrateCommand
  from flask_sqlalchemy import SQLAlchemy
  import os
  
  app = Flask(__name__)
  manager = Manager(app)
  base_dir = os.path.abspath(os.path.dirname(__file__))
  database_uri = "sqlite:///"+os.path.join(base_dir, "data.sqlite")
  app.config["SQLALCHEMY_DATABASE_URI"] = database_uri
  db = SQLAlchemy(app)
  migrate = Migrate(app, db)
  manager.add_command("db", MigrateCommand)
  
  class Student(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      name = db.Column(db.String(20), unique=True)
      course = db.relationship("Course", secondary = "xuankebiao", backref = db.backref("students",lazy="dynamic"),lazy = "dynamic")
  
  class Course(db.Model):
      id = db.Column(db.Integer, primary_key=True)
      name  = db.Column(db.String(20), unique=True)
  
  
  #声明中间关联表
  xuankebiao = db.Table("xuankebiao",
                        db.Column("student_id", db.Integer, db.ForeignKey("student.id")),
                        db.Column("course_id", db.Integer, db.ForeignKey("course.id")))
  
  # #添加数据路由
  # @app.route("/many_many/")
  # def many_many():
  #     s = Student.query.get(1)
  #     c = Course.query.get(3)
  #     print(s.course)
  #     s.course.append(c)
  #     return "学生"+s.name+"选了"+c.name
  
  #查询路由
  @app.route("/many_many/<id>/")
  def many_many(id):
      # #通过学生查课程
      # s = Student.query.get(id)
      # course  = s.course.all()
      # return ",".join(c.name for c in course)
      #通过课程找学生
      c = Course.query.get(id)
      #?
      students = c.students.all()
      return ",".join(s.name for s in students)
  if __name__ == '__main__':
      manager.run()
  
  ```

### flask缓存

- flask-cache
- pip install flask-cache
- 为什么要有缓存？为了提高网页浏览速度（在咱们前后端分离的代码中，不是咱们管的，）

```python
from flask import Flask
from flask_cache import Cache
from flask_script import Manager

app = Flask(__name__)
manager = Manager(app)
#要缓存到redis数据库
app.config["CACHE_TYPE"] = "redis"
app.config["CACHE_REDIS_HOST"] = "127.0.0.1"
app.config["CACHE_REDIS_PORT"] = 6379
app.config["CACHE_REDIS_DB"] = 1

cache = Cache(app, with_jinja2_ext=False)
@app.route("/")
#@cache.cached(timeout=1000, key_prefix="token")
def index():

    cache.set('token', 'xiaoming', timeout=1000)
    print(cache.get("token"))
    return "ok"

if __name__ == '__main__':
    manager.run()
```

<https://www.cnblogs.com/cwp-bg/p/9687005.html> 

### RESTFUL API接口开发(最重要的，以后你的功作就是做这个)

#### rest

- 简介：REST即表述性状态传递（英文：Representational State Transfer，简称REST）是Roy Fielding博士在2000年他的博士论文中提出来的一种[软件架构](https://baike.baidu.com/item/%E8%BD%AF%E4%BB%B6%E6%9E%B6%E6%9E%84)风格。它是一种针对[网络应用](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C%E5%BA%94%E7%94%A8/2196523)的设计和开发方式，可以降低开发的复杂性，提高系统的可伸缩性。 

- restful api    符合REST风格一种应用程序接口

- 资源：网络资源，在网络上面一切存在的实体都叫网络资源

- 动作：所谓的动作就是增删改查，curd都可以抽象称为网络资源的动作

  | 方法   | 行为                 | 示例                             |
  | ------ | -------------------- | -------------------------------- |
  | GET    | 获取资源的信息       | http://127.0.0.1:5000/source     |
  | GET    | 获取指定的信息       | http://127.0.0.1:5000/source/250 |
  | POST   | 创建新的资源（注册） | http://127.0.0.1:5000/source     |
  | PUT    | 修改指定的资源       | http://127.0.0.1:5000/source/250 |
  | DELETE | 删除指定的资源       | http://127.0.0.1:5000/source/250 |

- 前后端数据进行交互的类型就是，json数据

