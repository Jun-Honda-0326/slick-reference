Slickチートシート
========

* サンプルコードはわかりやすくするため戻り値の型を明示的に書いている部分があります
* SQLの実行例は実際にSlickから発行されるSQLとは異なる場合があります

## CRUDの実行方法

### 取得

よく使うメソッド

メソッド    |説明
------------|-------------
first       |1件取得する。取得できない場合は`NoSuchElementException`をスロー
firstOption |1件取得する。取得できない場合は`None`を返す
list        |結果を`List`で取得する
buildColl   |結果を指定した型で取得する
toMap       |結果をMapで取得する。取得項目（`map`メソッド）でタプル2（key-value形式）になっている必要がある
execute     |クエリを実行する。結果は無視されるので、戻り値は`Unit`
foreach     |クエリを実行し、結果を1件ずつ引数に受け取るコールバック関数を実行する

```scala
val res1: UsersRow = Users.filter(_.id is id.bind).first
val res2: Option[UsersRow] = Users.filter(_.id is id.bind).firstOption
val res3: List[UsersRow] = Users.list
val res4: Seq[UsersRow] = Users.buildColl[Seq]
val res5: Map[Long, UsersRow] = Users.map(t => t.id -> t).toMap
Users foreach println
```

### 登録

メソッド              |説明
----------------------|-------------
insert もしくは +=    |1件登録する。AutoIncのカラムは無視する
insertAll もしくは ++=|複数件登録する。AutoIncのカラムは無視する

```scala
val res1: Int = Users insert UsersRow(0, "なまえ")
val res2: Option[Int] = Users insertAll (
  UsersRow(0, "なまえ1"),
  UsersRow(0, "なまえ2")
)

// select-insertも可能
val res3: Int = Users insert SubUsers.filter(_.name is name.bind)
```

### 更新

メソッド    |説明
------------|-------------
update      |クエリに該当するレコードを更新する

```scala
val res1: Int = Users.filter(_.id is id.bind).update(UsersRow(1, "なまえ変更"))

// 特定のカラムだけ更新
val res2: Int = Users.map(t => t.name -> t.updDate).update("なまえ変更" -> new java.util.Date)
```

### 削除

メソッド    |説明
------------|-------------
delete      |クエリに該当するレコードを削除する

```scala
val res1: Int = Users.filter(_.id is id.bind).delete
```

## 条件: filter

### =

```scala
// select * from USERS x1 where x1.ID = ?
Users.filter(_.id is id.bind)
```

### <>

```scala
// select * from USERS x1 where x1.ID <> ?
Users.filter(_.id isNot id.bind)
```

### is Null

```scala
// select * from USERS x1 where x1.COMPANY_ID is null
Users.filter(_.companyId.isNull)
```

### is not Null

```scala
// select * from USERS x1 where x1.COMPANY_ID is not null
Users.filter(_.companyId.isNotNull)
```

### <, <=, >, >=

```scala
// select * from USERS x1 where x1.ID > ?
Users.filter(_.id > id.bind)
```

### IN

```scala
// select * from USERS x1 where x1.ID in (1, 2)
Users.filter(_.id inSet Seq(1, 2))

// select * from USERS x1 where x1.NAME in (?, ?)
Users.filter(_.name inSetBind Seq("a", "b"))
```

### not IN

```scala
// select * from USERS x1 where not (x1.ID in (1, 2))
Users.filterNot(_.id inSet Seq(1, 2))
```

### between

```scala
// select * from USERS x1 where x1.ID between ? and ?
Users.filter(_.id between (from.bind, to.bind))
```

### like

前方一致と後方一致には、`startsWith`、`endsWith`という便利メソッドがあります。

```scala
// select * from USERS x1 where x1.NAME like 'Biz%' escape '^'
Users.filter(_.name startsWith "Biz")

// select * from USERS x1 where x1.NAME like '%Biz' escape '^'
Users.filter(_.name endsWith "Biz")
```

あいまい検索には便利メソッドがなく、`like`を使います。

```scala
// select * from USERS x1 where x1.NAME like ?
Users.filter(_.name like "%Biz%".bind)
```

### AND

```
// select * from USERS x1 where x1.NAME = ? and x1.COMPANY_ID = ?
Users.filter(t => (t.name is name.bind) && (t.companyId is companyId.bind))
```

### OR

```scala
// select * from USERS x1 where x1.NAME = ? or x1.COMPANY_ID = ?
Users.filter(t => (t.name is name.bind) || (t.companyId is companyId.bind))
```

TODO toUpperCaseとかabsとか、取得時の便利メソッド


## 取得項目: map

### 1カラム

```scala
// select x1.NAME from USERS x1
Users.map(_.name)
```

### 2カラム以上

```scala
// select x1.ID, x1.NAME from USERS x1
Users.map(t => t.id -> t.name)

// select x1.ID, x1.NAME, x1.COMPANY_ID from USERS x1
Users.map(t => (t.id, t.name, t.companyId))
```

上記のクエリをたとえば`list`で結果を取得すると、

* 1件の場合      ⇒L ist[String]のように該当の型
* 2件以上の場合  ⇒ List[(Long, String)]のようにタプル

で取得できます。

### ケースクラスにマッピング

`<>`を使えば、`map`で絞り込んだ項目を別のケースクラスにマッピングして取得することができます。

```scala
case class Test(id: Long, name: String)

val res: List[Test] = Users.map { t =>
  t.id -> t.name <> (Test.tupled, Test.unapply)
}.list
```

### Optionカラムの取得方法

`NULL`可のカラムは、デフォルトでは`Option`で取得することになりますが、直接値を取得したり、`NULL`のときのデフォルト値を設定することで、`String`や`Int`などの型で取得することができます。

```scala
// ↓どちらも発行するSQLは、select x1.PRICE from PRODUCTS x1

// 直接値を取得する。NULLがあるときはSlickExceptionをスロー
Products.map(_.price.get)

// NULLのときのデフォルト値を設定
Products.map(_.price.getOrElse(0))
```

`IFNULL`関数はメソッドが用意されています。

```scala
// select IFNULL(x1.PRICE, 0), count(1) from PRODUCTS x1 group by x1.PRICE
Products
  .groupBy(_.price)
  .map { case (price, t) =>
    price.ifNull(0) -> t.length
  }
```

## ソート: sortBy

`sortBy`でカラムに指定できるもの

メソッド            |説明
--------------------|----------------------
asc                 |昇順。`asc`をつける
desc または reverse |降順。`desc`をつける
nullsFirst          |`NULL`は最初。`nulls first`をつける
nullsLast           |`NULL`は最後。`nulls last`をつける

```scala
// select * from USERS x1 order by x1.ID
Users.sortBy(_.id)

// select * from USERS x1 order by x1.ID, x1.NAME
Users.sortBy(t => t.id -> t.name)

// select * from USERS x1 order by x1.ID desc
Users.sortBy(_.id.desc)

// select * from USERS x1 order by x1.ID desc, x1.NAME
Users.sortBy(t => t.id.desc -> t.name)

// select * from USERS x1 order by x1.COMPANY_ID nulls first
Users.sortBy(_.companyId.nullsFirst)
```

## LIMIT, OFFSET: take, drop

```scala
// select * from USERS x1 limit 30 offset 30
Users.drop(30).take(30)
```

`drop`と`take`は呼び出す順番に注意してください。もし、`take(30).drop(30)`のように記述した場合、「30件取得して30件捨てる」と解釈されるので、`where false`とSQLが発行されます。

## グルーピング: groupBy

### count

```scala
// select x1.COMPANY_ID, count(1) from USERS x1 group by x1.COMPANY_ID
Users
  .groupBy(_.companyId)
  .map { case (companyId, t) =>
    companyId -> t.length
  }
```

### sum, avg, max, min

```scala
// select x1.COMPANY_ID, max(x1.UPD_DATETIME) from USERS x1 group by x1.COMPANY_ID
Users
  .groupBy(_.companyId)
  .map { case (companyId, t) =>
    companyId -> t.map(_.updDatetime).max
  }
```

## 結合: innerJoin, leftJoin

### 内部結合

```scala
// select
//   x1.NAME, x2.NAME
// from
//   USERS x1 inner join COMPANIES x2 on x1.COMPANY_ID = x2.ID
// where
//   x1.ID < ?
Users
  .innerJoin(Companies).on(_.companyId is _.id)
  .filter { case (t1, t2) =>
    t1.id < id.bind
  }
  .map { case (t1, t2) =>
    t1.name -> t2.name
  }
```

### 外部結合

```scala
// select
//   x1.NAME, x2.NAME
// from
//   USERS x1 left outer join COMPANIES x2 on x1.COMPANY_ID = x2.ID
// where
//   x1.ID < ?
Users
  .leftJoin(Companies).on(_.companyId is _.id)
  .filter { case (t1, t2) =>
    t1.id < id.bind
  }
  .map { case (t1, t2) =>
    t1.name -> t2.name.?
  }
```

`NULL`の可能性があるカラムは`?`にすることで`Option`で取得します。

### 複数テーブルを結合

```scala
// select
//   x1.NAME, x2.NAME, x3.FLAG
// from
//   TablesA x1
//     inner join TablesB x2 on x1.ID = x2.ID
//     left outer join TablesC x3 on x2.ID = x3.ID
TablesA
  .innerJoin(TablesB).on { (t1, t2) =>
    t1.id is t2.id
  }
  .leftJoin(TablesC).on { case ((t1, t2), t3) =>
    t2.id is t3.id
  }
  .map { case ((t1, t2), t3) =>
    (t1.name, t2.name, t3.flag.?)
  }
```

## UNION, UNION ALL: union, unionAll

## サブクエリ: in, exists

```scala
// select * from USERS x1 where x1.COMPANY_ID in (select x2.ID from COMPANIES x2)
Users.filter(_.companyId in Companies.map(_.id))

// select * from USERS x1
//   where exists(
//     select *
//     from COMPANIES x2
//     where x1.COMPANY_ID = x2.ID and x2.NAME like 'Biz%' escape '^'
//   )
Users
  .filter { t1 =>
    Companies.filter(t2 =>
      (t1.companyId is t2.id) && (t2.name startsWith "Biz")
    ).exists
  }
```
