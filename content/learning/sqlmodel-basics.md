---
title: SQLModel 基础
date: 2025-10-20 10:00:00 -0400
tags:
  - python
  - sql
---

## 创建表

概览：

```python
from sqlmodel import Field, SQLModel, create_engine

class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    secret_name: str
    age: int | None = None

sqlite_file_name = "database.db"
sqlite_url = f"sqlite:///{sqlite_file_name}"

engine = create_engine(sqlite_url, echo=True)

SQLModel.metadata.create_all(engine)
```

`Hero{:py}` 是我们定义的数据表，其中通过 `table=True{:py}` 指定这是一个要在数据库中创建的数据表，如果不指定的话，就不会在数据库中创建，那它就只是一个单纯的数据结构类。

### 定义列

变量名就是列名，变量类型就是列数据的类型。

像 `age: int | None = None{:py}` 的意思是，`age{:py}` 是可空的，而且默认值就是空（当然也可以是别的默认值），这样在之后插入数据的时候，`age{:py}` 这一列可以不指定，这样数据库会自动插入空值。

### 定义主键

像第一行的 `id{:py}` 列，我们要指定它为主键，就需要用到 `Field(primary_key=True){:py}` ，其实这个主键是肯定不能为空的，那这里为什么也指定 `id: int | None{:py}` 呢？

因为主键不为空这是数据库自己维护的，我们在写代码的时候不需要关心这个问题，也就是我们插入数据的时候其实不需要手动插入 `id{:py}` 列的值，就比如这样：

```python
my_hero = Hero(name="Spider-Boy", secret_name="Pedro Parqueador")
```

但如果声明的时候 `id{:py}` 用了 `int{:py}` 而非 `int | None{:py}` ，那在插入数据的时候就必须手动填一个数字，这是没必要的。不过即便指定了 `int | None{:py}` 我们不用非得填数字了，但还是得显式地写一个 `id=None{:py}` ，这也挺让人困惑的。所以还需要在 `Field{:py}` 中指定 `default=None{:py}` 这样就真的不用手动写什么了。

### 创建引擎

`engine{:py}` 是一个管理与数据库通信的对象。

要创建引擎我们需要传入一个 URL，这个 URL 每个数据库都有自己的格式，像 SQLite 的话，下面这种都行：

```text
sqlite:///database.db
sqlite:///databases/local/application.db
sqlite:///db.sqlite
sqlite://
```

最后一种表示创建一个只在内存中的数据库，关闭即删除。

`engine{:py}` 在整个应用中应该只有一个。另外一种 `Session{:py}` 的对象则不是，这个稍后说。

`echo=True{:py}` 的意思是打印每次执行的 SQL 语句，这个方便 debug，不过实际生产的时候不用设置。

### 创建实际的数据库文件和数据表

前面创建了 `engine{:py}` 并不会实际创建数据库文件，需要执行最后一句才会。

`SQLModel{:py}` 有一个 `metadata{:py}` 属性，我们前面定义的数据表，也就是用 `table=True{:py}` 标记了的类，都会注册到这个 `metadata{:py}` 里面。当我们调用 `create_all{:py}` 的时候，它就会创建数据库文件以及所有注册了的数据表。

也就是说在调用 `create_all{:py}` 之前，得先导入所有的表（如果这些表在其他文件中的话）。

### 最后一点调整

为了方便后续导入，把创建部分单独放在一个函数里，避免每次导入都会执行创建函数。

```python
from sqlmodel import Field, SQLModel, create_engine

class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    secret_name: str
    age: int | None = None

file_name = "data.db"
file_url = f"sqlite:///{file_name}"
engine = create_engine(file_url, echo=True)

def create_db_and_tables():
    SQLModel.metadata.create_all(engine)

if __name__ == "__main__":
    create_db_and_tables()
```

不过不用担心，多次执行创建函数不会覆盖数据库文件和数据表的数据。

## 插入行

首先创建实例

```python
def create_heroes():
    hero_1 = Hero(name="Deadpond", secret_name="Dive Wilson")
    hero_2 = Hero(name="Spider-Boy", secret_name="Pedro Parqueador")
    hero_3 = Hero(name="Rusty-Man", secret_name="Tommy Sharp", age=48)
```

前面说了，`engine{:py}` 是整个应用里独一份的，应该到处复用，但是每一次的与数据库的通信，要使用另一个机制，叫 `Session{:py}` 。这个是用来处理一批操作的东西。

概览：

```python
from sqlmodel import Session, ...

def create_heroes():
    hero_1 = Hero(name="Deadpond", secret_name="Dive Wilson")
    hero_2 = Hero(name="Spider-Boy", secret_name="Pedro Parqueador")
    hero_3 = Hero(name="Rusty-Man", secret_name="Tommy Sharp", age=48)

    session = Session(engine)

    session.add(hero_1)
    session.add(hero_2)
    session.add(hero_3)

    session.commit()
    session.close()
```

创建 `Session{:py}` 对象需要 `engine{:py}` 做参数。`session.add{:py}` 之后就指定了待会要把哪些数据（也就是行）插入到数据表中，指定好了就执行 `session.commit{:py}` 一次性提交所有变更（可能不只是插入）。然后关闭。

如果这一次提交中某个操作出问题了，那么这一批里的所有操作都不会成功。也就是提交操作是原子化的，要么都成功，要么都失败。

为了避免每次还得记得手动关闭 `session{:py}` ，我们可以用 `with{:py}` ：

```python
def create_heroes():
    hero_1 = Hero(name="Deadpond", secret_name="Dive Wilson")
    hero_2 = Hero(name="Spider-Boy", secret_name="Pedro Parqueador")
    hero_3 = Hero(name="Rusty-Man", secret_name="Tommy Sharp", age=48)

    with Session(engine) as session:
        session.add(hero_1)
        session.add(hero_2)
        session.add(hero_3)

        session.commit()
```

这种操作跟 git 挺像的，先 add，再 commit。

只有当 `commit{:py}` 的时候，`session{:py}` 才会与数据库通信，前面 `add{:py}` 的时候并不会。

### Python 对象与数据库数据的同步问题

运行如下代码：

```python
def create_heroes():
    hero_1 = Hero(name="Deadpond", secret_name="Dive Wilson")
    hero_2 = Hero(name="Spider-Boy", secret_name="Pedro Parqueador")
    hero_3 = Hero(name="Rusty-Man", secret_name="Tommy Sharp", age=48)

    with Session(engine) as session:
        print(f"Before add: {hero_1}")

        session.add(hero_1)
        session.add(hero_2)
        session.add(hero_3)

        print(f"Before commit: {hero_1}")
        session.commit()
        print(f"After commit: {hero_1}")
        print(f"After commit, show id: {hero_1.id}")
        print(f"After commit, show name: {hero_1.name}")
```

结果如下：

```console
Before add: name='Deadpond' secret_name='Dive Wilson' id=None age=None
Before commit: name='Deadpond' secret_name='Dive Wilson' id=None age=None

After commit: 

2025-10-20 11:38:03,480 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-10-20 11:38:03,483 INFO sqlalchemy.engine.Engine SELECT hero.id AS hero_id, hero.name AS hero_name, hero.secret_name AS hero_secret_name, hero.age AS hero_age 
FROM hero 
WHERE hero.id = ?
2025-10-20 11:38:03,483 INFO sqlalchemy.engine.Engine [generated in 0.00020s] (1,)
After commit, show id: 1
After commit, show name: Deadpond
```

可以看到，在 `add{:py}` 之前，`id{:py}` 是 `None{:py}` ，`add{:py}` 之后和 `commit{:py}` 之前，`id{:py}` 还是 `None{:py}` 。但是 `commit{:py}` 之后再打印对象，就是空的了，啥也没了。这是因为提交后这一行数据的某些内容可能就变了，比如更新日期，或者 ID 之类的，所以为了不让用户拿到过期的数据，`SQLModel{:py}` 就把 `hero_1{:py}` 对应的数据都清了。当下一次用户想要获取某个数据的时候，比如 `hero_1.id{:py}` 的时候，程序会隐式地抓取一遍数据，并更新到 `hero_1{:py}` 中，这样之后又获取另一个数据 `hero_1.name{:py}` 的时候，就不会再抓取了，同时显示的也是最新的数据。

这种抓取是隐式的，只有在代码要获取 `hero_1{:py}` 的某个值的时候才抓取，但是我们也可以让它显式抓取，只需要：

```python
session.refresh(hero_1)
```

如果说你构建了一个 Web API，用户提交创建了一个 `Hero{:py}` 之后，API 要返回这个创建的对象，那这时候用户没有隐式地触发数据抓取，我们就得显式地刷新数据，然后把包含最新数据的对象返回给用户。

## 查询行

概览：

```python
from sqlmodel import select, ...

def select_heroes():
    with Session(engine) as session:
        statement = select(Hero)
        results = session.exec(statement)
        for hero in results:
            print(hero)
```

在这个新的 `session` 中，通过 `select(Hero)` 指定要查询 `hero` 表中的所有行和所有数据，然后用 `session.exec` 执行实际的查询，返回的是一个可迭代对象。如果要获取列表，可以用 `all` 方法：

```python
heroes = results.all()
print(heroes)
```

### WHERE 语句

像这样

```python
with Session(engine) as session:
    statement = select(Hero).where(Hero.name == "Deadpond")
    heros = session.exec(statement).all()
    print(heros)
```

注意，这里比较的时候用的是 `Hero.name` 也就是类名，而非实际的对象名。

`Hero.name == "Deadpond"` 返回的是一个 `BinaryExpression` 对象，而 `hero.name == "Deadpond"` 返回的是一个简单的 `bool` 类型。

也就是说，`Hero.name` 是一个特殊的类型，而 `hero.name` 就只是一个字符串。在比较中你可以像这样写：

```python
select(Hero).where(Hero.name == hero.name)
```

同样，这里可以用 `!=` 、`>` 等各种比较。

#### 使用 AND

像这样

```python
select(Hero).where(Hero.age >= 35, Hero.age < 40)
```

注意是用逗号隔开，而不是用 `and` 。

#### 使用 OR

需要先导入 `or_` 

```python
from sqlmodel import or_, ...

select(Hero).where(or_(Hero.age < 35, Hero.age > 90))
```

如果在比较的时候有提示说不能把 `None` 和 `int` 比较，这是因为我们声明 `age` 的时候的确用了 `int | None` 。不过数据库知道这个比较只应该在 `age` 不是空的时候才执行，所以要避免这个错误的提示，可以用 `col` 包裹一下列：

```python
from sqlmodel import col, ...

select(Hero).where(col(Hero.age) > 35)
```

### 使用 INDEX

默认数据在插入数据表的时候，是按照插入的先后顺序排序的，如果要查询一个名字，这个名字列是乱序的，程序就得从头到尾挨个查一遍，比较慢，如果说查询名字这个操作频繁，可以给名字列建立一个索引，这个索引会把名字按照一个顺序排列，这样之后在查找的时候，就可以用二分法等更高效的方法查询了，而不用一个一个查，就像查字典一样。

但这样的代价就是，每次在插入一个新数据的时候，除了把数据添加到数据表的最后，它还得查一遍名字列的索引，把新的名字插入到索引的正确位置，以维护索引的有序性。同时建立索引还需要占用额外的空间。

所以，如果查询比插入更频繁，可以建立索引，如果插入比查询更频繁，那就不用建立索引了。

使用 SQL 语句建立索引的方法：
```sql
CREATE INDEX ix_hero_name ON hero (name)
```

在 `SQLModel` 中声明索引：

```python
class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    secret_name: str
    age: int | None = Field(default=None, index=True)
```

其实很简单，就是在 `Field(index=True)` 就可以了。

主键的话不需要声明 INDEX，因为主键默认就有 INDEX。

### 获取第一行

有时候查询的结果我们知道其实只有一行，但是数据库还是会返回一个表，我们还得处理这个迭代，比较繁琐，这时可以用 `first` 方法直接获取第一行结果。

```python
session.exec(select(Hero).where(col(Hero.age) > 35)).first()
```

如果结果一行也没有，`first` 会返回 `None` ，而不会报错。

### 获取绝对的一行

如果我们想强制要求返回的结果必须是一行，不是一行就报错，那么可以用 `one` 方法：

```python
session.exec(select(Hero).where(col(Hero.age) > 35)).one()
```

如果结果有多行，就会报 `sqlalchemy.exc.MultipleResultsFound` 错误，如果结果一行也没有，就会报 `sqlalchemy.exc.NoResultFound` 错误。

### 使用主键索引的便捷方法

因为通过主键查询是一个比较常见的操作，所以除了类似这样的方法：

```python
session.exec(select(Hero).where(Hero.id == 1)).first()
```

还可以直接这样写：

```python
session.get(Hero, 1)
```

简洁很多。当然如果这个 ID 没查到任何东西，就会返回 `None` 而不会报错。

### LIMIT 和 OFFSET

这个比较简单，直接链式调用就可以了：

```python
select(Hero).where(Hero.age > 32).offset(1).limit(2)
```

## 更新行

概览：

```python
def update_heroes():
    with Session(engine) as session:
        hero = session.get(Hero, 2)

        hero.age = 16
        hero.name = "Spider-Youngster"
        session.add(hero)
        session.commit()
        session.refresh(hero)
```

其实就是更改 Python 对象的属性值，然后重新 `add` 和 `commit`，然后为了之后用的是最新的数据，再 `refresh` 一下。在 `add` 之前可以修改多处属性值。

## 删除行

概览：

```python
def delete_heroes():
    with Session(engine) as session:
        hero = session.get(Hero, 2)
        session.delete(hero)
        session.commit()
        # session.refresh(hero)  # 报错
```

使用 `session.delete` 指定接下来要删除这个数据，然后 `commit` 提交一下。之后如果再 `refresh` 的话，就会报错。因为这个数据已经不存在了，但是 `hero` 这个变量本身还是一个合法的 Python 对象，所以里面的属性都还存在，只是它跟数据库就没有任何关系了。
